# HTTP/2 Implementation Design Spec

**Date**: 2026-02-13
**Branch**: `degroff/http2`
**Author**: Daniel DeGroff + Claude
**Status**: Draft

---

## Table of Contents

1. [Goals and Constraints](#1-goals-and-constraints)
2. [Scope](#2-scope)
3. [Industry Research](#3-industry-research)
4. [Architectural Approaches](#4-architectural-approaches)
5. [Recommended Architecture (Approach A)](#5-recommended-architecture-approach-a)
6. [Protocol Detection and Connection Routing](#6-protocol-detection-and-connection-routing)
7. [HTTP/2 Framing Layer](#7-http2-framing-layer)
8. [HPACK Header Compression](#8-hpack-header-compression)
9. [Stream Multiplexing](#9-stream-multiplexing)
10. [Flow Control](#10-flow-control)
11. [Connection and Stream Lifecycle](#11-connection-and-stream-lifecycle)
12. [Configuration API Design](#12-configuration-api-design)
13. [Backward Compatibility](#13-backward-compatibility)
14. [Testing Strategy](#14-testing-strategy)
15. [Load Testing](#15-load-testing)
16. [HTTP/1.1 Spec Gap Analysis](#16-http11-spec-gap-analysis)
17. [HTTP/1.1 Upgrade Mechanism](#17-http11-upgrade-mechanism)
18. [WebSocket Support](#18-websocket-support)
19. [Open Decision Points](#19-open-decision-points)
20. [Implementation Phases](#20-implementation-phases)
21. [File Inventory](#21-file-inventory)
22. [Security Considerations](#22-security-considerations)
23. [References](#23-references)

---

## 1. Goals and Constraints

### Primary Goals

1. **Native HTTP/2 support** following RFC 9113 as closely as practical
2. **No external dependencies** — pure Java, maintaining the zero-dependency principle
3. **Performance is paramount** — HTTP/2 should be at least as fast as HTTP/1.1 for single-request workloads, and significantly faster for multiplexed workloads
4. **Simple, readable code** — the next developer in the codebase should be able to understand it. Clean separation of concerns between HTTP/1.1 and HTTP/2 code paths
5. **Code reuse** — maximize reuse of existing infrastructure (HTTPHandler, HTTPRequest, HTTPResponse, configuration, TLS, logging, instrumentation)
6. **Backward compatible** — existing HTTP/1.1 functionality must not regress. Prefer a non-breaking 1.x release if possible
7. **Comprehensive testing** — follow existing test patterns (real servers, no mocks, socket-level edge case tests, high invocation counts for race detection)
8. **Expand load testing** to cover HTTP/2 benchmarks alongside existing HTTP/1.1 benchmarks

### Constraints

- Java 21+ (virtual threads, modern TLS/ALPN APIs)
- Server-side only (no HTTP client in this scope)
- Apache 2.0 license
- Must work with existing Savant build system

---

## 2. Scope

### In Scope

- HTTP/2 binary framing layer (RFC 9113)
- HPACK header compression (RFC 7541)
- Stream multiplexing with virtual threads
- Connection-level and per-stream flow control
- SETTINGS, PING, GOAWAY, RST_STREAM frame handling
- h2 over TLS with ALPN negotiation
- h2c with prior knowledge (plaintext HTTP/2, detect connection preface)
- Protocol detection and routing (HTTP/1.1 vs HTTP/2)
- HTTP/2 specific configuration options
- HTTP/2 integration and socket-level tests
- HTTP/2 load tests
- HTTP/1.1 critical gap fixes (Date header, HEAD suppression)
- HTTP/1.1 Upgrade mechanism (shared foundation for h2c Upgrade and WebSocket)
- h2c via HTTP/1.1 Upgrade (101 Switching Protocols -> HTTP/2)
- WebSocket support (RFC 6455)

### Out of Scope

- **Server Push (PUSH_PROMISE)** — intentionally omitted. Chrome removed support in Sept 2022. Firefox followed. Servlet 6.0 deprecated `PushBuilder`. Industry consensus is that server push is a failed feature. No compatibility issues from omitting it.
- **Stream Priorities (original RFC 7540 scheme)** — explicitly deprecated by RFC 9113. PRIORITY frames will be accepted and ignored per spec recommendation.
- **RFC 9218 Extensible Priorities** — only Jetty implements this natively among Java servers. Can be added later if demand arises.
- **HTTP client** — server-side only for this effort.
- **HTTP/1.1 non-critical gaps** — Range requests, conditional requests, etc. are documented in Section 16 for future work.

### Decision: h2 vs h2c

All three modes will be supported:

- **h2 (TLS + ALPN)** — required for browser clients. This is the primary mode.
- **h2c prior knowledge** — valuable for testing, debugging, and internal service-to-service communication. Trivial to detect (24-byte connection preface).
- **h2c Upgrade** — supported via the HTTP/1.1 Upgrade mechanism (see [Section 17](#17-http11-upgrade-mechanism)). Shares infrastructure with WebSocket upgrade.

---

## 3. Industry Research

### Java HTTP/2 Server Comparison

All six major Java HTTP servers support the core HTTP/2 feature set. This table shows the conventional agreement:

| Feature | Jetty | Netty | Undertow | Tomcat | Vert.x | Grizzly | java-http (planned) |
|---|---|---|---|---|---|---|---|
| h2 (TLS+ALPN) | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| h2c (Prior Knowledge) | Yes | Yes | Yes | Yes | Yes | Partial | **Yes** |
| h2c (Upgrade) | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| WebSocket | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| HPACK | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| Stream Multiplexing | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| Flow Control | Yes | Yes | Yes | Yes | Yes | Yes | **Yes** |
| Server Push | Yes | Yes | Yes | Yes | Yes | Limited | **No** (intentional) |
| Original Priorities | Yes | Yes | Basic | Basic | Basic | Basic | **No** (deprecated) |
| RFC 9218 Priorities | Jetty 12 only | No | No | No | No | No | **No** |

### Minimum Viable HTTP/2 (Industry Consensus)

Based on RFC 9113 requirements and what all major servers agree on:

1. HPACK header compression (mandatory per spec)
2. Stream multiplexing (core of HTTP/2)
3. Flow control — connection-level and per-stream (mandatory per spec)
4. SETTINGS exchange and acknowledgment
5. PING handling (trivial, mandatory)
6. GOAWAY for graceful connection close
7. RST_STREAM for stream resets
8. Error handling with proper error codes

Safely skipped:
- Server Push (dead feature)
- Original priority scheme (deprecated by RFC 9113)
- Extensible Priorities (only Jetty implements)

### JDK ALPN Support

Java 9+ includes native ALPN support. Since java-http requires JDK 21+, no external ALPN libraries are needed.

- `SSLParameters.setApplicationProtocols(String[])` — set offered protocols
- `SSLSocket.getApplicationProtocol()` — get negotiated protocol after handshake
- Full documentation in `javax.net.ssl` package

### Virtual Thread Architecture for HTTP/2

The recommended pattern (used by Tomcat and Jetty):
- One reader thread per HTTP/2 connection reads frames and demultiplexes by stream ID
- Each new stream dispatches to its own virtual thread for application-level processing
- This preserves the "one thread per request" blocking model that application handlers expect
- The frame writer serializes outgoing frames from concurrent streams onto the single TCP connection

This maps naturally to java-http's existing architecture where each HTTPWorker is a virtual thread handling one connection.

---

## 4. Architectural Approaches

Three approaches were considered for integrating HTTP/2:

### Approach A: Connection-level demux + virtual thread per stream (Recommended)

`HTTPServerThread` detects the protocol after accept. HTTP/1.1 connections go to the existing `HTTPWorker`. HTTP/2 connections go to a new `HTTP2ConnectionHandler` that runs a frame read loop and spawns a virtual thread per stream. Each stream calls the existing `HTTPHandler`.

**Pros:**
- Clean separation — HTTP/1.1 code is untouched
- Virtual threads per stream preserves the "one thread per request" simplicity
- Handlers don't know or care which protocol is underneath
- Easy to test HTTP/2 in isolation
- Natural fit for the existing architecture

**Cons:**
- Need a frame writer that serializes concurrent stream responses onto one TCP connection (requires synchronization)
- New classes to build and maintain

### Approach B: Unified worker with protocol branching

Modify `HTTPWorker` to detect protocol and branch internally. Same class handles both HTTP/1.1 and HTTP/2.

**Pros:**
- Fewer new classes
- Shared error handling/timeout logic

**Cons:**
- `HTTPWorker` becomes significantly more complex
- Mixes two very different protocol models (text vs binary, single-request vs multiplexed)
- Harder to test and maintain
- Violates the project's "keep code simple and readable" principle

### Approach C: Protocol abstraction layer

Create a `ProtocolHandler` interface with `HTTP11ProtocolHandler` and `HTTP2ProtocolHandler` implementations.

**Pros:**
- Most architecturally pure
- Easy to add protocols later

**Cons:**
- Over-engineered for what's needed today
- Adds abstraction layers that don't pay for themselves
- HTTP/3 (QUIC-based) would need completely different transport anyway, so the abstraction wouldn't help

### Decision

**Approach A is recommended** and is the current plan. It aligns with how Tomcat and Jetty handle HTTP/2 internally, fits the existing architecture naturally, and keeps the codebase simple and readable.

---

## 5. Recommended Architecture (Approach A)

### High-Level Architecture

```
HTTPServer
  |
  +-> HTTPServerThread (one per listener, platform thread)
        |
        +-> Accept loop
        |     |
        |     +-> Detect protocol (ALPN or connection preface)
        |     |
        |     +-> If HTTP/1.1: spawn HTTPWorker (existing, virtual thread)
        |     |     |
        |     |     +-> Parse text preamble -> HTTPRequest
        |     |     +-> Call HTTPHandler
        |     |     +-> Write text response
        |     |     +-> Keep-Alive loop
        |     |
        |     +-> If HTTP/2: spawn HTTP2ConnectionHandler (virtual thread)
        |           |
        |           +-> Exchange SETTINGS frames
        |           +-> Frame read loop:
        |           |     +-> Read frame header (9 bytes)
        |           |     +-> Dispatch by frame type:
        |           |           HEADERS -> create HTTP2Stream, spawn virtual thread
        |           |           DATA -> route to stream's input buffer
        |           |           SETTINGS -> handle connection settings
        |           |           WINDOW_UPDATE -> update flow control
        |           |           PING -> respond with PONG
        |           |           GOAWAY -> initiate graceful shutdown
        |           |           RST_STREAM -> cancel stream
        |           |
        |           +-> Each HTTP2Stream (virtual thread):
        |                 +-> HPACK decode HEADERS -> HTTPRequest
        |                 +-> Call HTTPHandler (same handler as HTTP/1.1)
        |                 +-> HTTPResponse -> HPACK encode -> HEADERS + DATA frames
        |
        +-> HTTPServerCleanerThread (monitors connections, same as today)
```

### Key Design Principles

1. **HTTPHandler is protocol-agnostic** — the same handler serves both HTTP/1.1 and HTTP/2 requests. No changes required to application code.

2. **HTTPRequest and HTTPResponse are reused** — HTTP/2 HEADERS frames are decoded via HPACK into the same `HTTPRequest` object. Response headers are encoded via HPACK from the same `HTTPResponse` object.

3. **One virtual thread per stream** — maintains the blocking I/O model. Each stream's virtual thread can read from a stream-specific input buffer and write to a stream-specific output that serializes through the connection's frame writer.

4. **Connection-level frame I/O is single-threaded for reads, synchronized for writes** — the reader loop is single-threaded (one virtual thread reads all frames for the connection). The frame writer must handle concurrent writes from multiple stream threads, so it needs synchronization (a write lock or a frame queue).

5. **Clean package separation** — all HTTP/2 code lives in new packages, keeping it separate from HTTP/1.1 internals.

### New Package Structure

```
io.fusionauth.http.server.internal/
  HTTPServerThread.java          (modified — protocol detection)
  HTTPWorker.java                (unchanged)
  HTTPBuffers.java               (unchanged)

io.fusionauth.http.server.internal.http2/
  HTTP2ConnectionHandler.java    (connection lifecycle, frame read loop)
  HTTP2FrameReader.java          (reads 9-byte frame headers + payloads)
  HTTP2FrameWriter.java          (writes frames, thread-safe)
  HTTP2Stream.java               (per-stream state machine + virtual thread)
  HTTP2StreamState.java          (stream state enum: idle, open, half-closed, closed)
  HTTP2Settings.java             (SETTINGS frame values)
  HTTP2FlowControl.java          (window management)
  HTTP2Error.java                (error codes enum)

io.fusionauth.http.server.internal.hpack/
  HpackDecoder.java              (decodes header blocks to name/value pairs)
  HpackEncoder.java              (encodes name/value pairs to header blocks)
  HpackStaticTable.java          (61-entry static table from RFC 7541 Appendix A)
  HpackDynamicTable.java         (FIFO dynamic table)
  HpackHuffman.java              (Huffman encoder/decoder + tables)
  HpackInteger.java              (variable-length integer encoding)
```

### Reused Existing Components

| Component | How It's Reused |
|---|---|
| `HTTPServer` | Entry point unchanged. Configuration extended. |
| `HTTPServerThread` | Modified to add protocol detection after accept. |
| `HTTPHandler` | Called identically from both HTTP/1.1 and HTTP/2 paths. |
| `HTTPRequest` | Populated from HPACK-decoded HEADERS instead of text preamble. |
| `HTTPResponse` | Read by HTTP/2 frame writer to produce HEADERS + DATA frames. |
| `HTTPServerConfiguration` | Extended with HTTP/2 settings. |
| `Configurable<T>` | Extended with HTTP/2 builder methods. |
| `HTTPListenerConfiguration` | Used for ALPN configuration on TLS listeners. |
| `SecurityTools` | Extended for ALPN parameter setup. |
| `Instrumenter` | Extended with HTTP/2 events (streams opened/closed, flow control). |
| `CountingInstrumenter` | Extended for HTTP/2 metrics in tests. |
| `Logger` / `LoggerFactory` | Used throughout HTTP/2 code. |
| `HTTPValues` | Extended with HTTP/2 pseudo-headers (`:method`, `:path`, `:scheme`, `:authority`, `:status`). |
| `Throughput` | Adapted for connection-level throughput monitoring. |
| `HTTPServerCleanerThread` | Extended to monitor HTTP/2 connections. |

---

## 6. Protocol Detection and Connection Routing

### TLS Connections (h2 via ALPN)

Current TLS setup in `HTTPServerThread` (lines 77-82):

```java
SSLContext context = SecurityTools.serverContext(listener.getCertificateChain(), listener.getPrivateKey());
this.socket = context.getServerSocketFactory().createServerSocket();
```

This creates an `SSLServerSocket` which produces `SSLSocket` instances on accept. To add ALPN:

**Option 1: Configure ALPN on SSLServerSocket**
```java
SSLServerSocket sslServerSocket = (SSLServerSocket) socket;
SSLParameters params = sslServerSocket.getSSLParameters();
params.setApplicationProtocols(new String[]{"h2", "http/1.1"});
sslServerSocket.setSSLParameters(params);
```

Then after accept:
```java
SSLSocket sslClient = (SSLSocket) sslServerSocket.accept();
sslClient.startHandshake();  // may be implicit on first read
String protocol = sslClient.getApplicationProtocol();
```

**Option 2: Use SSLContext + SSLSocketFactory for more control**

If `SSLServerSocket`-level ALPN configuration doesn't propagate to accepted sockets (needs verification), we may need to accept a plain `ServerSocket` and wrap with `SSLSocketFactory.createSocket(socket, ...)`, configuring ALPN on each accepted socket individually.

This needs to be verified during implementation. The JDK 21 documentation suggests Option 1 should work.

### Plaintext Connections (h2c Prior Knowledge)

The HTTP/2 connection preface is exactly 24 bytes:

```
PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
```

In hex: `505249202a20485454502f322e300d0a0d0a534d0d0a0d0a`

Detection approach:
1. Read first bytes into the existing `PushbackInputStream`
2. Compare against the 24-byte preface
3. If match → HTTP/2 path
4. If no match → push bytes back → HTTP/1.1 path (existing `HTTPWorker`)

Note: The string `PRI` was specifically chosen by the HTTP/2 spec authors because it is not a valid HTTP/1.1 method, so there is no ambiguity.

### Routing Logic in HTTPServerThread

```
After accept(clientSocket):
  if (listener.isTLS()):
    protocol = sslSocket.getApplicationProtocol()
    if (protocol == "h2"):
      spawn HTTP2ConnectionHandler(clientSocket)
    else:
      spawn HTTPWorker(clientSocket)  // existing path
  else:
    read first bytes via PushbackInputStream
    if (bytes match HTTP/2 preface):
      spawn HTTP2ConnectionHandler(clientSocket)
    else:
      pushback bytes
      spawn HTTPWorker(clientSocket)  // existing path
```

---

## 7. HTTP/2 Framing Layer

### Frame Format (RFC 9113 Section 4)

Every HTTP/2 frame has a 9-byte header:

```
+-----------------------------------------------+
|                 Length (24 bits)               |
+---------------+-------------------------------+
|   Type (8)    |   Flags (8)                   |
+-+-------------+-------------------------------+
|R|         Stream Identifier (31 bits)         |
+-+---------------------------------------------+
|                  Payload (variable)            |
+-----------------------------------------------+
```

- **Length**: payload size in bytes (max 16,384 by default, configurable via SETTINGS_MAX_FRAME_SIZE up to 16,777,215)
- **Type**: frame type (DATA=0x0, HEADERS=0x1, PRIORITY=0x2, RST_STREAM=0x3, SETTINGS=0x4, PUSH_PROMISE=0x5, PING=0x6, GOAWAY=0x7, WINDOW_UPDATE=0x8, CONTINUATION=0x9)
- **Flags**: type-specific flags
- **R**: reserved bit (must be 0)
- **Stream Identifier**: 31-bit stream ID (0 for connection-level frames)

### Frame Types to Implement

| Type | ID | Purpose | Complexity |
|---|---|---|---|
| DATA | 0x0 | Request/response body | Low — route payload to stream's input/output buffer |
| HEADERS | 0x1 | Request/response headers | Moderate — HPACK decode/encode, handle CONTINUATION |
| PRIORITY | 0x2 | Stream priority | Trivial — accept and ignore (deprecated) |
| RST_STREAM | 0x3 | Cancel a stream | Low — signal stream cancellation |
| SETTINGS | 0x4 | Connection configuration | Low — exchange settings, acknowledge |
| PUSH_PROMISE | 0x5 | Server push | Skip — not implemented |
| PING | 0x6 | Connection liveness | Trivial — echo back with ACK flag |
| GOAWAY | 0x7 | Graceful shutdown | Low — send last-stream-ID, error code |
| WINDOW_UPDATE | 0x8 | Flow control | Moderate — update send/receive windows |
| CONTINUATION | 0x9 | Header block continuation | Low — concatenate with preceding HEADERS |

### HTTP2FrameReader

Responsible for reading frames from the socket InputStream. Design:

- Read 9-byte header
- Validate length against SETTINGS_MAX_FRAME_SIZE
- Read payload bytes
- Return a frame object (type, flags, stream ID, payload)
- Handle CONTINUATION frames by buffering header block fragments until END_HEADERS flag is set

### HTTP2FrameWriter

Responsible for writing frames to the socket OutputStream. Must be **thread-safe** since multiple stream threads write concurrently.

Design options:
1. **Synchronized write lock** — simplest. Each frame write acquires a lock. Writes are atomic at the frame level (HTTP/2 requires frames to not be interleaved mid-frame).
2. **Frame queue** — stream threads enqueue frames, a dedicated writer thread dequeues and writes. More complex but avoids lock contention under high concurrency.

Recommendation: Start with synchronized write lock (option 1). Optimize to a queue if benchmarks show contention. The HTTP/2 spec requires that HEADERS and CONTINUATION frames for the same stream are not interleaved with frames from other streams, so the lock must be held for the entire HEADERS+CONTINUATION sequence.

### Unknown Frame Types

Per RFC 9113 Section 4.1: "Implementations MUST ignore and discard frames of unknown types." The frame reader should skip unknown frame types by reading and discarding the payload.

---

## 8. HPACK Header Compression

### Overview

HPACK (RFC 7541) is mandatory for HTTP/2. It compresses HTTP headers using:
- A static table of 61 common header name/value pairs
- A dynamic table (FIFO) of recently used headers per connection
- Huffman encoding for string literals
- Variable-length integer encoding

### Implementation Approach

**Decision: Implement from scratch in pure Java.**

Rationale:
- HPACK is a well-specified encoding algorithm, not cryptography. The RFC has exhaustive test vectors (Appendix C). The Huffman tables are fixed constants.
- ~800-1,250 lines of code across 6 files — comparable in complexity to the existing `ChunkedInputStream`/`ChunkedOutputStream` or the `RequestPreambleState` parser.
- Tomcat, Jetty, Netty, and Undertow all implemented their own HPACK from scratch.
- A from-scratch implementation fits naturally into the existing coding patterns and buffer strategies.

**Fallback option: Extract from Tomcat.** Tomcat's HPACK is the cleanest extraction candidate — 4 files, ~1,250 lines, Apache 2.0, minimal internal dependencies (just `StringManager` for error messages). Uses `ByteBuffer` natively. If time pressure demands it, this is a viable shortcut.

**Test control: Use Tomcat's HPACK in test scope** as a reference implementation to compare encoding/decoding results against our implementation.

### Component Design

#### HpackStaticTable.java
- 61 entries from RFC 7541 Appendix A
- Lookup maps: (name, value) -> index, name -> index
- Used by both encoder and decoder
- ~100-150 lines

#### HpackDynamicTable.java
- FIFO table with configurable maximum size
- Each entry size = name.length + value.length + 32 bytes (per spec)
- Operations: add (front), evict (back), lookup by index, resize
- Indices are relative: dynamic index = static_table_size + 1 + position
- ~100-150 lines

#### HpackHuffman.java
- Encoding: lookup table mapping byte -> bit pattern (from RFC 7541 Appendix B)
- Decoding: state machine or tree traversal with pre-computed decode table
- Huffman is optional per header field (encoder decides per-field whether to use it)
- ~300-500 lines (mostly table data)

#### HpackInteger.java
- Variable-length integer encoding with N-bit prefix (RFC 7541 Section 5.1)
- Encode: if value < 2^N - 1, fits in prefix; otherwise multi-byte encoding
- Decode: read prefix, if all 1s read continuation bytes
- ~40-60 lines

#### HpackEncoder.java
- Encodes a list of (name, value) pairs into a binary header block
- Decides representation per header: indexed, literal with indexing, literal without indexing, literal never indexed
- Sensitive headers (Authorization, Cookie, Set-Cookie) should use "never indexed"
- ~150-200 lines

#### HpackDecoder.java
- Decodes a binary header block into (name, value) pairs
- Reads representation type from prefix bits, dispatches accordingly
- Must enforce limits: max dynamic table size, max header list size, max header count
- ~150-250 lines

### Security Considerations

1. **HPACK bomb attack** — enforce SETTINGS_MAX_HEADER_LIST_SIZE to cap total decoded header size
2. **Dynamic table manipulation** — enforce SETTINGS_HEADER_TABLE_SIZE, limit resize frequency
3. **Huffman decoding errors** — invalid sequences are COMPRESSION_ERROR (connection error)
4. **Integer overflow** — bounds-check variable-length integers during decoding
5. **Table index validation** — invalid indices are COMPRESSION_ERROR
6. **Never-indexed fields** — respect the never-indexed flag for sensitive headers

### Test Strategy for HPACK

1. **RFC 7541 Appendix C test vectors** — comprehensive encode/decode examples with and without Huffman
2. **hpack-test-case project** (github.com/http2jp/hpack-test-case) — JSON test suite from multiple implementers
3. **Tomcat HPACK as test control** — decode the same input with both implementations, assert identical output
4. **Fuzz testing** — generate random header lists, encode with our encoder, decode, verify round-trip
5. **Security tests** — oversized headers, invalid Huffman, invalid indices, table overflow

---

## 9. Stream Multiplexing

### Stream Lifecycle (RFC 9113 Section 5.1)

```
                          +--------+
                  send PP |        | recv PP
                 ,--------| idle   |--------.
                /          |        |         \
               v           +--------+          v
        +----------+          |           +----------+
        |          |          | send H /  |          |
 ,------| reserved |          | recv H    | reserved |------.
 |      | (local)  |          |           | (remote) |      |
 |      +----------+          v           +----------+      |
 |          |            +--------+            |            |
 |          |    recv ES |        | send ES    |            |
 |   send H |   ,--------| open   |--------.  | recv H     |
 |          |  /         |        |         \  |            |
 |          v v          +--------+          v v            |
 |      +----------+          |           +----------+      |
 |      |   half-  |          |           |   half-  |      |
 |      |  closed  |          | send R /  |  closed  |      |
 |      | (remote) |          | recv R    | (local)  |      |
 |      +----------+          |           +----------+      |
 |           |                |                |            |
 |           | send ES /      |      recv ES / |            |
 |           | send R /       v       send R / |            |
 |           | recv R    +--------+   recv R   |            |
 | send R /  `---------->|        |<-----------'  send R /  |
 | recv R                | closed |               recv R    |
 `---------------------->|        |<------------------------'
                         +--------+
```

(PP = PUSH_PROMISE, H = HEADERS, ES = END_STREAM, R = RST_STREAM)

Since we are not implementing PUSH_PROMISE, the relevant states are:
- **idle** — initial state
- **open** — after receiving HEADERS (client sends request)
- **half-closed (remote)** — after receiving END_STREAM (client done sending)
- **half-closed (local)** — after sending END_STREAM (server done responding)
- **closed** — both sides done, or RST_STREAM received/sent

### HTTP2Stream Design

Each stream is represented by an `HTTP2Stream` object:

```
HTTP2Stream:
  - streamId (int, odd numbers for client-initiated)
  - state (HTTP2StreamState enum)
  - request (HTTPRequest, populated from HEADERS)
  - response (HTTPResponse, written by handler)
  - inputBuffer (pipe/buffer for DATA frames -> handler reads)
  - sendWindow (flow control: how many bytes we can send)
  - receiveWindow (flow control: how many bytes client can send)
  - thread (the virtual thread running the handler)
```

### Stream Creation Flow

1. Reader loop receives HEADERS frame with new stream ID
2. Validate stream ID (must be odd, must be greater than last seen)
3. Create `HTTP2Stream` object in `open` state
4. HPACK-decode the header block into an `HTTPRequest`
5. Map HTTP/2 pseudo-headers to HTTPRequest fields:
   - `:method` -> `HTTPRequest.method`
   - `:path` -> `HTTPRequest.path` (including query string)
   - `:scheme` -> `HTTPRequest.scheme`
   - `:authority` -> `HTTPRequest.host` + `HTTPRequest.port`
6. If END_STREAM flag set (no body) -> state = `half-closed (remote)`
7. Spawn virtual thread: call `HTTPHandler.handle(request, response)`
8. Stream thread reads body via stream's input buffer (populated by DATA frames)
9. When handler completes, encode response as HEADERS + DATA frames
10. Send END_STREAM on last DATA frame -> state = `half-closed (local)` then `closed`

### Concurrency Model

- **Reader thread** (one per connection): reads frames, demultiplexes to streams, handles connection-level frames (SETTINGS, PING, GOAWAY, WINDOW_UPDATE on stream 0)
- **Stream threads** (one per active stream): run the HTTPHandler, write response frames via the shared frame writer
- **Max concurrent streams**: configurable, default 100 (per RFC 9113 recommendation). Enforced by rejecting new HEADERS when limit is reached (RST_STREAM with REFUSED_STREAM).

---

## 10. Flow Control

### Overview (RFC 9113 Section 5.2)

HTTP/2 flow control operates at two levels:
- **Connection-level**: total DATA bytes across all streams
- **Per-stream**: DATA bytes for an individual stream

Flow control applies only to DATA frames. Other frame types are not flow-controlled.

### Window Management

- Each endpoint maintains a **send window** (how many bytes it can send) and a **receive window** (how many bytes the peer can send)
- Initial window size: 65,535 bytes (per RFC default), configurable via SETTINGS_INITIAL_WINDOW_SIZE
- When DATA is sent, the sender decreases its send window
- When DATA is received, the receiver decreases its receive window
- WINDOW_UPDATE frames increase the peer's send window
- If send window reaches 0, the sender must wait for WINDOW_UPDATE

### Implementation Strategy

```
HTTP2FlowControl:
  - connectionSendWindow (shared across all streams)
  - connectionReceiveWindow (shared across all streams)

  Per-stream:
  - stream.sendWindow
  - stream.receiveWindow

  Sending DATA:
    1. Wait until both connectionSendWindow > 0 and stream.sendWindow > 0
    2. Determine max send size = min(connectionSendWindow, stream.sendWindow, maxFrameSize)
    3. Write DATA frame of that size
    4. Decrement both windows

  Receiving DATA:
    1. Decrement connectionReceiveWindow and stream.receiveWindow
    2. If either window gets low, send WINDOW_UPDATE to replenish
    3. If sender exceeds window, this is a FLOW_CONTROL_ERROR (connection error)

  Auto WINDOW_UPDATE strategy:
    Send WINDOW_UPDATE when receive window drops below 50% of initial size.
    This balances responsiveness with frame overhead.
```

### Flow Control Considerations

- The writer must handle backpressure gracefully — when the send window is exhausted, stream threads must block (which is fine with virtual threads)
- SETTINGS_INITIAL_WINDOW_SIZE changes affect all existing streams (delta applied to all open streams)
- Connection-level WINDOW_UPDATE is on stream 0
- Per-stream WINDOW_UPDATE is on the specific stream ID

---

## 11. Connection and Stream Lifecycle

### Connection Establishment

**HTTP/2 over TLS (h2):**
1. TCP connect
2. TLS handshake with ALPN (`h2` negotiated)
3. Both endpoints send connection preface:
   - Client: magic bytes + SETTINGS frame
   - Server: SETTINGS frame
4. Both acknowledge peer's SETTINGS (SETTINGS frame with ACK flag)
5. Connection is ready for streams

**HTTP/2 over plaintext (h2c prior knowledge):**
1. TCP connect
2. Client sends 24-byte magic preface + SETTINGS frame
3. Server detects magic, sends own SETTINGS frame
4. Both acknowledge peer's SETTINGS
5. Connection is ready for streams

### Connection Shutdown

**Graceful shutdown (GOAWAY):**
1. Server sends GOAWAY frame with last-stream-ID = highest processed stream ID
2. Server stops accepting new streams
3. In-flight streams complete normally
4. After all streams complete, close TCP connection

**Abrupt shutdown:**
1. Server sends GOAWAY with error code (if applicable)
2. Close TCP connection immediately

**Integration with HTTPServer.close():**
- Call GOAWAY on all HTTP/2 connections
- Wait for in-flight streams to complete (up to `shutdownDuration`)
- Force close remaining connections

### Error Handling

**Connection errors** (affect the entire connection):
- Invalid frame format
- HPACK decompression failure (COMPRESSION_ERROR)
- Flow control violation (FLOW_CONTROL_ERROR)
- SETTINGS_MAX_FRAME_SIZE exceeded
- Action: send GOAWAY with error code, close connection

**Stream errors** (affect one stream):
- Invalid stream state transition
- Stream refused (too many concurrent streams)
- Cancel (client or server)
- Action: send RST_STREAM with error code

### Error Codes (RFC 9113 Section 7)

| Code | Name | Meaning |
|---|---|---|
| 0x0 | NO_ERROR | Graceful close |
| 0x1 | PROTOCOL_ERROR | Generic protocol violation |
| 0x2 | INTERNAL_ERROR | Unexpected internal error |
| 0x3 | FLOW_CONTROL_ERROR | Flow control limits exceeded |
| 0x4 | SETTINGS_TIMEOUT | SETTINGS ACK not received timely |
| 0x5 | STREAM_CLOSED | Frame received on closed stream |
| 0x6 | FRAME_SIZE_ERROR | Invalid frame size |
| 0x7 | REFUSED_STREAM | Stream refused (e.g., too many) |
| 0x8 | CANCEL | Stream cancelled |
| 0x9 | COMPRESSION_ERROR | HPACK error |
| 0xa | CONNECT_ERROR | CONNECT tunnel error |
| 0xb | ENHANCE_YOUR_CALM | Excessive load |
| 0xc | INADEQUATE_SECURITY | TLS requirements not met |
| 0xd | HTTP_1_1_REQUIRED | Use HTTP/1.1 for this request |

---

## 12. Configuration API Design

HTTP/2 introduces several server-side settings that should be configurable. Three options for the API:

### Option 1: Flat methods on Configurable (consistent with existing API)

```java
server.withHttp2MaxConcurrentStreams(100)
      .withHttp2InitialWindowSize(65535)
      .withHttp2MaxFrameSize(16384)
      .withHttp2MaxHeaderListSize(8192)
      .withHttp2HeaderTableSize(4096)
```

**Pros:** Consistent with existing `withKeepAliveTimeoutDuration()`, `withMaxRequestHeaderSize()`, etc. One builder chain. Discoverable via IDE autocomplete on `withHttp2`.

**Cons:** `Configurable<T>` interface grows. HTTP/2 settings mixed with HTTP/1.1 settings.

### Option 2: Nested configuration object

```java
server.withHttp2Configuration(new HTTP2Configuration()
          .withMaxConcurrentStreams(100)
          .withInitialWindowSize(65535)
          .withMaxFrameSize(16384)
          .withMaxHeaderListSize(8192)
          .withHeaderTableSize(4096))
```

**Pros:** Clean namespace separation. HTTP/2 settings grouped together. `Configurable<T>` doesn't grow. HTTP2Configuration can have its own defaults and validation.

**Cons:** Slight departure from existing flat builder pattern. Two builder chains.

### Option 3: Hybrid — nested object with shortcut

```java
// Full configuration
server.withHttp2Configuration(new HTTP2Configuration()
          .withMaxConcurrentStreams(100))

// Or shortcut for common cases
server.withHttp2MaxConcurrentStreams(100)
```

**Pros:** Flexibility. Shortcuts for common settings. Grouped object for advanced configuration.

**Cons:** Two ways to do the same thing. More API surface.

### Recommendation

**Option 2 (nested configuration object)** is recommended. It provides clean separation, keeps `Configurable<T>` from growing unbounded, and matches the existing `MultipartConfiguration` pattern already in the codebase. The `withMultipartConfiguration(MultipartConfiguration)` precedent makes this feel natural.

### HTTP/2 Settings with Defaults

| Setting | Default | RFC Reference | Description |
|---|---|---|---|
| `maxConcurrentStreams` | 100 | SETTINGS_MAX_CONCURRENT_STREAMS | Max simultaneous streams per connection |
| `initialWindowSize` | 65,535 | SETTINGS_INITIAL_WINDOW_SIZE | Initial flow control window size (bytes) |
| `maxFrameSize` | 16,384 | SETTINGS_MAX_FRAME_SIZE | Max size of a single frame payload (bytes) |
| `maxHeaderListSize` | 8,192 | SETTINGS_MAX_HEADER_LIST_SIZE | Max size of decoded header list (bytes) |
| `headerTableSize` | 4,096 | SETTINGS_HEADER_TABLE_SIZE | HPACK dynamic table size (bytes) |
| `connectionWindowSize` | 65,535 | (custom) | Initial connection-level flow control window |
| `enabled` | true | (custom) | Enable/disable HTTP/2 (can force HTTP/1.1 only) |

---

## 13. Backward Compatibility

### Goal: Non-breaking change (1.x release)

The following must remain unchanged:

- **HTTPHandler interface** — `void handle(HTTPRequest request, HTTPResponse response)` — no changes
- **HTTPRequest API** — all existing methods continue to work identically
- **HTTPResponse API** — all existing methods continue to work identically
- **HTTPServer construction** — existing code without HTTP/2 configuration works exactly as before
- **HTTP/1.1 behavior** — completely unchanged code path when clients use HTTP/1.1
- **Default behavior** — server auto-detects protocol. No configuration required to enable HTTP/2.

### What changes:

- **New classes** — HTTP/2 internal classes (non-public, not part of the API)
- **New configuration** — `HTTP2Configuration` class, `withHttp2Configuration()` method on `Configurable<T>`
- **New module exports** — none needed (HTTP/2 internals are in the existing `internal` package, which is not exported)
- **Modified `HTTPServerThread`** — internal change to add protocol detection
- **Extended `Instrumenter`** — new default methods for HTTP/2 events (backward compatible via default methods)
- **Extended `HTTPValues`** — new constants for HTTP/2 pseudo-headers

### Versioning Decision

If all changes are additive (new classes, new default methods, new configuration with defaults), this can ship as **1.5.0** (minor version bump).

If any breaking change is discovered during implementation (unlikely but possible — e.g., if `HTTPRequest` needs an incompatible change for pseudo-headers), it would require **2.0.0**.

This decision will be finalized during implementation.

---

## 14. Testing Strategy

### Test Organization

```
src/test/java/io/fusionauth/http/
  http2/
    HTTP2CoreTest.java           # Core HTTP/2 tests (schemes, keep-alive analog)
    HTTP2SocketTest.java         # Frame-level socket tests
    HTTP2StreamTest.java         # Stream lifecycle, multiplexing
    HTTP2FlowControlTest.java   # Flow control edge cases
    HTTP2SettingsTest.java       # SETTINGS exchange
    HTTP2ErrorTest.java          # Error handling, GOAWAY, RST_STREAM
  hpack/
    HpackIntegerTest.java        # Variable-length integer encoding
    HpackHuffmanTest.java        # Huffman encode/decode
    HpackStaticTableTest.java    # Static table lookups
    HpackDynamicTableTest.java   # Dynamic table operations
    HpackEncoderTest.java        # Full header encoding
    HpackDecoderTest.java        # Full header decoding
    HpackRoundTripTest.java      # Encode -> decode round-trip
    HpackRFCVectorTest.java      # RFC 7541 Appendix C test vectors
```

### Test Patterns

Following existing conventions:

1. **Real server instances** — every HTTP/2 test spins up a real `HTTPServer` with `makeServer()` pattern
2. **JDK HttpClient** as primary client — supports HTTP/2 natively, configure with `HttpClient.Version.HTTP_2`
3. **Raw socket tests** for frame-level edge cases:
   - Extend `BaseSocketTest` pattern for binary frame construction
   - Write raw HTTP/2 frames, validate raw frame responses
   - Test malformed frames, invalid stream IDs, flow control violations
4. **DataProviders** — test across `http` and `https` schemes (extend existing `schemes()` provider)
5. **invocationCount** — high repeat counts on protocol edge case tests to catch races
6. **Instrumentation** — use `CountingInstrumenter` extended for HTTP/2 metrics
7. **No mocks** — all tests use real servers, real sockets, real TLS

### HPACK-Specific Tests

1. **RFC 7541 Appendix C test vectors** — encode/decode examples from the spec
2. **Tomcat HPACK as control** — include Tomcat's HPACK classes in test scope, decode same input with both implementations, assert identical results
3. **Round-trip tests** — random header lists, encode with our encoder, decode with our decoder, verify identity
4. **Edge cases** — max table size, table eviction, never-indexed fields, Huffman padding errors, integer overflow
5. **Security tests** — oversized header bombs, invalid Huffman sequences, invalid table indices

### HTTP/2 Specific Test Scenarios

| Category | Test Cases |
|---|---|
| **Connection setup** | Preface exchange, SETTINGS, SETTINGS ACK, invalid preface |
| **Basic requests** | GET, POST, PUT, DELETE over HTTP/2, verify handler receives correct HTTPRequest |
| **Headers** | Pseudo-headers, regular headers, cookies, large headers, header-only requests |
| **Request body** | DATA frames, large bodies, chunked-equivalent (DATA frames), compressed bodies |
| **Response** | Status codes, response headers, response body, large responses |
| **Multiplexing** | Concurrent requests on same connection, verify all complete correctly |
| **Flow control** | Window exhaustion, WINDOW_UPDATE, connection vs stream windows |
| **Stream states** | Idle -> open -> half-closed -> closed transitions |
| **Errors** | Invalid stream ID, stream refused, GOAWAY, RST_STREAM |
| **Protocol detection** | ALPN negotiation, h2c preface detection, fallback to HTTP/1.1 |
| **Mixed traffic** | Same server handling HTTP/1.1 and HTTP/2 clients simultaneously |
| **TLS** | Certificate chain, ALPN with multiple protocols |
| **Timeouts** | Idle connection timeout, processing timeout |
| **Graceful shutdown** | GOAWAY, in-flight streams complete, new streams refused |
| **Backward compat** | Existing HTTP/1.1 tests still pass unchanged |

---

## 15. Load Testing

### Tool: h2load (nghttp2)

`h2load` is the standard HTTP/2 benchmarking tool, part of the nghttp2 project. It supports:
- HTTP/2 over TLS (h2) and plaintext (h2c)
- Configurable concurrent streams, connections, and requests
- Latency statistics
- Throughput measurement

### Load Test Plan

Extend the existing `load-tests/` directory:

```
load-tests/
  self/
    src/main/java/io/fusionauth/http/load/
      Main.java          # Modified to log protocol version
      LoadHandler.java   # Unchanged (protocol-agnostic)
  h2load/
    run-h2-benchmark.sh  # h2load benchmark script
    run-h1-benchmark.sh  # HTTP/1.1 baseline (existing ab or h2load --h1)
```

### Benchmark Scenarios

| Scenario | Tool | Configuration | Purpose |
|---|---|---|---|
| HTTP/2 baseline | h2load | 100 clients, 100K req, 1 stream/conn | Compare to HTTP/1.1 baseline |
| HTTP/2 multiplexing | h2load | 10 clients, 100 streams/conn, 100K req | Measure multiplexing benefit |
| HTTP/2 large response | h2load | 10 clients, 1MB responses | Verify flow control under load |
| HTTP/2 vs HTTP/1.1 | h2load | Same parameters, --h1 flag | Direct comparison |
| HTTP/2 vs Tomcat | h2load | Same parameters | Competitive benchmark |
| HTTP/2 TLS | h2load | Same as baseline, over TLS | TLS + HTTP/2 overhead |

### Performance Targets

- HTTP/2 single-stream throughput should be within 5% of HTTP/1.1 for the same workload
- HTTP/2 multiplexed throughput should exceed HTTP/1.1 keep-alive for concurrent requests
- Zero failed requests under sustained load
- Latency should not regress compared to HTTP/1.1

---

## 16. HTTP/1.1 Spec Gap Analysis

A comprehensive gap analysis was performed comparing java-http's HTTP/1.1 implementation against RFC 9110/9111/9112 and major Java HTTP servers (Tomcat, Jetty, Netty, Undertow).

### Critical Gaps (address in this release)

#### 16.1 Date Header Auto-Generation

**RFC 9110 Section 6.6.1**: "An origin server MUST send a Date header field in all cases" (with narrow exceptions for 1xx and 5xx).

**Current state**: `DateTools` class exists, `HTTPResponse.setDateHeader()` exists, but the server does not auto-generate Date if the handler doesn't set it.

**All competitors**: Auto-generate Date on every response.

**Fix**: In `HTTPOutputStream.commit()`, check if `response.containsHeader("Date")` and add one with `DateTools.format(ZonedDateTime.now())` if missing.

**Effort**: ~15 minutes.

#### 16.2 HEAD Method Body Suppression

**RFC 9110 Section 9.3.2**: HEAD response "MUST NOT contain a message body" but MUST include the same headers as a GET response (including Content-Length).

**Current state**: `HTTPMethod.HEAD` exists and parses correctly, but the server does not suppress body output. Handlers must manually check the method and skip writing.

**All competitors**: Automatically discard body writes for HEAD responses while preserving headers.

**Fix**: In `HTTPOutputStream`, when the request method is HEAD, wrap the delegate OutputStream in a discarding stream that counts bytes (for Content-Length) but does not write them to the socket.

**Effort**: ~1-2 hours.

### Important Gaps (document for future work)

#### 16.3 Range Requests (206 Partial Content)

**Current state**: No support. No Range/Accept-Ranges/Content-Range headers defined.

**All competitors**: Support range requests for static file serving.

**Practice**: Essential for media streaming, resumable downloads, PDF viewers.

**Recommendation**: Important but complex. Defer to a separate effort after HTTP/2.

#### 16.4 Conditional Requests (ETags, 304 Not Modified)

**Current state**: `If-Modified-Since` and `Last-Modified` header constants exist. No ETag support. No automatic 304 response generation.

**All competitors**: Full conditional request support.

**Practice**: Fundamental to HTTP caching. Used by every browser.

**Recommendation**: Important but can be implemented after HTTP/2. Mostly application-level.

#### 16.5 WebSocket Upgrade (RFC 6455)

**Current state**: No Upgrade header handling, no 101 Switching Protocols, no WebSocket framing.

**All competitors**: Full WebSocket support.

**Practice**: Widely used for real-time applications.

**Resolution**: Now in scope. See [Section 17](#17-http11-upgrade-mechanism) (Upgrade mechanism) and [Section 18](#18-websocket-support) (WebSocket).

#### 16.6 OPTIONS * Server-Level Handling

**Current state**: `OPTIONS *` parses successfully (path = `*`) but no server-level response (no auto `Allow` header).

**Recommendation**: Minor. Application handlers can respond to OPTIONS.

### Nice-to-Have Gaps

| Gap | Status | Recommendation |
|---|---|---|
| 408 Request Timeout response | Server closes socket on timeout without sending 408 | Small fix, improves client error handling |
| Server header auto-generation | No Server header sent | Configurable, default to `java-http/<version>` |
| Trailers (chunked) | Parsed/skipped but not exposed | Complete chunked encoding support |
| Content negotiation (Accept parser) | Accept-Encoding done, Accept not parsed | Convenience utility |
| 103 Early Hints | Only 100 Continue supported | Growing adoption but still niche |
| URI absolute-form handling | Parses but doesn't normalize | Proxy compatibility |
| Forwarded header (RFC 7239) | Only X-Forwarded-* supported | Standardized but less common than X-Forwarded |

### Gaps Intentionally Skipped

| Gap | Reason |
|---|---|
| CONNECT method | Proxy-only feature |
| Transfer-Encoding beyond chunked | Essentially unused in practice |
| HTTP/1.1 pipelining | Deprecated, HTTP/2 is the replacement |
| Via header | Proxy concern |
| TE header | Extremely rare |

---

## 17. HTTP/1.1 Upgrade Mechanism

### Overview

The HTTP/1.1 Upgrade mechanism (RFC 9110 Section 7.8) allows a client to request a protocol switch on an existing TCP connection. This is the shared foundation for both **h2c Upgrade** and **WebSocket**.

### How It Works

```
Client sends:
  GET / HTTP/1.1
  Host: example.com
  Connection: Upgrade
  Upgrade: <protocol>
  [protocol-specific headers]

Server responds (if accepting):
  HTTP/1.1 101 Switching Protocols
  Connection: Upgrade
  Upgrade: <protocol>

  [connection now uses new protocol]
```

After the 101 response, the TCP socket is handed off to the new protocol handler. The HTTP/1.1 connection is over.

### Implementation in HTTPWorker

The Upgrade mechanism is implemented in `HTTPWorker` during request processing:

1. After parsing the request preamble, check for `Connection: Upgrade` and `Upgrade` headers
2. Determine the requested protocol from the `Upgrade` header value
3. Dispatch to the appropriate upgrade handler:
   - `h2c` → HTTP/2 upgrade handler
   - `websocket` → WebSocket upgrade handler
4. If accepted: write 101 response, hand off the socket
5. If rejected: continue with normal HTTP/1.1 processing (or send 400/426)

### Design: UpgradeHandler Interface

```java
@FunctionalInterface
public interface UpgradeHandler {
  /**
   * Handle a protocol upgrade. Called after the 101 response is written.
   * The handler takes ownership of the socket and is responsible for
   * closing it when done.
   *
   * @param request  The original HTTP/1.1 request that initiated the upgrade
   * @param socket   The raw socket (already past the 101 response)
   * @param in       The socket InputStream (may have buffered bytes from preamble parsing)
   * @param out      The socket OutputStream
   */
  void handle(HTTPRequest request, Socket socket, InputStream in, OutputStream out) throws Exception;
}
```

This interface allows the Upgrade mechanism to be protocol-agnostic. The `HTTPWorker` doesn't need to know about HTTP/2 or WebSocket internals — it just detects the Upgrade, calls the appropriate handler, and exits.

### h2c Upgrade Flow

When a client sends `Upgrade: h2c`:

```
Client sends:
  GET / HTTP/1.1
  Host: example.com
  Connection: Upgrade, HTTP2-Settings
  Upgrade: h2c
  HTTP2-Settings: <base64url-encoded SETTINGS payload>

Server responds:
  HTTP/1.1 101 Switching Protocols
  Connection: Upgrade
  Upgrade: h2c

  [server sends HTTP/2 SETTINGS frame]
  [server processes the original request as HTTP/2 stream 1]
  [connection is now full HTTP/2]
```

Key details:
- The `HTTP2-Settings` header contains the client's initial SETTINGS, base64url-encoded
- The original request becomes stream 1 in the new HTTP/2 connection
- The server must respond to the original request over HTTP/2 after the switch

### Integration with HTTP2ConnectionHandler

After the 101 is written, the socket is passed to `HTTP2ConnectionHandler` with:
- The decoded `HTTP2-Settings` from the request as the client's initial SETTINGS
- The original request pre-loaded as stream 1
- The connection preface exchange abbreviated (client already sent its settings via the header)

### New Files for Upgrade Mechanism

| File | Package | Purpose |
|---|---|---|
| `UpgradeHandler.java` | `server` | Public interface for protocol upgrade handlers |

### Modified Files

| File | Changes |
|---|---|
| `HTTPWorker.java` | Detect `Connection: Upgrade`, dispatch to registered UpgradeHandler |
| `HTTPServerConfiguration.java` | Add `Map<String, UpgradeHandler>` for registering upgrade handlers |
| `Configurable.java` | Add `withUpgradeHandler(String protocol, UpgradeHandler handler)` |
| `HTTPValues.java` | Add `Upgrade`, `HTTP2-Settings` header constants, status 101 |

### Effort Estimate

- Upgrade mechanism in HTTPWorker: ~1-2 days
- h2c Upgrade handler: ~1 day (delegates to HTTP2ConnectionHandler)
- Total: ~2-3 days

---

## 18. WebSocket Support

### Overview

WebSocket (RFC 6455) provides full-duplex communication over a single TCP connection. It is initiated via the HTTP/1.1 Upgrade mechanism and then operates with its own binary framing protocol.

### Why Include WebSocket

**Value:**
- WebSocket is supported by all six major Java HTTP servers (Tomcat, Jetty, Netty, Undertow, Vert.x, Grizzly)
- Widely used for real-time applications: chat, notifications, live updates, collaborative editing, gaming, dashboards
- Even behind a reverse proxy, the origin server must handle WebSocket — proxies pass through the Upgrade but don't interpret WebSocket frames
- Completes the library's feature set for modern web applications

**Without WebSocket**, users who need real-time features must either:
- Use a different server (losing java-http's simplicity and performance benefits)
- Use long-polling or SSE (inferior alternatives)
- Run a separate WebSocket server alongside java-http

### WebSocket Handshake (RFC 6455 Section 4)

```
Client sends:
  GET /chat HTTP/1.1
  Host: example.com
  Connection: Upgrade
  Upgrade: websocket
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13
  [Sec-WebSocket-Protocol: chat, superchat]
  [Sec-WebSocket-Extensions: permessage-deflate]

Server responds:
  HTTP/1.1 101 Switching Protocols
  Connection: Upgrade
  Upgrade: websocket
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
  [Sec-WebSocket-Protocol: chat]
```

The `Sec-WebSocket-Accept` value is computed as:
```
Base64(SHA-1(Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-5AB0DC85B11B"))
```

This uses `java.security.MessageDigest` (SHA-1) and `java.util.Base64` — both in the JDK, no external deps.

### WebSocket Frame Format (RFC 6455 Section 5.2)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data (continued)                  |
+---------------------------------------------------------------+
```

### Frame Types (Opcodes)

| Opcode | Name | Purpose |
|---|---|---|
| 0x0 | Continuation | Continuation of a fragmented message |
| 0x1 | Text | UTF-8 text message |
| 0x2 | Binary | Binary message |
| 0x8 | Close | Connection close |
| 0x9 | Ping | Ping (keepalive) |
| 0xA | Pong | Pong (response to ping) |

### Key Protocol Details

- **Masking**: Client-to-server frames MUST be masked (XOR with 4-byte key). Server-to-client frames MUST NOT be masked.
- **Fragmentation**: Large messages can be split across multiple frames (first frame has opcode, continuations use 0x0, last frame has FIN bit).
- **Close handshake**: Either side sends Close frame with status code. The other side responds with Close frame, then TCP closes.
- **Ping/Pong**: Either side can send Ping; receiver must respond with Pong containing the same payload. Used for keepalive.
- **Max payload**: 7-bit length (0-125 bytes), 16-bit extended (126 = up to 65,535 bytes), or 64-bit extended (127 = up to 2^63 bytes).

### Public API Design

WebSocket needs a new handler interface since the interaction model is fundamentally different from request/response:

```java
/**
 * Handler for WebSocket connections. Implement this to handle WebSocket messages.
 */
public interface WebSocketHandler {
  /**
   * Called when a new WebSocket connection is established.
   * Use the session to send messages back to the client.
   */
  void onOpen(WebSocketSession session);

  /**
   * Called when a text message is received from the client.
   */
  void onMessage(WebSocketSession session, String message);

  /**
   * Called when a binary message is received from the client.
   */
  void onMessage(WebSocketSession session, byte[] message);

  /**
   * Called when the WebSocket connection is closed.
   *
   * @param statusCode The close status code (1000 = normal, etc.)
   * @param reason     The close reason string (may be empty)
   */
  void onClose(WebSocketSession session, int statusCode, String reason);

  /**
   * Called when an error occurs on the WebSocket connection.
   */
  default void onError(WebSocketSession session, Throwable error) {}
}
```

```java
/**
 * Represents an active WebSocket connection. Thread-safe for sending.
 */
public interface WebSocketSession {
  /** Send a text message to the client. */
  void send(String message) throws IOException;

  /** Send a binary message to the client. */
  void send(byte[] message) throws IOException;

  /** Send a ping frame. */
  void ping(byte[] payload) throws IOException;

  /** Close the WebSocket connection with a status code and reason. */
  void close(int statusCode, String reason) throws IOException;

  /** Close with normal closure (1000). */
  void close() throws IOException;

  /** Get the original HTTP request that initiated the WebSocket upgrade. */
  HTTPRequest getUpgradeRequest();

  /** Check if the connection is still open. */
  boolean isOpen();
}
```

### Configuration

WebSocket configuration would follow the nested object pattern:

```java
server.withWebSocketHandler("/chat", new ChatHandler())
      .withWebSocketConfiguration(new WebSocketConfiguration()
          .withMaxMessageSize(64 * 1024)        // 64KB default
          .withMaxFrameSize(16 * 1024)           // 16KB default
          .withIdleTimeout(Duration.ofMinutes(5)) // 5 min default
          .withSubProtocols("chat", "superchat")) // Sec-WebSocket-Protocol
```

WebSocket handlers are registered by path. When an Upgrade request arrives for `/chat`, the server matches the path to the registered handler.

### Architecture

```
HTTPWorker receives request with Upgrade: websocket
  |
  +-> Validate handshake (Sec-WebSocket-Key, Version 13)
  +-> Match path to registered WebSocketHandler
  +-> Compute Sec-WebSocket-Accept
  +-> Write 101 Switching Protocols response
  +-> Hand off socket to WebSocketConnectionHandler
        |
        +-> Call handler.onOpen(session)
        +-> Frame read loop (virtual thread):
        |     +-> Read frame header
        |     +-> Unmask payload
        |     +-> Handle by opcode:
        |           Text -> handler.onMessage(session, text)
        |           Binary -> handler.onMessage(session, bytes)
        |           Ping -> auto-respond with Pong
        |           Pong -> ignore (or notify handler)
        |           Close -> handler.onClose(session, code, reason)
        |           Continuation -> buffer until FIN
        |
        +-> WebSocketSession.send() writes frames to socket
```

### New Package Structure

```
io.fusionauth.http.server/
  WebSocketHandler.java           (public interface)
  WebSocketSession.java           (public interface)
  WebSocketConfiguration.java     (public configuration)

io.fusionauth.http.server.internal.ws/
  WebSocketConnectionHandler.java  (frame read loop, dispatches to handler)
  WebSocketFrameReader.java        (reads WebSocket frames, handles masking)
  WebSocketFrameWriter.java        (writes WebSocket frames)
  WebSocketSessionImpl.java        (WebSocketSession implementation)
  WebSocketCloseCode.java          (close status code constants)
```

### Comparison: WebSocket Complexity vs HTTP/2

| Aspect | HTTP/2 | WebSocket |
|---|---|---|
| Frame format | 9-byte header, 10 types | 2-14 byte header, 6 opcodes |
| Header compression | HPACK (~1,000 lines) | None |
| Multiplexing | Yes (streams) | No (single channel) |
| Flow control | Yes (windows) | No |
| State machine | Complex (stream states) | Simple (open/closing/closed) |
| **Overall complexity** | **High** | **Moderate** |

WebSocket is significantly simpler than HTTP/2. The frame format is straightforward, there's no header compression, no multiplexing, and no flow control. The main implementation effort is:
1. Frame reader with masking (~200 lines)
2. Frame writer (~150 lines)
3. Connection handler with message assembly (~300 lines)
4. Session implementation (~150 lines)
5. Handshake validation (~100 lines)

**Estimated total: ~900-1,200 lines of implementation code.**

### Pros and Cons

**Pros:**
- Completes the library for modern web applications
- All major competitors support it — this is an expected feature
- Moderate implementation effort (~1-2 weeks)
- Shares the Upgrade mechanism with h2c, reducing marginal cost
- Virtual threads are a natural fit (one thread per WebSocket connection)
- Zero additional dependencies (SHA-1 and Base64 are in the JDK)
- The simple frame protocol is well within the project's existing capabilities (comparable to ChunkedInputStream/ChunkedOutputStream)

**Cons:**
- Adds a new public API surface (WebSocketHandler, WebSocketSession, WebSocketConfiguration)
- Different interaction model than request/response — may need documentation/examples
- Ongoing maintenance burden for a second protocol
- Testing complexity — need WebSocket client tests (JDK's `HttpClient` does NOT include a WebSocket client, but `java.net.http` has a `WebSocket` builder since JDK 11)
- Extensions like `permessage-deflate` (compression) add complexity — could be deferred

### WebSocket over HTTP/2 (RFC 8441)

RFC 8441 defines WebSocket over HTTP/2 using the CONNECT method with `:protocol` pseudo-header. This allows WebSocket connections to be multiplexed over an HTTP/2 connection.

**Recommendation**: Defer. This is an advanced feature that only Undertow supports among Java servers. Standard WebSocket over HTTP/1.1 Upgrade covers all real-world use cases. Can be added later if needed.

### Testing Strategy for WebSocket

```
src/test/java/io/fusionauth/http/
  ws/
    WebSocketHandshakeTest.java     # Handshake validation, Sec-WebSocket-Accept
    WebSocketFrameTest.java         # Frame encoding/decoding, masking
    WebSocketConnectionTest.java    # Full connection lifecycle
    WebSocketMessageTest.java       # Text, binary, fragmented messages
    WebSocketCloseTest.java         # Close handshake, status codes
    WebSocketPingPongTest.java      # Keepalive
    WebSocketErrorTest.java         # Invalid frames, protocol violations
    WebSocketLoadTest.java          # Many concurrent connections
```

**Test client**: JDK 11+ `java.net.http.WebSocket` — a built-in WebSocket client suitable for testing.

### Effort Estimate

| Component | Effort |
|---|---|
| Upgrade mechanism (shared with h2c) | 1-2 days |
| WebSocket handshake | 0.5 days |
| WebSocket frame reader/writer | 1-2 days |
| WebSocket connection handler | 1-2 days |
| Public API (handler, session, config) | 1 day |
| Testing | 2-3 days |
| **Total** | **~7-10 days** |

---

## 19. Open Decision Points

### 19.1 h2c Support Scope

**Question**: Should we implement h2c (HTTP/2 over plaintext)?

**Options**:

| Option | Effort | Value |
|---|---|---|
| h2 only (TLS + ALPN) | Lowest | Covers all browser clients |
| h2 + h2c prior knowledge | Low (incremental) | Adds testing convenience + internal service support |
| h2 + h2c prior knowledge + h2c Upgrade | Moderate | Full spec compliance, shares Upgrade mechanism with WebSocket |

**Decision**: All three modes. h2c prior knowledge detection is trivial (24-byte check). h2c Upgrade shares infrastructure with WebSocket Upgrade (see [Section 17](#17-http11-upgrade-mechanism)), reducing marginal cost.

**All six major Java servers** support all three modes.

### 19.2 Version Number

**Question**: 1.x or 2.0?

**Decision criteria**: If the implementation requires breaking changes to the public API (HTTPHandler, HTTPRequest, HTTPResponse, Configurable), it's 2.0. If all changes are additive, it's 1.x.

**Current assessment**: Likely 1.x. The design intentionally preserves the existing API. To be confirmed during implementation.

### 19.3 HPACK Implementation

**Question**: Build from scratch or extract from Tomcat?

**Current leaning**: Build from scratch, using Tomcat as a test-scope reference implementation for comparison.

**Fallback**: Extract from Tomcat if time pressure warrants it.

### 19.4 Frame Writer Synchronization

**Question**: Synchronized lock vs frame queue for the connection-level frame writer?

**Recommendation**: Start with synchronized lock (simpler). Optimize to a queue only if benchmarks show contention. Profile before optimizing.

### 19.5 HTTP/2 Configuration Namespacing

**Question**: How to expose HTTP/2 settings in the API?

**Current leaning**: Option 2 (nested `HTTP2Configuration` object), following the `MultipartConfiguration` precedent.

---

## 20. Implementation Phases

All phases ship together in a single release. Phases are for development ordering.

### Phase 1: Foundation (~1 week)

1. **HPACK implementation** (6 files, ~1,000 lines)
   - HpackInteger, HpackStaticTable, HpackHuffman, HpackDynamicTable, HpackDecoder, HpackEncoder
   - Full test suite with RFC test vectors
   - Tomcat HPACK as test control

2. **HTTP/2 frame codec** (2 files)
   - HTTP2FrameReader — read 9-byte header + payload
   - HTTP2FrameWriter — write frames (synchronized)
   - Frame type constants, error codes

3. **HTTP/1.1 critical fixes**
   - Date header auto-generation
   - HEAD method body suppression

### Phase 2: Connection Handling (~1 week)

4. **Protocol detection** in HTTPServerThread
   - ALPN on TLS listeners
   - Connection preface detection on plaintext listeners

5. **HTTP2ConnectionHandler** — connection lifecycle
   - Connection preface exchange (SETTINGS)
   - SETTINGS frame handling + ACK
   - PING/PONG
   - GOAWAY (graceful and error)
   - Unknown frame type handling (ignore)

6. **HTTP2Settings** — settings negotiation
   - Parse/emit SETTINGS frames
   - Apply settings changes to connection state

### Phase 3: Stream Processing (~1.5 weeks)

7. **HTTP2Stream** — per-stream state machine
   - Stream state transitions
   - HEADERS -> HTTPRequest mapping (HPACK decode + pseudo-header mapping)
   - HTTPResponse -> HEADERS + DATA frame mapping (HPACK encode)
   - Virtual thread per stream

8. **Flow control** — HTTP2FlowControl
   - Connection-level and per-stream windows
   - WINDOW_UPDATE handling
   - Send-side blocking when window exhausted
   - Auto WINDOW_UPDATE strategy

9. **Stream input/output**
   - Stream input buffer (DATA frames -> InputStream for handler)
   - Stream output (handler OutputStream -> DATA frames via frame writer)
   - END_STREAM handling

### Phase 4: Configuration and Integration (~0.5 weeks)

10. **HTTP2Configuration** class
11. **Configurable<T>** extension — `withHttp2Configuration()`
12. **Instrumenter** extension — HTTP/2 events
13. **HTTPValues** extension — pseudo-header constants
14. **HTTPServerCleanerThread** — HTTP/2 connection monitoring

### Phase 5: Testing (~1.5 weeks)

15. **HPACK tests** (done in Phase 1, but expand)
16. **HTTP/2 integration tests** — real server + JDK HttpClient
17. **HTTP/2 socket tests** — raw frame-level edge cases
18. **Mixed protocol tests** — HTTP/1.1 and HTTP/2 on same server
19. **Error handling tests** — GOAWAY, RST_STREAM, invalid frames
20. **Backward compatibility verification** — all existing tests still pass
21. **TLS/ALPN tests**

### Phase 6: Load Testing and Optimization (~1 week)

22. **h2load benchmark scripts**
23. **Benchmark: HTTP/2 vs HTTP/1.1**
24. **Benchmark: java-http HTTP/2 vs Tomcat HTTP/2**
25. **Performance profiling and optimization**
26. **Frame writer optimization** (queue vs lock, based on profiling)
27. **Update load-tests/ directory**

### Phase 7: Upgrade Mechanism and WebSocket (~2 weeks)

28. **HTTP/1.1 Upgrade mechanism** (~2-3 days)
    - `UpgradeHandler` interface
    - Upgrade detection in `HTTPWorker`
    - Configuration for registering upgrade handlers
    - h2c Upgrade handler (delegates to `HTTP2ConnectionHandler`)

29. **WebSocket implementation** (~5-7 days)
    - `WebSocketHandler` and `WebSocketSession` public API
    - `WebSocketConfiguration`
    - WebSocket handshake validation (Sec-WebSocket-Key/Accept)
    - WebSocket frame reader (with unmasking)
    - WebSocket frame writer
    - `WebSocketConnectionHandler` (frame read loop, message assembly)
    - `WebSocketSessionImpl` (send, ping, close)
    - Close handshake, Ping/Pong
    - Message fragmentation (continuation frames)

30. **WebSocket and Upgrade testing** (~3-4 days)
    - Handshake validation tests
    - Frame encoding/decoding tests
    - Full connection lifecycle tests
    - Text, binary, and fragmented message tests
    - Close handshake and error handling tests
    - Concurrent connection load tests
    - h2c Upgrade integration tests

---

## 21. File Inventory

### New Files

| File | Package | Purpose |
|---|---|---|
| `HTTP2ConnectionHandler.java` | `server.internal.http2` | Connection lifecycle, frame read loop |
| `HTTP2FrameReader.java` | `server.internal.http2` | Read binary frames from InputStream |
| `HTTP2FrameWriter.java` | `server.internal.http2` | Write binary frames to OutputStream (thread-safe) |
| `HTTP2Stream.java` | `server.internal.http2` | Per-stream state machine |
| `HTTP2StreamState.java` | `server.internal.http2` | Stream state enum |
| `HTTP2Settings.java` | `server.internal.http2` | SETTINGS frame values and defaults |
| `HTTP2FlowControl.java` | `server.internal.http2` | Window management |
| `HTTP2Error.java` | `server.internal.http2` | Error codes enum |
| `HTTP2Configuration.java` | `server` | Public configuration class |
| `HpackDecoder.java` | `server.internal.hpack` | Header block decoder |
| `HpackEncoder.java` | `server.internal.hpack` | Header block encoder |
| `HpackStaticTable.java` | `server.internal.hpack` | Static table (61 entries) |
| `HpackDynamicTable.java` | `server.internal.hpack` | Dynamic table (FIFO) |
| `HpackHuffman.java` | `server.internal.hpack` | Huffman encode/decode + tables |
| `HpackInteger.java` | `server.internal.hpack` | Variable-length integer encoding |
| `UpgradeHandler.java` | `server` | Public interface for protocol upgrade handlers |
| `WebSocketHandler.java` | `server` | Public WebSocket handler interface |
| `WebSocketSession.java` | `server` | Public WebSocket session interface |
| `WebSocketConfiguration.java` | `server` | Public WebSocket configuration |
| `WebSocketConnectionHandler.java` | `server.internal.ws` | Frame read loop, dispatches to handler |
| `WebSocketFrameReader.java` | `server.internal.ws` | Reads WebSocket frames, handles masking |
| `WebSocketFrameWriter.java` | `server.internal.ws` | Writes WebSocket frames |
| `WebSocketSessionImpl.java` | `server.internal.ws` | WebSocketSession implementation |
| `WebSocketCloseCode.java` | `server.internal.ws` | Close status code constants |

### Modified Files

| File | Changes |
|---|---|
| `HTTPServerThread.java` | Protocol detection (ALPN + preface), routing to HTTP2ConnectionHandler |
| `HTTPServerConfiguration.java` | Add `HTTP2Configuration` field |
| `Configurable.java` | Add `withHttp2Configuration()` default method |
| `HTTPValues.java` | Add HTTP/2 pseudo-header constants |
| `Instrumenter.java` | Add HTTP/2 event methods (with defaults for backward compat) |
| `CountingInstrumenter.java` | Add HTTP/2 metric counters |
| `SecurityTools.java` | Add ALPN parameter setup helper |
| `HTTPOutputStream.java` | Date header auto-generation, HEAD body suppression |
| `HTTPWorker.java` | Detect `Connection: Upgrade`, dispatch to registered UpgradeHandler |
| `Configurable.java` | Also add `withUpgradeHandler()`, `withWebSocketHandler()`, `withWebSocketConfiguration()` |
| `module-info.java` | May need to export `server.internal.ws` if WebSocket public types reference internal types, otherwise no changes |

### New Test Files

| File | Purpose |
|---|---|
| `http2/HTTP2CoreTest.java` | Core HTTP/2 request/response tests |
| `http2/HTTP2SocketTest.java` | Frame-level protocol tests |
| `http2/HTTP2StreamTest.java` | Stream lifecycle, multiplexing |
| `http2/HTTP2FlowControlTest.java` | Flow control edge cases |
| `http2/HTTP2SettingsTest.java` | SETTINGS exchange |
| `http2/HTTP2ErrorTest.java` | Error handling |
| `hpack/HpackIntegerTest.java` | Integer encoding tests |
| `hpack/HpackHuffmanTest.java` | Huffman tests |
| `hpack/HpackStaticTableTest.java` | Static table tests |
| `hpack/HpackDynamicTableTest.java` | Dynamic table tests |
| `hpack/HpackEncoderTest.java` | Encoder tests |
| `hpack/HpackDecoderTest.java` | Decoder tests |
| `hpack/HpackRoundTripTest.java` | Encode/decode round-trip |
| `hpack/HpackRFCVectorTest.java` | RFC 7541 Appendix C vectors |
| `ws/WebSocketHandshakeTest.java` | Handshake validation, Sec-WebSocket-Accept |
| `ws/WebSocketFrameTest.java` | Frame encoding/decoding, masking |
| `ws/WebSocketConnectionTest.java` | Full connection lifecycle |
| `ws/WebSocketMessageTest.java` | Text, binary, fragmented messages |
| `ws/WebSocketCloseTest.java` | Close handshake, status codes |
| `ws/WebSocketPingPongTest.java` | Keepalive |
| `ws/WebSocketErrorTest.java` | Invalid frames, protocol violations |
| `ws/WebSocketLoadTest.java` | Many concurrent connections |
| `http2/HTTP2UpgradeTest.java` | h2c Upgrade via 101 Switching Protocols |

---

## 22. Security Considerations

### HPACK Security

- **HPACK bomb**: Enforce SETTINGS_MAX_HEADER_LIST_SIZE to cap decoded header size
- **Dynamic table manipulation**: Enforce table size limits, limit resize frequency
- **Huffman decoding**: Reject invalid sequences (COMPRESSION_ERROR)
- **Integer overflow**: Bounds-check variable-length integers
- **Sensitive headers**: Respect "never indexed" flag for Authorization, Cookie, etc.

### HTTP/2 Protocol Security

- **Slow read attacks (Slowloris)**: Existing throughput monitoring applies. HTTP/2 connections are longer-lived, so monitoring must account for idle streams vs dead connections.
- **Stream flood**: Enforce SETTINGS_MAX_CONCURRENT_STREAMS. RST_STREAM with REFUSED_STREAM for excess.
- **SETTINGS flood**: Rate-limit SETTINGS frames. GOAWAY with ENHANCE_YOUR_CALM if excessive.
- **PING flood**: Rate-limit PING frames.
- **Empty frame flood**: Rate-limit zero-length DATA frames or HEADERS with no useful content.
- **RST_STREAM flood**: Track rapid stream creation/reset patterns.
- **Connection window exhaustion**: Send WINDOW_UPDATE proactively to prevent deadlocks.

### WebSocket Security

- **Origin validation**: The server should expose the `Origin` header from the handshake request via `WebSocketSession` so applications can validate the origin and prevent Cross-Site WebSocket Hijacking (CSWSH).
- **Message size limits**: Enforce `WebSocketConfiguration.maxMessageSize` to prevent memory exhaustion from oversized messages or fragmentation abuse.
- **Frame masking**: Server MUST reject unmasked client frames (RFC 6455 Section 5.1). Masking prevents cache poisoning attacks on intermediaries.
- **UTF-8 validation**: Text frames MUST contain valid UTF-8. Invalid UTF-8 is a protocol error (close code 1007).
- **Close frame flood**: Rate-limit or ignore excessive close frames to prevent resource exhaustion.
- **Ping flood**: Rate-limit ping frames to prevent the server from being overwhelmed generating pong responses.
- **Fragmentation abuse**: Enforce limits on fragmented message assembly to prevent memory exhaustion from incomplete fragments.

### TLS Requirements

RFC 9113 Section 9.2 specifies TLS requirements for h2:
- TLS 1.2+ required (TLS 1.3 preferred)
- Certain cipher suites are prohibited (e.g., TLS_RSA_WITH_AES_128_CBC_SHA)
- SNI (Server Name Indication) should be supported

The current `SecurityTools.serverContext()` uses `SSLContext.getInstance("TLS")` which negotiates the highest mutually supported version. JDK 21 defaults to TLS 1.3 when available. No changes needed for basic compliance, but we should verify cipher suite filtering in implementation.

---

## 23. References

### RFCs

- **RFC 9113** — HTTP/2 (June 2022, replaces RFC 7540)
- **RFC 7541** — HPACK: Header Compression for HTTP/2
- **RFC 9110** — HTTP Semantics (includes Upgrade mechanism, Section 7.8)
- **RFC 9111** — HTTP Caching
- **RFC 9112** — HTTP/1.1
- **RFC 6455** — The WebSocket Protocol
- **RFC 8441** — Bootstrapping WebSockets with HTTP/2 (deferred, for future reference)

### Test Resources

- RFC 7541 Appendix C — HPACK test vectors
- github.com/http2jp/hpack-test-case — comprehensive HPACK test suite
- h2spec (github.com/summerwind/h2spec) — HTTP/2 conformance testing tool

### Tools

- h2load (nghttp2) — HTTP/2 benchmarking
- nghttp (nghttp2) — HTTP/2 client for debugging
- Wireshark — HTTP/2 frame inspection

### Existing Codebase Key Files

| File | Relevance |
|---|---|
| `HTTPServer.java` | Entry point, configuration, lifecycle |
| `HTTPServerThread.java` | Accept loop, will be modified for protocol detection |
| `HTTPWorker.java` | HTTP/1.1 request lifecycle, model for HTTP2Stream |
| `HTTPRequest.java` | Request API, reused for HTTP/2 |
| `HTTPResponse.java` | Response API, reused for HTTP/2 |
| `HTTPServerConfiguration.java` | Configuration, will be extended |
| `Configurable.java` | Builder pattern, will be extended |
| `HTTPOutputStream.java` | Response output, will be modified for Date/HEAD fixes |
| `SecurityTools.java` | TLS setup, will be extended for ALPN |
| `RequestPreambleState.java` | HTTP/1.1 parser (reference for coding style) |
| `ChunkedInputStream.java` | Stream handling (reference for coding patterns) |
| `BaseTest.java` | Test infrastructure (reference for test patterns) |
| `BaseSocketTest.java` | Socket-level tests (reference for HTTP/2 socket tests) |
