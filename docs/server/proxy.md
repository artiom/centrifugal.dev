---
id: proxy
title: Proxy to backend
---

It's possible to proxy some client connection events from Centrifugo to the application backend and react to them in a custom way. For example, it's possible to authenticate connection via request from Centrifugo to application backend, refresh client sessions and answer to RPC calls sent by a client over bidirectional connection.

## Proxy overview

The list of events that can be proxied:

* Connect – called when a client connects to Centrifugo, so it's possible to authenticate user, return custom data to a client, attach meta information to the connection, and so on. Works for bidirectional and unidirectional transports.
* Refresh - called when a client session is going to expire, so it's possible to prolong it or just let it expire. Works for bidirectional and unidirectional transports.
* Subscribe - called when clients try to subscribe on a channel, so it's possible to check permissions and return custom initial subscription data. Works for bidirectional transports only.
* Publish - called when a client tries to publish into a channel, so it's possible to check permissions and optionally modify publication data. Works for bidirectional transports only.
* RPC - called when a client sends RPC, you can do whatever logic you need based on a client-provided RPC method and params. Works for bidirectional transports only.

At the moment Centrifugo can proxy these events over two protocols:

* HTTP
* GRPC

## HTTP proxy

HTTP proxy in Centrifugo converts client connection events into HTTP call to the application backend.

### HTTP request structure

All proxy calls are **HTTP POST** requests that will be sent from Centrifugo to configured endpoints with a configured timeout. These requests will have some headers copied from the original client request (see details below) and include JSON body which varies depending on call type (for example data sent by a client in RPC call etc, see more details about JSON bodies below).

### Proxy HTTP headers

The good thing about Centrifugo HTTP proxy is that it transparently proxies original HTTP request headers in a request to app backend. But it's required to provide an explicit list of HTTP headers you want to be proxied, for example:

```json title="config.json"
{
    ...
    "proxy_http_headers": [
        "Origin",
        "User-Agent",
        "Cookie",
        "Authorization",
        "X-Real-Ip",
        "X-Forwarded-For",
        "X-Request-Id"
    ]
}
```

Alternatively, you can set a list of headers via an environment variable (space separated):

```
export CENTRIFUGO_PROXY_HTTP_HEADERS="Cookie User-Agent X-B3-TraceId X-B3-SpanId" ./centrifugo
```

:::note

Centrifugo forces the` Content-Type` header to be `application/json` in all HTTP proxy requests since it sends the body in JSON format to the application backend.

:::

### Connect proxy

With the following options in the configuration file:

```json
{
  ...
  "proxy_connect_endpoint": "http://localhost:3000/centrifugo/connect",
  "proxy_connect_timeout":  "1s"
}
```

– connection requests **without JWT set** will be proxied to `proxy_connect_endpoint` URL endpoint. On your backend side, you can authenticate the incoming connection and return client credentials to Centrifugo in response to the proxied request.

:::danger

Make sure you properly configured [allowed_origins](configuration.md#allowed_origins) Centrifugo option or check request origin on your backend side upon receiving connect request from Centrifugo. Otherwise, your site can be vulnerable to CSRF attacks if you are using WebSocket transport for client connections.

:::

Yes, this means you don't need to generate JWT and pass it to a client-side and can rely on a cookie while authenticating the user. **Centrifugo should work on the same domain in this case so your site cookie could be passed to Centrifugo by browsers**. This also means that **every** new connection from a user will result in an HTTP POST request to your application backend. While with JWT token you usually generate it once on application page reload, if client reconnects due to Centrifugo restart or internet connection loss it uses the same JWT it had before thus usually no additional requests are generated during reconnect process (until JWT expired).

![](/img/diagram_connect_proxy.png)

Payload example that will be sent to app backend when client without token wants to establish a connection with Centrifugo and `proxy_connect_endpoint` is set to non-empty URL string:

```json
{
  "client":"9336a229-2400-4ebc-8c50-0a643d22e8a0",
  "transport":"websocket",
  "protocol": "json",
  "encoding":"json"
}
```

Response expected:

```json
{"result": {"user": "56"}}
```

This response allows connecting and tells Centrifugo the ID of a user. See below the full list of supported fields in the result.

#### Connect request fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| client       | string     | no | unique client ID generated by Centrifugo for each incoming connection  |
| transport    | string     | no | transport name (ex. `websocket`, `sockjs`, `uni_sse` etc)        |
| protocol     | string     | no | protocol type used by the client (`json` or `protobuf` at moment)            |
| encoding     | string     | no | protocol encoding type used (`json` or `binary` at moment)            |
| name         | string     | yes | optional name of the client (this field will only be set if provided by a client on connect)            |
| version      | string     | yes | optional version of the client (this field will only be set if provided by a client on connect)            |
| data         | JSON object     | yes | optional data from client (this field will only be set if provided by a client on connect)            |
| b64data      | JSON object     | yes | optional data from the client in base64 format (if the binary proxy mode is used)            |
| channels      | Array of strings     | yes | list of server-side channels client want to subscribe to, the application server must check permissions and add allowed channels to result               |

#### Connect result fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| user       | string     | no |  user ID (calculated on app backend based on request cookie header for example). Return it as an empty string for accepting unauthenticated requests |
| expire_at    | integer     | yes | a timestamp when connection must be considered expired. If not set or set to `0` connection won't expire at all        |
| info     | JSON object     | yes | a connection info JSON            |
| b64info     | string     | yes | binary connection info encoded in base64 format, will be decoded to raw bytes on Centrifugo before using in messages            |
| data         | JSON object     | yes | a custom data to send to the client in connect command response.           |
| b64data      | string     | yes | a custom data to send to the client in the connect command response for binary connections, will be decoded to raw bytes on Centrifugo side before sending to client            |
| channels      | array of strings     | yes | allows providing a list of server-side channels to subscribe connection to. See more details about [server-side subscriptions](server_subs.md)       |
| subs         | map of SubscribeOptions     | yes | map of channels with options to subscribe connection to. See more details about [server-side subscriptions](server_subs.md)           |
| meta         | JSON object     | yes | a custom data to attach to connection (this **won't be exposed to client-side**)  |

#### Options

`proxy_connect_timeout` (float, in seconds) config option controls timeout of HTTP POST request sent to app backend.

#### Example

Here is the simplest example of the connect handler in Tornado Python framework (note that in a real system you need to authenticate the user on your backend side, here we just return `"56"` as user ID):

```python
class CentrifugoConnectHandler(tornado.web.RequestHandler):

    def check_xsrf_cookie(self):
        pass

    def post(self):
        self.set_header('Content-Type', 'application/json; charset="utf-8"')
        data = json.dumps({
            'result': {
                'user': '56'
            }
        })
        self.write(data)


def main():
    options.parse_command_line()
    app = tornado.web.Application([
      (r'/centrifugo/connect', CentrifugoConnectHandler),
    ])
    app.listen(3000)
    tornado.ioloop.IOLoop.instance().start()


if __name__ == '__main__':
    main()
```

This example should help you to implement a similar HTTP handler in any language/framework you are using on the backend side.

### Refresh proxy

With the following options in the configuration file:

```json
{
  ...
  "proxy_refresh_endpoint": "http://localhost:3000/centrifugo/refresh",
  "proxy_refresh_timeout":  "1s"
}
```

– Centrifugo will call `proxy_refresh_endpoint` when it's time to refresh the connection. Centrifugo itself will ask your backend about connection validity instead of refresh workflow on the client-side.

The payload sent to app backend in refresh request (when the connection is going to expire):

```json
{
  "client":"9336a229-2400-4ebc-8c50-0a643d22e8a0",
  "transport":"websocket",
  "protocol": "json",
  "encoding":"json",
  "user":"56"
}
```

Response expected:

```json
{"result": {"expire_at": 1565436268}}
```

#### Refresh request fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| client       | string     | no | unique client ID generated by Centrifugo for each incoming connection  |
| transport    | string     | no | transport name (ex. `websocket`, `sockjs`, `uni_sse` etc.)        |
| protocol     | string     | no | protocol type used by client (`json` or `protobuf` at moment)            |
| encoding     | string     | no | protocol encoding type used (`json` or `binary` at moment)            |
| user         | string     | no | a connection user ID obtained during authentication process         |
| meta         | JSON object | yes | a connection attached meta (off by default, enable with `"proxy_include_connection_meta": true`)         |


#### Refresh result fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| expired       | bool     | yes |  a flag to mark the connection as expired - the client will be disconnected  |
| expire_at    | integer     | yes | a timestamp in the future when connection must be considered expired       |
| info     | JSON object     | yes | a connection info JSON            |
| b64info     | string     | yes | binary connection info encoded in base64 format, will be decoded to raw bytes on Centrifugo before using in messages            |

#### Options

`proxy_refresh_timeout` (float, in seconds) config option controls timeout of HTTP POST request sent to app backend.

### RPC proxy

With the following option in the configuration file:

```json
{
  ...
  "proxy_rpc_endpoint": "http://localhost:3000/centrifugo/connect",
  "proxy_rpc_timeout":  "1s"
}
```

RPC calls over client connection will be proxied to `proxy_rpc_endpoint`. This allows a developer to utilize WebSocket (or SockJS) connection in a bidirectional way.

Payload example sent to app backend in RPC request:

```json
{
  "client":"9336a229-2400-4ebc-8c50-0a643d22e8a0",
  "transport":"websocket",
  "protocol": "json",
  "encoding":"json",
  "user":"56",
  "method": "getCurrentPrice",
  "data":{"params": {"object_id": 12}}
}
```

Response expected:

```json
{"result": {"data": {"answer": "2019"}}}
```

#### RPC request fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| client       | string     | no | unique client ID generated by Centrifugo for each incoming connection  |
| transport    | string     | no | transport name (ex. `websocket` or `sockjs`)        |
| protocol     | string     | no | protocol type used by the client (`json` or `protobuf` at moment)            |
| encoding     | string     | no | protocol encoding type used (`json` or `binary` at moment)            |
| user         | string     | no |  a connection user ID obtained during authentication process         |
| method         | string     | yes |  an RPC method string, if the client does not use named RPC call then method will be omitted      |
| data         | JSON object     | yes |  RPC custom data sent by client       |
| b64data         | string     | yes |  will be set instead of `data` field for binary proxy mode       |
| meta         | JSON object | yes | a connection attached meta (off by default, enable with `"proxy_include_connection_meta": true`)         |

#### RPC result fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| data     | JSON object     | yes | RPC response - any valid JSON is supported            |
| b64data     | string     | yes | can be set instead of `data` for binary response encoded in base64 format   |

#### Options

`proxy_rpc_timeout` (float, in seconds) config option controls timeout of HTTP POST request sent to app backend.

See below on how to return a custom error.

### Subscribe proxy

With the following option in the configuration file:

```json
{
  ...
  "proxy_subscribe_endpoint": "http://localhost:3000/centrifugo/subscribe",
  "proxy_subscribe_timeout":  "1s"
}
```

– subscribe requests sent over client connection will be proxied to `proxy_subscribe_endpoint`. This allows you to check the access of the client to a channel.

:::tip

**Subscribe proxy does not proxy [private](channels.md#private-channel-prefix) and [user-limited](channels.md#user-channel-boundary) channels at the moment**. That's because those are already providing a level of security (user-limited channels check current user ID, private channels require subscription token). In some cases you may use subscribe proxy as a replacement for private channels actually: if you prefer to check permissions using the proxy to backend mechanism – just stop using `$` prefixes in channels, properly configure subscribe proxy and validate subscriptions upon proxy from Centrifugo to your backend (issued each time user tries to subscribe on a channel for which subscribe proxy enabled).

:::

Unlike proxy types described above subscribe proxy must be enabled per channel namespace. This means that every namespace (including global/default one) has a boolean option `proxy_subscribe` that enables subscribe proxy for channels in a namespace.

So to enable subscribe proxy for channels without namespace define `proxy_subscribe` on a top configuration level:

```json
{
  ...
  "proxy_subscribe_endpoint": "http://localhost:3000/centrifugo/subscribe",
  "proxy_subscribe_timeout":  "1s",
  "proxy_subscribe": true
}
```

Or for channels in namespace `sun`:

```json
{
  ...
  "proxy_subscribe_endpoint": "http://localhost:3000/centrifugo/subscribe",
  "proxy_subscribe_timeout":  "1s",
  "namespaces": [{
    "name": "sun",
    "proxy_subscribe": true
  }]
}
```

Payload example sent to app backend in subscribe request:

```json
{
  "client":"9336a229-2400-4ebc-8c50-0a643d22e8a0",
  "transport":"websocket",
  "protocol": "json",
  "encoding":"json",
  "user":"56",
  "channel": "chat:index"
}
```

Response expected:

```json
{"result": {}}
```

#### Subscribe request fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| client       | string     | no | unique client ID generated by Centrifugo for each incoming connection  |
| transport    | string     | no | transport name (ex. `websocket` or `sockjs`)        |
| protocol     | string     | no | protocol type used by the client (`json` or `protobuf` at moment)            |
| encoding     | string     | no | protocol encoding type used (`json` or `binary` at moment)            |
| user         | string     | no |  a connection user ID obtained during authentication process         |
| channel         | string     | no |  a string channel client wants to subscribe to        |
| meta         | JSON object | yes | a connection attached meta (off by default, enable with `"proxy_include_connection_meta": true`)         |

#### Subscribe result fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| info     | JSON object     | yes | a channel info JSON         |
| b64info     | string     | yes | a binary connection channel info encoded in base64 format, will be decoded to raw bytes on Centrifugo before using   |
| data         | JSON object     | yes | a custom data to send to the client in subscribe command reply.           |
| b64data      | string     | yes | a custom data to send to the client in subscribe command reply, will be decoded to raw bytes on Centrifugo side before sending to client |
| override       | Override object       | yes |  Allows dynamically override some channel options defined in Centrifugo configuration on a per-connection basis (see below available fields)  |

#### Override object

| Field | Type | Optional | Description  |
| -------------- | -------------- | ------------ | ---- |
| presence       | BoolValue       | yes | Override presence   |
| join_leave       | BoolValue       | yes | Override join_leave   |
| position       | BoolValue       | yes | Override position   |
| recover       | BoolValue       | yes |  Override recover   |

BoolValue is an object like this:

```json
{
  "value": true/false
}
```

See below on how to return an error in case you don't want to allow subscribing.

#### Options

`proxy_subscribe_timeout` (float, in seconds) config option controls timeout of HTTP POST request sent to app backend.

### Publish proxy

With the following option in the configuration file:

```json
{
  ...
  "proxy_publish_endpoint": "http://localhost:3000/centrifugo/publish",
  "proxy_publish_timeout":  "1s"
}
```

– publish calls sent by a client will be proxied to `proxy_publish_endpoint`.

This request happens BEFORE a message is published to a channel, so your backend can validate whether a client can publish data to a channel. An important thing here is that publication to the channel can fail after your backend successfully validated publish request (for example publish to Redis by Centrifugo returned an error). In this case, your backend won't know about the error that happened but this error will propagate to the client-side. 

![](/img/diagram_publish_proxy.png)

Like the subscribe proxy, publish proxy must be enabled per channel namespace. This means that every namespace (including the global/default one) has a boolean option `proxy_publish` that enables publish proxy for channels in the namespace. All other namespace options will be taken into account before making a proxy request, so you also need to turn on the `publish` option too.

So to enable publish proxy for channels without namespace define `proxy_publish` and `publish` on a top configuration level:

```json
{
  ...
  "proxy_publish_endpoint": "http://localhost:3000/centrifugo/publish",
  "proxy_publish_timeout":  "1s",
  "publish": true,
  "proxy_publish": true
}
```

Or for channels in namespace `sun`:

```json
{
  ...
  "proxy_publish_endpoint": "http://localhost:3000/centrifugo/publish",
  "proxy_publish_timeout":  "1s",
  "namespaces": [{
    "name": "sun",
    "publish": true,
    "proxy_publish": true
  }]
}
```

Keep in mind that this will only work if the `publish` channel option is on for a channel namespace (or for a global top-level namespace).

Payload example sent to app backend in a publish request:

```json
{
  "client":"9336a229-2400-4ebc-8c50-0a643d22e8a0",
  "transport":"websocket",
  "protocol": "json",
  "encoding":"json",
  "user":"56",
  "channel": "chat:index",
  "data":{"input":"hello"}
}
```

Response example:

```json
{"result": {}}
```

#### Publish request fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| client       | string     | no | unique client ID generated by Centrifugo for each incoming connection  |
| transport    | string     | no | transport name (ex. `websocket`, `sockjs`)        |
| protocol     | string     | no | protocol type used by the client (`json` or `protobuf` at moment)            |
| encoding     | string     | no | protocol encoding type used (`json` or `binary` at moment)            |
| user         | string     | no |  a connection user ID obtained during authentication process         |
| channel         | string     | no |  a string channel client wants to publish to        |
| data     | JSON object     | yes | data sent by client        |
| b64data     | string     | yes |  will be set instead of `data` field for binary proxy mode   |
| meta         | JSON object | yes | a connection attached meta (off by default, enable with `"proxy_include_connection_meta": true`)         |

#### Publish result fields

| Field | Type | Optional | Description |
| ------------ | -------------- | ------------ | ---- |
| data     | JSON object     | yes | an optional JSON data to send into a channel **instead of** original data sent by a client         |
| b64data     | string     | yes | a binary data encoded in base64 format, the meaning is the same as for data above, will be decoded to raw bytes on Centrifugo side before publishing  |
| skip_history     | bool     | yes | when set to `true` Centrifugo won't save publication to the channel history |

See below on how to return an error in case you don't want to allow publishing.

#### Options

`proxy_publish_timeout` (float, in seconds) config option controls timeout of HTTP POST request sent to app backend.

### Return custom error

Application backend can return JSON object that contains an error to return it to the client:

```json
{
  "error": {
    "code": 1000,
    "message": "custom error"
  }
}
```

Application **should use error codes >= 1000**, error codes in the range 0-999 are reserved by Centrifugo internal protocol. Error code field is `uint32` internally.

:::note

Returning custom error does not apply to response on refresh request as there is no sense in returning an error (will not reach client anyway). 

:::

### Return custom disconnect

Application backend can return JSON object that contains a custom disconnect object to disconnect client in a custom way:

```json
{
  "disconnect": {
    "code": 4000,
    "reconnect": false,
    "reason": "custom disconnect"
  }
}
```

Application **must use numbers in the range 4000-4999 for custom disconnect codes**. Code is `uint32` internally. Numbers below 4000 are reserved by Centrifugo internal protocol. Keep in mind that **due to WebSocket protocol limitations and Centrifugo internal protocol needs you need to keep disconnect reason string no longer than 32 symbols**.

:::note

Returning custom disconnect does not apply to response on refresh request as there is no way to control disconnect at moment - the client will always be disconnected with `expired` disconnect reason.

:::


## GRPC proxy

Centrifugo can also proxy connection events to your backend over GRPC instead of HTTP. In this case, Centrifugo acts as a GRPC client and your backend acts as a GRPC server.

GRPC service definitions can be found in the Centrifugo repository: [proxy.proto](https://github.com/centrifugal/centrifugo/blob/master/internal/proxyproto/proxy.proto).

:::tip

GRPC proxy inherits all the fields for HTTP proxy – so you can refer to field descriptions for HTTP above. Both proxy types in Centrifugo share the same Protobuf schema definitions.

:::

Every proxy call in this case is a unary GRPC call. Centrifugo puts client headers into GRPC metadata (since GRPC doesn't have headers concept).

All you need to do to enable proxying over GRPC instead of HTTP is to use `grpc` schema in endpoint, for example for the connect proxy:

```json title="config.json"
{
  ...
  "proxy_connect_endpoint": "grpc://localhost:12000",
  "proxy_connect_timeout":  "1s"
}
```

Refresh proxy:

```json title="config.json"
{
  ...
  "proxy_refresh_endpoint": "grpc://localhost:12000",
  "proxy_refresh_timeout":  "1s"
}
```


Or for RPC proxy:

```json title="config.json"
{
  ...
  "proxy_rpc_endpoint": "grpc://localhost:12000",
  "proxy_rpc_timeout":  "1s"
}
```

For publish proxy in namespace `chat`:

```json title="config.json"
{
  ...
  "proxy_publish_endpoint": "grpc://localhost:12000",
  "proxy_publish_timeout":  "1s"
  "namespaces": [
    {
      "name": "chat",
      "publish": true,
      "proxy_publish": true
    }
  ]
}
```

Use subscribe proxy for all channels without namespaces:

```json title="config.json"
{
  ...
  "proxy_subscribe_endpoint": "grpc://localhost:12000",
  "proxy_subscribe_timeout":  "1s",
  "proxy_subscribe": true
}
```

So the same as for HTTP, just the different endpoint scheme.

### Options

#### proxy_grpc_cert_file

String, default: `""`.

Path to cert file for secure TLS connection. If not set then an insecure connection with the backend endpoint is used.

#### proxy_grpc_credentials_key

String, default `""` (i.e. not used).

Add custom key to per-RPC credentials.

#### proxy_grpc_credentials_value

String, default `""` (i.e. not used).

A custom value for `proxy_grpc_credentials_key`.

### GRPC proxy example

We have [an example of backend server](https://github.com/centrifugal/examples/tree/master/proxy/grpc) (written in Go language) which can react to events from Centrifugo over GRPC. For other programming languages the approach is similar, i.e.:

1. Copy proxy Protobuf definitions
1. Generate GRPC code
1. Run backend service with you custom business logic
1. Point Centrifugo to it.

## Header proxy rules

Centrifugo not only supports HTTP-based client transports but also GRPC-based (for example GRPC unidirectional stream). Here is a table with rules used to proxy headers/metadata in various scenarios:

| Client protocol type  | Proxy type  | Client headers | Client metadata |
| ------------- | ------------- | -------------- | -------------- |
| HTTP  | HTTP  |  In proxy request headers |    N/A  |
| GRPC  | GRPC  |  N/A  |  In proxy request metadata  |
| HTTP  | GRPC  |  In proxy request metadata  |    N/A  |
| GRPC  | HTTP  |  N/A  |  In proxy request headers  |
