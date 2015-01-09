# WebSocket Emulation (WSE) Protocol

## Abstract
This document specifies the behavior of WebSocket protocol emulated via HTTP.  It can be used to enable full WebSocket 
capabilities for HTTP clients while minimizing the syntactic differences to maintain consistent performance and bandwidth 
utilization profiles.

## Status of This Memo

This document specifies a protocol for the Internet community, and requests discussion and suggestions for improvement.
This memo is written in the style of an IETF RFC, but is _not_ an official IETF RFC.

## Copyright Notice

Copyright (c) 2007 Kaazing Corporation. All rights reserved.

## Table of Contents

  * [Introduction](#introduction)
    * [Background](#background) 
    * [Protocol Overview](#protocol-overview) 
    * [Design Philosophy](#design-philosophy) 
    * [Security Model](#security-model) 
    * [Relationship to TCP and HTTP](#relationship-to-tcp-and-http) 
    * [Establishing a Connection](#establishing-a-connection) 
    * [Subprotocols using the WebSocket Emulation Protocol](#subprotocols-using-the-websocket-emulation-protocol) 
  * [Conformance Requirements](#conformance-requirements)
  * [WebSocket Emulation URIs](#websocket-emulation)
  * [Opening Handshake](#opening-handshake) 
    * [Client Requirements](#client-requirements)
    * [Server Requirements](#server-requirements)
  * [Data Frames](#data-frames) 
  * [Control Frames](#control-frames) 
  * [Closing Handshake](#closing-handshake) 
  * [Error Handling](#error-handling) 
  * [Extensions](#extensions) 
  * [Browser Considerations](#browser-considerations)
    * [Content Type Sniffing](#content-type-sniffing)
    * [Binary as Text](#binary-as-text) 
    * [Binary as Escaped Text](#binary-as-escaped-text) 
    * [Binary as Mixed Text](#binary-as-mixed-text) 
    * [Garbage Collection](#garbage-collection)
  * [Proxy Considerations](#proxy-considerations)
    * [Buffering Proxies](#buffering-proxies)
    * [Destructive Proxies](#destructive-proxies)
  * [Security Considerations](#security-considerations) 
  * [References](#references) 

## Introduction

### Background

_This section is non-normative._

Although all major browser vendors have endorsed the WebSocket specification, deployment of Web applications requires a 
solution for older browsers that have yet to be upgraded.  In addition, plug-in technologies such as Flash, Silverlight 
and Java did not initially support WebSocket.  It is anticipated that native support will be made available for all of these 
technologies in future.  This situation presents adoption problems for WebSocket until all end-users have upgraded accordingly.

The WebSocket Emulation protocol addresses these adoption issues by defining a protocol that depends only on typical application
usage of HTTP with solutions for common limitations of HTTP client implementations.

### Protocol Overview

_This section is non-normative._

The WebSocket Emulation protocol is layered on top of HTTP and has three parts: an HTTP handshake, an HTTP client-to-server 
(upstream) data transfer, and an HTTP server-to-client (downstream) data transfer.

### Design Philosophy

_This section is non-normative._

The WebSocket Emulation (WSE) protocol is designed to support a client-side WebSocket API with identical semantics
when compared to a WebSocket API using the WebSocket protocol defined by [RFC 6455](https://tools.ietf.org/html/rfc6455) while
making use of (perhaps limited) HTTP APIs only at the client, and a corresponding WebSocket Emulation server implementation.

For example, it should be possible to provide a compatible [W3C WebSocket API](http://www.w3.org/TR/websockets/) in JavaScript
using only an HTML4 browser's HTTP APIs to implement the WebSocket Emulation protocol. Notably, the end-user should not be 
required to install any browser plug-ins, instead the emulated WebSocket API should be delivered as part of the Web application 
JavaScript, perhaps using an HTML `<script>` tag.

The WebSocket Emulation protocol overhead is kept to a minimum such that performance and scalability can be considered 
approximately equivalent when compared to the WebSocket protocol defined by [RFC 6455](https://tools.ietf.org/html/rfc6455).

### Security Model

_This section is non-normative._

WebSocket Emulation protocol is layered on top of HTTP and therefore inherits the [CORS](http://www.w3.org/TR/cors) origin model 
when used from browser clients.

### Subprotocols using the WebSocket Emulation Protocol

_This section is non-normative._

The client can request that the server use a specific subprotocol by including the `X-WebSocket-Protocol` field in its handshake.
If it is specified, the server needs to include the same field and one of the selected subprotocol values in its response for
the connection to be established.

These subprotocol names should following the guidelines described by the WebSocket protocol in 
[RFC 6455, Section 1.9, paragraphs 2 and 3](https://tools.ietf.org/html/rfc6455#section-1.9). 

## Conformance Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and 
"OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

## WebSocket Emulatation URIs

The WebSocket Emulation protocol uses WebSocket protocol URIs, as defined by
[RFC 6455, Section 3](https://tools.ietf.org/html/rfc6455#section-3).

## Opening Handshake

### Client Handshake Requirements

To establish an emulated WebSocket connection, a client makes an HTTP handshake request.

* the HTTP handshake request method MUST be `POST` 
* the HTTP handshake request uri scheme MUST be derived from the WebSocket URL by changing `ws` to `http`, or `wss` to `https`. 
* the HTTP handshake request uri path MUST be derived from the WebSocket URL by appending a suitable create encoding path suffix
  * `/;e/cb` for binary encoding
  * `/;e/ct` for text encoding  (see [Binary as Text](#binary-as-text))
  * `/;e/cte` for text/escaped encoding  (see [Binary as Escaped Text](#binary-as-escaped-text)) 
  * `/;e/ctm` for text/mixed encoding  (see [Binary as Mixed Text](#binary-as-mixed-text))
* the HTTP handshake request path query parameters MUST include all query parameters from the WebSocket URL

Browser clients MUST sent the `Origin` HTTP header with the source origin.

Clients MUST sent the `X-WebSocket-Version` HTTP header with the value `wseb-1.1`.

Clients SHOULD send the `X-Accept-Commands` HTTP header with the value `ping` to indicate that both `PING` and `PONG` frames 
are understood by the client.

Clients MUST send an empty handshake request body.

For example, given the WebSocket Emulation URL `ws://host.example.com:8080/path?query` and binary encoding.

```
POST /path/;e/cb?query HTTP/1.1
Host: host.example.com:8080
Origin: [source-origin]
Content-Length: 0
X-WebSocket-Version: wseb-1.1
X-Accept-Commands: ping

```

When the handshake request is sent, the emulated WebSocket is in the `CONNECTING` state.

A successful WebSocket Emulation handshake response has status code `201` and the response body with content type 
`text/plain;charset=utf-8` consisting of two lines separated by a `\n` linefeed character.

The first line is an HTTP(S) URL for upstream data transfer of the emulated WebSocket connection.

The second line is an HTTP(S) URL for upstream data transfer of the emulated WebSocket connection.

The upstream and downstream data transfer URLs MAY use different ports than the original WebSocket URL, and they MAY each 
optionally include query parameters.

For any handshake response status code other than `201`, clients MUST fail the emulated WebSocket connection.

For any handshake response content type other than `text/plain;charset=utf-8`, clients MUST fail the emulated WebSocket 
connection.

For any handshake response `X-WebSocket-Version` HTTP header value not matching the value sent in the handshake request, clients 
MUST fail the emulated WebSocket connection.

For an upstream data transfer URL scheme other than `http` or `https`, clients MUST fail the emulated WebSocket 
connection.

For an upstream data transfer URL host not matching the host of the original WebSocket URL, clients MUST fail the emulated 
WebSocket connection.

For an upstream data transfer URL path not prefixed by the path of the original WebSocket URL, clients MUST fail the emulated 
WebSocket connection.

```
HTTP/1.1 201 Created
X-WebSocket-Version: wseb-1.1
Content-Type: text/plain;charset=utf-8
Content-Length: 105

http://host.example.com:8080/path/uofdbnreiodfkbqi
https://host.example.com:8443/path/kwebfbkjwehkdsfa
```

The emulated WebSocket is now in the `CONNECTED` state.

### Server Handshake Requirements

When a client starts an emulated WebSocket connection, it sends an HTTP handshake request.  The server must parse and and process
the handshake request to generate a handshake response.

If the server determines that any of the following conditions are not met by the HTTP handshake request, then the server MUST
send an HTTP response with a `4xx` status code, such as `400 Bad Request`.

* the HTTP handshake request method MUST be `POST` 
* the HTTP handshake request header `X-WebSocket-Version` MUST have the value `wseb-1.1`
* the HTTP handshake request header `X-Accept-Commands` is OPTIONAL, and when present MUST have the value `ping`

If the `X-Accept-Commands` HTTP header is present, then the server MAY send `PING` and `PONG` frames to the client.

The server SHOULD ignore any request body and MAY choose to enforce a maximum handshake request body length.

If any of the above conditions are not met, the server MUST reject the handshake request with a `400 Bad Request` status code.

Otherwise, the server MUST generate a HTTP handshake response with the following characteristics.
* the HTTP handshake response status MUST be `201`
* the HTTP handshake response header `X-WebSocket-Version` MUST have the value `wseb-1.1`
* the HTTP handshake response header `Content-Type` MUST have the value `text/plain;charset=utf-8`
* the HTTP handshake response body must contain two URLs separated by a `\n` (LF) character

The first URL in the response body is the upstream data transfer URL.

The second URL in the response body is the downstream data transfer URL.

Each URL may change the handshake HTTP request scheme from `http` to `https` and select a different port number, but the host
MUST remain the same as the host for the HTTP handshake request.

If a `;` is present in the original handshake request URL path then each URL MUST consist of a unique path prefixed by the 
original handshake request URL path, up to but not including the `;`.

If no `;` is present in the original handshake request URL path then each URL MUST consist of a unique path prefixed by the 
original handshake request URL path.

## Attaching the Downstream Response

### Downstream Client Requirements

Once the emulated WebSocket connection is established, the client MUST send an HTTP request for downstream data
transfer.
* the HTTP downstream request method MUST be `GET` 

The downstream request associates a continuously streaming HTTP response to the emulated WebSocket connection.

For example, with a downstream data transfer URL `https://host.example.com:8443/path/kwebfbkjwehkdsfa`.

```
GET /path/kwebfbkjwehkdsfa HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]

```

Receiving a downstream response status code of `200`, complete with all HTTP headers, indicates that the downstream response is 
ready to deliver emulated WebSocket frames to the client.

For any downstream response status code other than `200`, the client MUST fail the emulated WebSocket connection.

For any binary downstream response content type other than `application/octet-stream`, the client MUST fail the emulated 
WebSocket connection.

```
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Connection: close

[emulated websocket frames]
```

The downstream response MAY use `Connection: close` or `Transfer-Encoding: chunked` to provide a continuously streaming response
to the client.

See [Buffering Proxies](#buffering-proxies) for further client requirements when attaching the downstream.

### Downstream Server Requirements

When processing a binary HTTP downstream request the server generates an HTTP downstream response for the emulated WebSocket.

See [Binary as Text](#binary-as-text) for details of processing a text HTTP downstream request.

See [Binary as Escaped Text](#binary-as-escaped-text) for details of processing an escaped text HTTP downstream request.

See [Binary as Mixed Text](#binary-as-mixed-text) for details of processing a mixed text HTTP downstream request.

If the emulated WebSocket cannot be located for the HTTP downstream request path, then the server MUST generate an HTTP response
with a `404 Not Found` status code.

If the `.ki` query parameter is present with value `p`, see [Buffering Proxies](#buffering-proxies) for further server 
requirements when attaching the downstream.

Otherwise, if `.ki` query parameter is _not_ present with the value `p`, the server generates an HTTP downstream response as
follows.
* the downstream HTTP response status code MUST have the value `200`
* the downstream HTTP `Content-Type` header MUST have the value `application/octet-stream` 
* the downstream HTTP `Connection` header MUST have the value `close` 

The server MUST flush these HTTP response headers to the client even before data is available to send to the client.

If the `.kp` parameter is present, see [Content-Type Sniffing](#content-type-sniffing) for further server requirements when 
attaching the downstream.

## Sending the Upstream Request

### Upstream Client Requirements

Any upstream data frames are sent in the payload of a transient HTTP upstream request.
* the HTTP upstream request method MUST be `POST`
* the HTTP upstream request `Content-Type` HTTP header MUST be `application/octet-stream`

For example, with an upstream data transfer URL `http://host.example.com:8080/path/uofdbnreiodfkbqi`.
```
POST /path/uofdbnreiodfkbqi HTTP/1.1
Host: host.example.com:8080
Origin: [source-origin]
Content-Type: application/octet-stream
Content-Length: [size]

[emulated websocket frames]
```

An upstream response status code of `200` indicates that the frames were received successfully by the emulated WebSocket 
connection.

For any upstream response status code other than `200`, the client MUST fail the emulated WebSocket connection.

The client MUST NOT send another upstream request before the previous upstream response has completed.

### Upstream Server Requirements

When processing a binary HTTP upstream request the server generates an HTTP upstream response for the emulated WebSocket.

See [Binary as Text](#binary-as-text) for details of processing a text HTTP upstream request.

See [Binary as Escaped Text](#binary-as-escaped-text) for details of processing an escaped text HTTP upstream request.

See [Binary as Mixed Text](#binary-as-mixed-text) for details of processing a mixed text HTTP upstream request.

If the emulated WebSocket cannot be located for the HTTP upstream request path, then the server MUST generate an HTTP response
with a `404 Not Found` status code.

If the emulated WebSocket is already processing an HTTP upstream request, then the server MUST generate an HTTP response
with a `400 Bad Request` status code and fail the emulated WebSocket connection.

Otherwise, the server decodes the emulated WebSocket frames from the upstream request body and generates an HTTP upstream 
response as follows.
* the upstream HTTP response body MUST have status code `200` 
* the upstream HTTP response body MUST be empty with a `Content-Length` header value of `0` 

The server MAY choose to delay the HTTP upstream response until some or all of the emulated WebSocket frames from the upstream
request body have been processed to throttle the emulated WebSocket upstream from the client.

## Data Frames

The client requirements for data frame syntax are defined by 
[Draft-76, Section 4.2](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76#section-4.2).

The server requirements for data frame syntax are defined by 
[Draft-76, Section 5.3](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76#section-5.3).

## Command Frames

The frames used by an emulated WebSocket connection for upstream and downstream data transfer extend those defined by 
[Draft-76](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76), by adding a command frame.

The text-based command frame has frame type `0x01`, which masks to `0x00` indicating text content.

The content of each command frame is a sequence of bytes encoded as hex.  For example, the bytes `5` `B` (`0x35 0x42`) decode to 
the hypothetical command hex code `0x5B`. 

Command hex code `0x00` (`0x30 0x30`) indicates padding and heart beat and is therefore ignored.
Command hex code `0x01` (`0x30 0x31`) is the reconnect command used for controlled reconnect.
Command hex code `0x02` (`0x30 0x32`) is the close command.

## Control Frames

The frames used by an emulated WebSocket connection for upstream and downstream communication extend those defined by 
[Draft-76](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76), by adding `PING` and `PONG` control frames.

These control frames have the leading bit set, which in [Draft-76](http://tools.ietf.org/html/draft-hixie-thewebsocketprotocol-76)
indicates a binary frame payload.

The `PING` control frame code is `0x89`. This is the same frame code as `PING` in [RFC 6455](http://tools.ietf.org/html/rfc6455))
but only supports a zero length payload.

The `PONG` control frame code is `0x8A`. This is the same frame code as `PONG` in [RFC 6455](http://tools.ietf.org/html/rfc6455))
but only supports a zero length payload.

If a client does not include the HTTP `X-Accept-Commands` header with a value of `ping` in the handshake request, then the 
server MUST NOT send a `PING` or `PONG` frame to the client.

If a client does not include the HTTP `X-Accept-Commands` header with a value of `ping` in the handshake request, then the 
server SHOULD fail the emulated WebSocket connection if it receives a `PING` or `PONG` frame from the client.

## Closing Handshake

### Client-initiated Close
Client sends a text-based `CLOSE` command frame on the upstream and closes the stream. Client’s readyState is set to `CLOSING`.

The Gateway responds by closing the downstream.  The Gateway transmits an emulated text-based command frame on the downstream 
with a `CLOSE` command (`0x02`) followed by a `RECONNECT` (`0x01`), then ends the stream.

The client verifies that the Gateway has responded with `CLOSE` followed by `RECONNECT` followed by termination of the stream. 
Finally, the client fires a close event as defined by the WebSocket API.  Otherwise, it is a protocol violation, and an error 
event is fired.

Note that this section is extended by close code and reason, as described in the Close Handshake section below.

### Server-initiated Close
Server transmits a `CLOSE` command (`0x02`) followed by a `RECONNECT` (`0x01`), then ends the stream.

The client verifies that the downstream `CLOSE` command followed by the `RECONNECT` command and end of stream, then the emulated
WebSocket is in the `CLOSED` state.

### Native WebSocket Protocol Close Frame Sample

The following close handshake frame samples are captured with Chrome WebSocket

#### Case #1: 

Client called Close() without parameter, the frame is `0x88 0x80 0x16 0x8f 0x0c 0x0a`
`0x88` - close frame
`0x80` - mask bit (first bit = 1) + data length (equals to 0)
`0x16 0x8f 0x0c 0x0a` - mask key (4 bytes random generated by client)

Server responded with the same data without mask, the frame is `0x88 0x00`
`0x88` - close frame
`0x00` - mask bit (first bit = 0) + data length (equals to 0)

Client fired CLOSE Event
```javascript
evt.wasClean = true;
evt.code = 1005
evt.reason = “”
```

#### Case #2: 
Client called close with parameters, such as Close(1000, “abc”), the frame is `0x88 0x85 0x78 0x20 0xef 0x1c 0x7b 0xc8c 0x8e 0x7e 0xeb`

`0x88` - close frame
`0x85` - mask bit (first bit = 1) + data length (equals to 5)
`0x78 0x20 0xef 0x1c` - mask key (4 bytes)
`0x7b 0xc8` - first parameter [code] (unsigned short, equals to 1000 or range from 3000 to 4999)
`0x8e 0x7e 0xeb` - second parameter [reason] (UTF-8 encoded, no longer than 123 bytes)

Server responded with same data without mask, the frame is  `0x88 0x05 0x03 0xe8 0x61 0x62 0x63`

`0x88` - close frame
`0x05` - mask bit (first bit = 0, no mask) + data length (equals to 5)
`0x03 0xe8` - code (value equals to 1000)
`0x61 0x62 0x63` - reason (value equals to “abc”)

Clent fired CLOSE event
```javascript
evt.wasClean = true
evt.code = 1000
evt.reason = “abc”
```

## Error Handling

### Client-initiated Error
If the Client detects a protocol violation, then it will shutdown all outstanding requests, and fire events as required by 
WebSocket API.

### Server-initiated Error
Server terminates the downstream with no `RECONNECT` or `CLOSE` frame.

Alternatively, the server can respond with an HTTP error-code on any HTTP create, upstream or downstream request.

### Connection Loss
The semantics of connection loss for emulated WebSocket match those of native WebSocket.  The lifetime of the downstream 
indicates the lifetime of the emulated WebSocket.  Note that the downstream can span more than one HTTP request due to either 
a garbage collecting reconnect or buffering proxy detection.  If the downstream response completes without a “reconnect” command
frame, then the WebSocket connection is closed.

## Extensibility

[TODO]
`X-WebSocket-Protocol`
`X-WebSocket-Extensions`
same as RFC 6455, X-WebSocket-Extensions, no RSV bits

## Proxy Considerations

### Buffering Proxies
Some HTTP intermediaries, such as transparent and explicit proxies, prevent HTTP responses from being successfully streamed to 
the client.  Instead, such intermediaries may buffer the response introducing undesirable latency.  Once detected, there are two
possible solutions; long-polling, and secure streaming.

We detect the presence of a buffering proxy by timing out the arrival of the status code and complete headers from the streaming 
downstream HTTP response.  In this case some emulated WebSocket frames may be blocked at the proxy, so to avoid data loss, we 
send a second overlapping downstream HTTP request, with the `.ki=p` query parameter to indicate the interaction mode is “proxy”.

In reaction to receiving this overlapping proxy mode downstream request for the same emulated WebSocket connection, the initial 
downstream response is completed normally, automatically flushing the entire response through any buffering proxy.  Then, the 
response to the second downstream request is to either redirect to an encrypted location for secure streaming, or fall back to 
long-polling as a last resort.

When long-polling, any frame sent to the client triggers a controlled reconnect in the same way as garbage collection.

### Destructive Proxies
Some HTTP intermediaries, such as transparent and explicit proxies, attempt to reclaim resources for idle HTTP responses that are
still incomplete after a timeout period.  A heartbeat command frame must be sent to the client at regular intervals to prevent 
the downstream from being severed by the proxy.

When long-polling, the heartbeat frame triggers a controlled reconnect.

Clients MAY request a shorter heartbeat interval by setting the `.kkt` query parameter. For example, `?.kkt=20` requests a 20 
second interval. This is necessary on user agents that shut down HTTP requests after only 30 seconds.

## Browser Considerations

### Content-Type Sniffing
The programmatic HTTP client runtime provided by many browser environments will attempt to determine the “real” content type of
a response served with explicit content type `text/plain`.  

The browsers achieve this by waiting for a certain amount of content to arrive for content type analysis before exposing any of
the content to the client runtime.  This prevents delivery of any emulated WebSocket frames that happen to be smaller than the
buffer size.  The exact buffer size can be determined by experimentation for different browser implementations.  In some cases 
however it is not necessary.

Padding the beginning of the response with additional control frames to exceed the content sniffing buffer size allows immediate
delivery of emulated WebSocket frames delivered to such clients.  The maximum amount of padding required is communicated in the
request via the `.kp` query parameter.
```
GET /path/kwebfbkjwehkdsfa?.kp=256 HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]
```
IE8+ supports a special HTTP response header that can be used to switch off content type sniffing, thus eliminating the need to
send any extra padding at the beginning of the response.
```
HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
Connection: close
X-Content-Type-Options: nosniff

[emulated websocket frames]
```
Even when padding the body with command frames, IE7 determines the downstream response content type to be 
`application/octet-stream`, preventing programmatic access to the response text from JavaScript.

This issue only comes up in a special case of HTTP request emulation [HTTPE], where response headers are promoted to the 
beginning of the response body anyway, so setting a sufficiently long header with text value fills the content type sniffing 
buffer at the client side, allowing the text-based content type to be properly detected and proving programmatic access to the 
response text from IE7’s JavaScript runtime. 
```
GET /path/kwebfbkjwehkdsfa?.kns=1 HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]
```
The behavior to inject a long text header is triggered by the presence of the `.kns=1` query parameter above.
```
HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
Connection: close
X-Content-Type-Options: nosniff

HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
X-Content-Type-Nosniff: [long text string]

[emulated websocket frames]
```

### Binary as Text
The JavaScript execution environment provided by browsers did not historically have support for binary data types, so all HTTP 
responses accessible to JavaScript were described as strings.

```
POST /path/;e/ct HTTP/1.1
Host: host.example.com:8000
Origin: [source-origin]
Content-Length: 0

```
Here, the handshake request location path uses `/;e/ct` instead

```
HTTP/1.1 201 Created
X-WebSocket-Version: wseb-1.1
Content-Type: text/plain;charset=utf-8
Content-Length: 105

http://host.example.com:8080/path/uofdbnreiodfkbqi
https://host.example.com:8443/path/kwebfbkjwehkdsfa
```

The binary-as-text downstream HTTP response MUST have content type `text/plain;charset=windows-1252`.
```
GET /path/kwebfbkjwehkdsfa HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]

```
```
HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
Connection: close

[emulated websocket frames]
```
The charset `windows-1252` [Windows-1252](http://en.wikipedia.org/wiki/Windows-1252) allows bytes to be transferred by the server
without modification because each character is represented by a single byte and exactly 256 distinct character codes are 
available, and 224 of those character codes match their byte representations when processed by the client.  The remaining 
character codes can be mapped to the corresponding binary representations accordingly.

### Binary as Escaped Text

Some browsers have difficulty representing a few specific character codes in the HTTP response text.  For example, IE6+ will 
interpret character code zero as end-of-response, truncating any body content following that character.  IE6+ also automatically
canonicalizes carriage return and linefeed characters as all carriage returns.  Conversely, IE9 strict document mode 
canonicalizes carriage return and linefeed characters as all linefeeds.

Therefore, the server MUST escape these characters must as follows:

| Byte Value | Character | Escaped Byte Sequence | Escaped Characters |
|------------|-----------|-----------------------|--------------------|
| 0x00       | \0        | 0x7f 0x3f             | DEL 0              |
| 0x0d       | \r        | 0x7f 0x72             | DEL r              |
| 0x0a       | \n        | 0x7f 0x63             | DEL n              |
| 0x7f       | DEL       | 0x7f 0x7f             | DEL DEL            |

For clients requiring escaped text responses, the initial handshake uses a different derived location path.
```
POST /path/;e/cte HTTP/1.1
Host: host.example.com:8000
Origin: [source-origin]
Content-Length: 0

```
Here, the handshake request location path uses `/;e/cte` instead
```
HTTP/1.1 201 Created
X-WebSocket-Version: wseb-1.1
Content-Type: text/plain;charset=utf-8
Content-Length: 105

http://host.example.com:8080/path/uofdbnreiodfkbqi
https://host.example.com:8443/path/kwebfbkjwehkdsfa
```

The binary-as-escaped-text downstream HTTP response MUST have content type `text/plain;charset=windows-1252`.
```
GET /path/kwebfbkjwehkdsfa HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]

```
```
HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
Connection: close

[emulated websocket frames]
```

However, the client MUST also unescape the `DEL 0`, `DEL r`, `DEL n` and `DEL DEL` escaped character sequences to
get the logical bytes transfered.

### Binary as Mixed Text

For clients requiring mixed text responses, the initial handshake uses a different derived location path.
```
POST /path/;e/ctm HTTP/1.1
Host: host.example.com:8000
Origin: [source-origin]
Content-Length: 0

```
Here, the handshake request location path uses `/;e/ctm` instead
```
HTTP/1.1 201 Created
X-WebSocket-Version: wseb-1.1
Content-Type: text/plain;charset=utf-8
Content-Length: 105

http://host.example.com:8080/path/uofdbnreiodfkbqi
https://host.example.com:8443/path/kwebfbkjwehkdsfa
```

The binary-as-mixed-text downstream HTTP response MUST have content type `text/plain;charset=windows-1252`.
```
GET /path/kwebfbkjwehkdsfa HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]

```
```
HTTP/1.1 200 OK
Content-Type: text/plain;charset=windows-1252
Connection: close

[emulated websocket frames]
```

IE8+ `XDomainRequest` can distinguish all 256 byte-as-character-code values, for a downstream response body with content-type 
`text/plain;charset=windows-1252`.

However, the `XDomainRequest` `POST` request cannot specify content-type, and so `text/plain;charset=UTF-8` is assumed, and a
bug remains where the `\0` (NUL) character cannot be included in the `POST` body since the text is then clipped at the `\0`.

Rather than escaping the `\0` (NUL) byte-as-character-code, the client MUST use character code `256` to represent zero, so the
client MUST send a `\u0100` character for each `\u0000` character.

The server MUST decode the binary-as-mixed-text upstream such that each character codepoint is computed `mod 0x0100` to determine 
each originally intended byte value.

*Note* This approach remains untested for IE6+ downstream scenario as an alternative to the escaping strategy above for `\0`, 
`\r` and `\n`, but may very well be more efficient than what we have today.

### Garbage Collection
Most HTTP client runtimes such as Flash, Silverlight and Java have a streaming capable HTTP response implementation, meaning that
fragments of the response can be consumed without waiting for the entire document to be transferred.

The JavaScript HTTP client runtime provided by most browsers is a little different.  A notification is provided each time new 
data arrives in the response body, but the aggregated response text still builds up.  When a certain amount of data builds up on 
the client, it is beneficial to let that response complete normally, letting the browser reclaim all memory associated with the 
aggregated response, and then reconnect the downstream with a new HTTP request.

The amount of data that the client is willing to build up between garbage collecting reconnects is defined in kilobytes by the 
`.kb` query parameter in the corresponding request.
```
GET /path/kwebfbkjwehkdsfa?.kb=512 HTTP/1.1
Host: host.example.com:8443
Origin: [source-origin]
```
When the limit is exceeded, a `RECONNECT` command frame is sent by the server to complete the response.  The `RECONNECT` frame 
MUST be the last frame on the downstream response during a reconnect, otherwise undetectable loss could occur for subsequent 
frames.

**enhancement**
Client may choose not to supply the `.kb` parameter. Instead, client establishes a new downstream connection when it decides it 
needs to (based on how much data has already been buffered on the current connection, and the network connect latency). When the
server detects the new downstream connection, it will send a `RECONNECT` command on the current downstream connection, then close 
it, and write all further data messages to the new downstream connection. 

Note: no data frames, in fact no frames at all, will be sent after the `RECONNECT` frame.

Note: the `.kb` parameter is still supported for backward compatibility.

## Security Considerations

[TODO]

## References

[RFC2616](http://tools.ietf.org/html/rfc2616)  "Hypertext Transfer Protocol -- HTTP/1.1"

[RFC6455](http://tools.ietf.org/html/rfc6455)  "The WebSocket Protocol"
