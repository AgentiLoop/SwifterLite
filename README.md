![Platform](https://img.shields.io/badge/Platform-iOS%20%7C%20macOS%20%7C%20tvOS-4BC51D.svg?style=flat)
![Swift](https://img.shields.io/badge/Swift-5.6%2B-4BC51D.svg?style=flat)
![Protocol](https://img.shields.io/badge/Protocol-HTTP%2F1.1-4BC51D.svg?style=flat)

# SwifterLite

SwifterLite is a small HTTP/1.1 server you can embed in an iOS, macOS or tvOS app. It is a cut-down fork of [Swifter](https://github.com/httpswift/swifter) by Damian Kołakowski, built for one job: serving a local REST API and HLS audio to `AVPlayer` inside the same app, quickly and reliably.

StarPlayrX uses it to run a server on `localhost`. The player loads playlists, AES keys and AAC segments from that server, and the server fetches them from upstream.

SwifterLite is about half the size of Swifter. Many Swifter features were removed, so it can't be merged back into Swifter.

## Features

- **GET and POST** routes. Each method has its own route table.
- **Path parameters** such as `/m3u/:channelid`.
- **Keep-alive** connections (`Connection: keep-alive`).
- **Four body types**: JSON, plain text, `Data` and `[UInt8]`, each with an optional content type.
- **Custom response headers**.
- **One thread per connection**. Each accepted socket is handled on a global dispatch queue.
- **No dependencies**. It uses only Foundation and Darwin sockets.

### What it doesn't do

- **No HTTPS.** It only speaks plain HTTP. Put a proxy such as nginx or Cloudflare in front of it if you need TLS.
- **IPv4 only.**
- **Two status codes**: `200 OK` and `404 Not Found`.
- **No** PUT/DELETE, WebSockets, static file serving, chunked transfer encoding or middleware.
- **Apple platforms only.** It calls Darwin directly, so it won't build on Linux.

> **Note:** The listening socket binds to `INADDR_ANY` (all interfaces), not just `127.0.0.1`, so other devices on the network can reach it. Keep that in mind when you choose what to serve.

## Installation

Swift Package Manager:

```swift
dependencies: [
    .package(url: "https://github.com/AgentiLoop/SwifterLite", branch: "mustang")
],
targets: [
    .target(
        name: "YourTarget",
        dependencies: [.product(name: "SwifterLite", package: "SwifterLite")]
    )
]
```

In Xcode, use **File > Add Package Dependencies…**, enter the URL above and choose the `mustang` branch.

## Quick start

```swift
import SwifterLite

let server = HttpServer()

server.get["/api/v3/ping"] = { request in
    HttpResponse.ok(.ping("pong"))
}

try server.start(8080)   // listens on port 8080 and returns right away
// ...
server.stop()
```

`start(_:priority:)` opens the socket and returns. Connections are accepted on a background queue (QoS `.userInteractive` by default), and the server keeps running until you call `stop()` or the `HttpServer` is deallocated, so keep a strong reference to it.

```bash
curl -H 'Connection: keep-alive' http://localhost:8080/api/v3/ping   # pong
```

> **Note:** Send `Connection: keep-alive`. Without it the server closes the socket right after writing the response, and because sockets use `SO_LINGER` 0 the close is a reset, so the client can lose the body (curl reports error 56). `AVPlayer` sends keep-alive, so StarPlayrX isn't affected.

## Routing

Assign a handler to a path on `server.get` or `server.post`. A handler is a `httpReq`, which is a `(HttpRequest) -> HttpResponse` closure.

```swift
server.get ["/api/v3/key"]            = keyRoute()
server.post["/api/v3/login"]          = loginRoute()
server.get ["/api/v3/m3u/:channelid"] = playlistRoute()
```

The usual pattern is a function that returns the closure, as StarPlayrX does:

```swift
func keyRoute() -> httpReq {{ request in
    guard let data = Data(base64Encoded: key) else {
        return HttpResponse.notFound(.none)
    }
    return HttpResponse.ok(.data(data, contentType: "application/octet-stream"))
}}
```

### Path parameters

A path segment that starts with `:` is a variable. Read it from `request.params`. **The key keeps the colon:**

```swift
server.get["/api/v3/aac/:aac"] = { request in
    guard let segment = request.params[":aac"] else {
        return HttpResponse.notFound(.none)
    }
    // ...
}
```

A variable only matches the **last** segment of a route. Requests that don't match any route get `404 Not Found`.

### Listing routes

`server.routes` returns every node in the route tree as a string. The method is **not** included, intermediate segments are listed too, and the order is not fixed:

```swift
print(server.routes.sorted())   // ["/api", "/api/v3", "/api/v3/login", "/api/v3/ping", ...]
```

## Requests

`HttpRequest` has these properties:

| Property      | Type                 | Notes                                             |
|---------------|----------------------|---------------------------------------------------|
| `method`      | `String`             | `"GET"` or `"POST"`                               |
| `path`        | `String`             | Path without the query string                     |
| `queryParams` | `[(String, String)]` | Query items in their original order               |
| `headers`     | `[String: String]`   | Header names are **lowercased**                   |
| `body`        | `[UInt8]`            | Read using `Content-Length`                       |
| `params`      | `[String: String]`   | Path parameters, keyed with the colon (`":id"`)   |
| `address`     | `String?`            | Defaults to `"127.0.0.1"`                         |

Parsing a JSON POST body:

```swift
server.post["/api/v3/session"] = { request in
    let json = try? JSONSerialization.jsonObject(with: Data(request.body)) as? [String: String]
    guard let channelId = json?["channelid"] else { return HttpResponse.notFound(.none) }
    return HttpResponse.ok(.json(["success": true, "data": channelId]))
}
```

## Responses

```swift
public enum HttpResponse {
    case ok(HttpResponseBody, [String: String] = [:])   // 200, with optional extra headers
    case notFound(HttpResponseBody? = nil)              // 404
}

public enum HttpResponseBody {
    case json(Any,     contentType: String? = "application/json")
    case ping(String,  contentType: String? = "text/plain")
    case data(Data,    contentType: String? = nil)
    case byts([UInt8], contentType: String? = nil)
}
```

Examples:

```swift
HttpResponse.ok(.json(["success": true]))
HttpResponse.ok(.ping(playlistText, contentType: "application/x-mpegURL"))
HttpResponse.ok(.data(aacSegment, contentType: "audio/aac"), ["X-Channel-ID": channelId])
HttpResponse.ok(.byts([0x00, 0x01], contentType: "application/octet-stream"))
HttpResponse.notFound(.none)
```

`.json` is encoded with `JSONSerialization`. If encoding fails, the body becomes the text `Http error: …`.

Every response gets a `Content-Length` header and a `Server: SwifterLite mustang` header. A `Connection: keep-alive` header is added when the client asked for keep-alive.

## More examples

Every example below was compiled and run against the `mustang` branch.

### 1. Start on a free port and read it back

Pass `0` and the system picks a port. `port()` returns it, and `state` tracks the lifecycle.

```swift
let server = HttpServer()
try server.start(0)
let port = try server.port()   // e.g. 50367
print(server.state)            // running
server.stop()
print(server.state)            // stopped
```

### 2. Choose the accept-queue priority

```swift
try server.start(8080, priority: .utility)   // default is .userInteractive
```

### 3. Handle start errors

`start` throws a `SocketError`. Each case carries `errno` as a string.

```swift
do {
    try server.start(8080)
} catch SocketError.bindFailed(let errno) {
    print("bind failed, errno \(errno)")
} catch {
    print("could not start: \(error)")
}
```

Because the listening socket sets `SO_REUSEPORT`, starting a second server on a port that's already in use **does not** throw; both sockets end up sharing the port.

### 4. Query parameters

`queryParams` is an array of `(name, value)` tuples, so repeated names are kept.

```swift
server.get["/api/v3/search"] = { request in
    let q = request.queryParams.first { $0.0 == "q" }?.1 ?? ""
    let all = Dictionary(request.queryParams, uniquingKeysWith: { first, _ in first })
    return .ok(.json(["q": q, "count": all.count]))
}
// GET /api/v3/search?q=jazz&q=rock&x=1  ->  {"q":"jazz","count":2}
```

### 5. Request headers

Header names are lowercased.

```swift
server.get["/api/v3/agent"] = { request in
    let agent = request.headers["user-agent"] ?? "unknown"
    let range = request.headers["range"] ?? "-"
    return .ok(.ping("\(agent) \(range)"))
}
```

### 6. Echo the body with custom headers

```swift
server.post["/api/v3/echo"] = { request in
    .ok(.byts(request.body, contentType: request.headers["content-type"]),
        ["X-Method": request.method, "X-Path": request.path])
}
```

### 7. Register or disable a route through the router

`server.get` and `server.post` expose their `HttpRouter`. Setting a route to `nil` makes it return 404, though it stays in `server.routes`.

```swift
server.get.router.register("GET", path: "/api/v3/direct") { _ in .ok(.ping("direct")) }

server.get["/api/v3/old"] = { _ in .ok(.ping("old")) }
server.get["/api/v3/old"] = nil   // now 404
```

### 8. Look up a route without a request

`route(_:path:)` returns the matched path parameters and the handler, or `nil`.

```swift
if let (params, handler) = server.get.router.route("GET", path: "/api/v3/aac/seg1.aac") {
    print(params)   // [":aac": "seg1.aac"]
}
server.get.router.route("GET", path: "/api/v3/old") == nil   // true after example 7
```

### 9. Override `dispatch` in a subclass

Subclass `HttpServer` (its `init()` is public; `HttpServerIO`'s isn't) to log requests or answer some paths before the router runs.

```swift
final class LoggingServer: HttpServer {
    override func dispatch(_ request: HttpRequest) -> dispatchHttpReq {
        print(request.method, request.path)
        if request.path == "/health" {
            return ([:], { _ in .ok(.ping("ok")) })
        }
        return super.dispatch(request)
    }
}
```

### 10. Inspect a response

```swift
let response = HttpResponse.ok(.data(segment, contentType: "audio/aac"), ["Cache-Control": "no-cache"])
response.statusCode     // 200
response.reasonPhrase   // "OK"
response.headers()      // ["Cache-Control": "no-cache", "Content-Type": "audio/aac", "Server": "SwifterLite mustang"]

HttpResponse.notFound().headers()   // ["Server": "SwifterLite mustang"]
```

### 11. Version string

```swift
HttpServer.version   // "mustang"
```

### 12. Raw sockets

`Socket` can be used on its own. This echoes one line back to a client:

```swift
let listener = try Socket.tcpSocketForListen(9000)   // backlog defaults to SOMAXCONN
print(try listener.port())                           // 9000

DispatchQueue.global().async {
    guard let client = try? listener.acceptClientSocket() else { return }
    defer { client.close() }
    let line = (try? client.readLine()) ?? ""        // reads up to \n, drops \r
    try? client.writeUtf8("echo: \(line)\n")
}
```

Other calls: `read()` (one byte), `read(length:)`, `writeData(_:)`, `writeUInt8(_:)`, `Socket(socketFileDescriptor:)` to wrap an existing descriptor, and the class helpers `Socket.setNoSigPipe(_:)` and `Socket.close(_:)`. `ErrNumString.description()` returns the current `errno` as a string.

### Public but not usable from outside the module

- `HttpParser.readHttpRequest(_:)` is public, but `HttpParser` has no public initializer.
- `HttpRequest` has no public initializer, so you only get one inside a handler or `dispatch`.
- `HttpServerIODelegate` and `delegate` exist, but SwifterLite never calls `socketConnectionReceived(_:)`.
- `HttpResponseBodyWriter` is only used internally to write bodies. `SerializationError` isn't thrown anywhere.
- `request.address` is never set by the parser, so it's always `"127.0.0.1"`.

## Lower-level API

Most apps only need `HttpServer`, but these are public too:

- **`HttpServerIO`**: the accept loop and connection handling. Subclass `HttpServer` (not `HttpServerIO`, whose initializer is internal) and override `dispatch(_:)` to route requests yourself. `state` (`.starting`, `.running`, `.stopping`, `.stopped`) and `port()` report the server's status.
- **`HttpRouter` / `MethodRoute`**: the route tree and the `server.get[...]` / `server.post[...]` subscripts.
- **`Socket`**: a POSIX TCP socket with `tcpSocketForListen`, `acceptClientSocket`, `read`, `readLine`, `writeUtf8`, `writeData`, `writeUInt8` and `port()`. Sockets set `SO_REUSEADDR`, `SO_REUSEPORT`, `SO_LINGER` (0) and `SO_NOSIGPIPE`. These free the port as soon as a socket closes and stop the app from crashing on a broken pipe when it goes into the background.
- **`SocketError`**: errors thrown by socket calls. Each case includes `errno` as a string.

## Package layout

```
Sources/SwifterLite/
├── HttpServer.swift          HttpServer: GET/POST route tables, dispatch, version
├── HttpServerIO.swift        Accept loop, keep-alive, response writing, start/stop
├── HttpRouter.swift          Route tree and :param matching
├── MethodRoute.swift         server.get[...] / server.post[...] subscripts
├── HttpParser.swift          Request line, headers and body parsing
├── HttpRequest.swift         HttpRequest
├── HttpResponse.swift        HttpResponse (200 / 404) and headers
├── HttpRespBody.swift        HttpResponseBody (.json / .ping / .data / .byts)
├── HttpRespBodyWriter.swift  HttpResponseBodyWriter protocol
├── Socket*.swift             POSIX socket wrapper: open, accept, read, write, port, errors
└── TypeAliases.swift         httpReq, dispatchHttpReq, httpWriter
```

## Branches

`mustang` is the current branch and the one StarPlayrX uses. `horse` and `viper` are older branches whose last commits are from 2022.

## Used by

- **StarPlayrX** (iOS and macOS): the app's local streaming server lives in each app's `Routes/` folder, and it uses [StarPlayrRadioKit](https://github.com/AgentiLoop/StarPlayrRadioKit) for the upstream API calls.

## License

BSD 3-Clause. See [LICENSE](LICENSE).
Swifter © 2014–2016 Damian Kołakowski. SwifterLite changes © 2022 Todd Bruss.

---

**AgentiLoop:** [agentiloop.ai](https://agentiloop.ai/)
