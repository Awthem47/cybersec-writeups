# HTTP Protocol

HTTP (Hypertext Transfer Protocol) is the application-layer protocol that powers the web. This section covers how to identify and analyse HTTP traffic in Wireshark, including the request-response cycle, conditional requests, and the difference between persistent and non-persistent connections.

## Part 1 — The Basic GET/Response Cycle

When a browser visits a webpage, two things happen at the HTTP layer:

1. The browser sends an HTTP GET request asking for the resource
2. The server responds with an HTTP response containing status code and content

To isolate these in Wireshark, apply the filter `http`. Selecting a GET packet and expanding the HTTP section in the detail pane reveals the request line and headers.

### Identifying the HTTP version

The request line shows the version directly. A line of `GET /path HTTP/1.1` indicates HTTP/1.1, which is the most common version still in widespread use alongside HTTP/2.

### Identifying client language preferences

The `Accept-Language` header lists the languages the browser will accept, in order of preference. A typical value is `en-us,en;q=0.9`.

### Identifying client and server IP addresses

In the GET packet:

- Source IP = client (your computer)
- Destination IP = server

In the matching response packet, these are reversed: the server becomes the source and your computer becomes the destination.

### Identifying response status

The response packet's status line indicates how the server handled the request. The most common is `HTTP/1.1 200 OK`, meaning the request succeeded and the content follows.

### Identifying the last-modified date

The `Last-Modified` header in the response indicates when the resource was last changed on the server. This value is what the browser will quote back in subsequent conditional requests.

### Identifying response size

The `Content-Length` header indicates the number of bytes in the response body. In one lab capture this value was 128 bytes for a small text file.

## Part 2 — The Conditional GET

This is the mechanism that makes browser caching work efficiently.

### First visit to a page

1. Browser sends a GET request with no caching headers
2. Server responds with 200 OK and the full content
3. Browser stores the content and the `Last-Modified` date in its cache

### Second visit to the same page

1. Browser sends a GET request that includes `If-Modified-Since: [date from first response]`
2. Server checks whether the resource has changed since that date
3. If unchanged, server responds with `304 Not Modified` and no body — the browser uses its cached copy
4. If changed, server responds with `200 OK` and the new content

### Why this matters

Conditional GETs save bandwidth and reduce server load. Instead of re-downloading unchanged content, the browser and server exchange just enough information to confirm the cache is still valid.

### How to spot it in Wireshark

- First GET → no `If-Modified-Since` header → response is `200 OK` with content
- Second GET → has `If-Modified-Since` header → response is `304 Not Modified` with no body

## Part 3 — Persistent vs Non-Persistent Connections

A single web page typically references many separate resources: the HTML itself, CSS files, JavaScript files, images, fonts. How HTTP handles these multiple requests depends on the version in use.

### Non-persistent HTTP (HTTP/1.0 behaviour)

A new TCP connection is opened for every object. Each connection requires its own three-way handshake before any data can flow.

**Analogy:** Calling someone to ask one question, hanging up, calling again to ask another, hanging up, and so on. Each call gets a different connection — in Wireshark this appears as a different source port number for each request.

The overhead is significant. For a page with 30 resources, that means 30 separate three-way handshakes — 30 round trips before any useful data flows.

### Persistent HTTP (HTTP/1.1 default behaviour)

A single TCP connection stays open and is reused for multiple requests. The `Connection: Keep-Alive` header signals this intent.

**Analogy:** Calling someone once, asking all your questions in a single conversation, then hanging up. One call, one connection — in Wireshark this appears as the same source port number across multiple requests.

The improvement is substantial. The same 30-resource page now needs only one handshake, with all subsequent requests piggybacking on the established connection.

### How to spot it in Wireshark

- Look at the source port numbers across multiple GET requests
- Same port across requests → persistent connection (one TCP session reused)
- Different port for each request → non-persistent (new connection per object)

## Reference Tables

### HTTP status codes to recognise

| Code | Meaning |
| --- | --- |
| 200 | OK — request successful |
| 301 | Moved Permanently — resource has a new URL |
| 304 | Not Modified — cached copy is still valid |
| 400 | Bad Request — server could not parse the request |
| 404 | Not Found — resource does not exist |
| 500 | Internal Server Error — server-side fault |

### HTTP methods

The server can be thought of as a library, and the client as a visitor. HTTP methods describe the different requests a visitor can make.

| Method | Purpose | Example | Library analogy |
| --- | --- | --- | --- |
| **GET** | Retrieve a resource from the server | Loading a webpage | "Can I see that book?" — you are asking the server to give you something. You are not changing anything. |
| **POST** | Send data to the server for processing | Submitting a login form | "Here is a document I need you to file." You are sending the server new data to process. |
| **HEAD** | Like GET, but returns only the headers, no body | Checking if a page has been updated | "Is that book still on the shelf?" Same as GET, but the server returns only metadata. |
| **PUT** | Replace a resource entirely | Updating an entire user profile | "Replace this book with this new version." Different from POST: PUT places the resource at a specific location, replacing whatever was there. |
| **DELETE** | Remove a resource | Deleting a post | "Remove that book from the shelf." |

## Key Takeaways

- HTTP/1.1 is identified by the `HTTP/1.1` literal in the request and status lines
- Conditional GETs use `If-Modified-Since` to enable efficient caching
- Persistent connections (HTTP/1.1 default) reuse a single TCP connection for multiple requests, identified in Wireshark by the same source port across requests
- The 200/301/304/404 status codes are the ones you will encounter most often in real captures
