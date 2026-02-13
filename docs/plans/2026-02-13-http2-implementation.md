# HTTP/2, WebSocket, and Benchmark Framework Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add native HTTP/2, WebSocket, and a comprehensive benchmark framework to the java-http library.

**Architecture:** Connection-level protocol detection (ALPN or preface) routes to either the existing HTTPWorker (HTTP/1.1) or a new HTTP2ConnectionHandler. Each HTTP/2 stream runs on its own virtual thread, calling the same HTTPHandler. WebSocket uses the HTTP/1.1 Upgrade mechanism. See `docs/plans/2026-02-13-http2-design.md` for full design.

**Tech Stack:** Java 21 (virtual threads, ALPN), TestNG, Savant build system (`sb`), zero runtime dependencies.

---

## Build and Test Commands

All commands run from the worktree root: `/Users/robotdan/dev/fusionauth/java-http/.worktrees/degroff/http2`

```bash
# Compile and run all tests
JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home sb test

# Compile only (faster feedback during implementation)
JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home sb compile

# Run a single test class (TestNG via Savant doesn't support this directly,
# so use compile + targeted test via IDE or testng.xml override)
```

## Source Paths

- Main source: `src/main/java/io/fusionauth/http/`
- Test source: `src/test/java/io/fusionauth/http/`
- Design spec: `docs/plans/2026-02-13-http2-design.md`

## Conventions

- Follow existing code style: 2-space indentation, Apache 2.0 header on every file
- Copyright: `Copyright (c) 2022-2026, FusionAuth, All Rights Reserved`
- No compile-time dependencies (test deps: jackson5, restify, testng, slf4j-nop)
- Tests use real servers, no mocks. `BaseTest.makeServer()` pattern for server setup.
- High `invocationCount` on edge case tests to catch races (100-250 typical)
- `@author Daniel DeGroff` on new files

---

## Phase 1: Foundation (~1 week)

### Task 1: HPACK Variable-Length Integer Encoding

Implement RFC 7541 Section 5.1 variable-length integer encoding/decoding. This is the lowest-level HPACK primitive.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackInteger.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackIntegerTest.java`

**Step 1: Write HpackIntegerTest**

Test encode/decode round-trips and RFC 7541 Section C.1 examples:
- Encoding 10 with 5-bit prefix = `0x0a` (single byte)
- Encoding 1337 with 5-bit prefix = `0x1f 0x9a 0x0a` (multi-byte)
- Encoding 42 with 8-bit prefix = `0x2a` (single byte)
- Edge cases: 0, max int, prefix sizes 4-8

```java
@Test
public class HpackIntegerTest {
  @Test
  public void rfc_c1_example_10() {
    // RFC 7541 C.1.1: Encoding 10 using a 5-bit prefix
    byte[] encoded = HpackInteger.encode(10, 5);
    assertEquals(encoded, new byte[]{0x0a});
    assertEquals(HpackInteger.decode(encoded, 0, 5), 10);
  }

  @Test
  public void rfc_c1_example_1337() {
    // RFC 7541 C.1.2: Encoding 1337 using a 5-bit prefix
    byte[] encoded = HpackInteger.encode(1337, 5);
    assertEquals(encoded, new byte[]{0x1f, (byte) 0x9a, 0x0a});
    assertEquals(HpackInteger.decode(encoded, 0, 5), 1337);
  }

  @Test
  public void rfc_c1_example_42() {
    // RFC 7541 C.1.3: Encoding 42 using a 8-bit prefix
    byte[] encoded = HpackInteger.encode(42, 8);
    assertEquals(encoded, new byte[]{0x2a});
    assertEquals(HpackInteger.decode(encoded, 0, 8), 42);
  }

  @Test
  public void round_trip() {
    // Round-trip various values with all prefix sizes
    for (int prefix = 4; prefix <= 8; prefix++) {
      for (int value : new int[]{0, 1, 30, 31, 127, 128, 255, 256, 1000, 65535, Integer.MAX_VALUE}) {
        byte[] encoded = HpackInteger.encode(value, prefix);
        int decoded = HpackInteger.decode(encoded, 0, prefix);
        assertEquals(decoded, value, "Failed for value=" + value + " prefix=" + prefix);
      }
    }
  }
}
```

**Step 2: Run tests, verify they fail** (class not found)

**Step 3: Implement HpackInteger**

Reference: RFC 7541 Section 5.1. Two static methods:
- `encode(int value, int prefixBits)` -> `byte[]`
- `decode(byte[] data, int offset, int prefixBits)` -> `int`

Encoding algorithm:
1. If value < 2^N - 1: fits in prefix byte
2. Otherwise: set prefix to all 1s, subtract 2^N-1, encode remainder in continuation bytes (7 bits each, high bit = continuation flag)

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackInteger.java \
        src/test/java/io/fusionauth/http/hpack/HpackIntegerTest.java
git commit -m "Add HPACK variable-length integer encoding (RFC 7541 Section 5.1)"
```

---

### Task 2: HPACK Static Table

The 61-entry static table from RFC 7541 Appendix A.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackStaticTable.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackStaticTableTest.java`

**Step 1: Write HpackStaticTableTest**

Test that:
- Table has exactly 61 entries (indices 1-61, 0 is unused)
- Index 1 = (`:authority`, `""`)
- Index 2 = (`:method`, `GET`)
- Index 3 = (`:method`, `POST`)
- Index 4 = (`:path`, `/`)
- Index 5 = (`:path`, `/index.html`)
- Index 6 = (`:scheme`, `http`)
- Index 7 = (`:scheme`, `https`)
- Index 8 = (`:status`, `200`)
- Lookup by (name, value) returns correct index
- Lookup by name only returns first matching index

**Step 2: Run tests, verify they fail**

**Step 3: Implement HpackStaticTable**

- `static final` array of 61 `HeaderField(name, value)` entries
- `HashMap<String, HashMap<String, Integer>>` for (name, value) -> index lookup
- `HashMap<String, Integer>` for name -> first index lookup
- Methods: `get(int index)`, `findIndex(String name, String value)`, `findNameIndex(String name)`

Reference: RFC 7541 Appendix A for the complete table.

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackStaticTable.java \
        src/test/java/io/fusionauth/http/hpack/HpackStaticTableTest.java
git commit -m "Add HPACK static table (RFC 7541 Appendix A)"
```

---

### Task 3: HPACK Dynamic Table

FIFO table for recently-used headers per connection.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackDynamicTable.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackDynamicTableTest.java`

**Step 1: Write HpackDynamicTableTest**

Test:
- Add entry, retrieve by index (dynamic indices start at static_table_size + 1 = 62)
- FIFO eviction when table exceeds max size
- Entry size = name.length + value.length + 32 (per RFC 7541 Section 4.1)
- Resize table (evicts entries to fit)
- Empty table returns null for lookups
- Index 62 = most recently added entry

**Step 2: Run tests, verify they fail**

**Step 3: Implement HpackDynamicTable**

- Circular buffer or `ArrayDeque` of `HeaderField` entries
- `maxSize` field (default 4096 bytes per SETTINGS_HEADER_TABLE_SIZE)
- `currentSize` tracking
- Methods: `add(name, value)`, `get(int index)`, `setMaxSize(int)`, `size()`
- On add: insert at front, evict from back until currentSize <= maxSize

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackDynamicTable.java \
        src/test/java/io/fusionauth/http/hpack/HpackDynamicTableTest.java
git commit -m "Add HPACK dynamic table (RFC 7541 Section 2.3.2)"
```

---

### Task 4: HPACK Huffman Encoding

Fixed Huffman code table from RFC 7541 Appendix B.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackHuffman.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackHuffmanTest.java`

**Step 1: Write HpackHuffmanTest**

Test using RFC 7541 Appendix C examples:
- `www.example.com` encodes to known bytes (C.4.1)
- `no-cache` encodes to known bytes (C.4.2)
- Round-trip encode/decode for ASCII strings
- Edge case: empty string
- Error case: invalid Huffman padding (should throw)

**Step 2: Run tests, verify they fail**

**Step 3: Implement HpackHuffman**

Two components:
1. **Encode**: Lookup table `int[] CODES` (256 entries) + `byte[] CODE_LENGTHS`. For each byte, output the corresponding bit pattern. Pad with EOS symbol bits at end.
2. **Decode**: Build a decode tree or state machine from the Huffman table. Each node has 0/1 children. Walk the tree bit-by-bit, emitting bytes at leaf nodes.

The Huffman table is a static constant — copy from RFC 7541 Appendix B. This is ~300 lines of table data.

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackHuffman.java \
        src/test/java/io/fusionauth/http/hpack/HpackHuffmanTest.java
git commit -m "Add HPACK Huffman encoding (RFC 7541 Appendix B)"
```

---

### Task 5: HPACK Decoder

Decodes binary header blocks into name/value pairs.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackDecoder.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackDecoderTest.java`

**Step 1: Write HpackDecoderTest**

Test using RFC 7541 Appendix C.3 and C.4 (request examples without and with Huffman):
- C.3.1: First request — `:method GET`, `:scheme http`, `:path /`, `:authority www.example.com`
- C.3.2: Second request — same, with `cache-control: no-cache` added, dynamic table changes
- C.3.3: Third request — different path, custom-key/custom-value
- C.4.1-C.4.3: Same examples but with Huffman encoding
- Security: oversized header bomb (enforce max header list size)
- Security: invalid table index (expect COMPRESSION_ERROR)

**Step 2: Run tests, verify they fail**

**Step 3: Implement HpackDecoder**

Reads representation type from prefix bits:
- `1xxxxxxx` → Indexed (full match in static or dynamic table)
- `01xxxxxx` → Literal with indexing (add to dynamic table)
- `0000xxxx` → Literal without indexing
- `0001xxxx` → Literal never indexed
- `001xxxxx` → Dynamic table size update

Uses HpackInteger for prefix parsing, HpackStaticTable + HpackDynamicTable for lookups, HpackHuffman for string decoding.

Constructor takes: `maxDynamicTableSize`, `maxHeaderListSize`

Method: `decode(byte[] headerBlock) -> List<HeaderField>`

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackDecoder.java \
        src/test/java/io/fusionauth/http/hpack/HpackDecoderTest.java
git commit -m "Add HPACK decoder (RFC 7541)"
```

---

### Task 6: HPACK Encoder

Encodes name/value pairs into binary header blocks.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/hpack/HpackEncoder.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackEncoderTest.java`
- Create: `src/test/java/io/fusionauth/http/hpack/HpackRoundTripTest.java`

**Step 1: Write HpackEncoderTest and HpackRoundTripTest**

Encoder tests: verify output matches RFC C.3/C.4 examples.

Round-trip tests: encode arbitrary headers with our encoder, decode with our decoder, verify identity. Include:
- Common HTTP headers (Content-Type, Content-Length, etc.)
- Custom headers
- Sensitive headers (Authorization, Cookie — should use never-indexed)
- Large header values
- Many headers (exercise dynamic table eviction)

**Step 2: Run tests, verify they fail**

**Step 3: Implement HpackEncoder**

Encoding strategy per header:
1. Check static table for (name, value) match → indexed representation
2. Check dynamic table for (name, value) match → indexed representation
3. Check static/dynamic table for name-only match → literal with indexing (reuse name index)
4. No match → literal with indexing (new name + new value)
5. Sensitive headers (Authorization, Cookie, Set-Cookie) → literal never indexed

Constructor takes: `maxDynamicTableSize`, `useHuffman` (boolean)

Method: `encode(List<HeaderField> headers) -> byte[]`

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/hpack/HpackEncoder.java \
        src/test/java/io/fusionauth/http/hpack/HpackEncoderTest.java \
        src/test/java/io/fusionauth/http/hpack/HpackRoundTripTest.java
git commit -m "Add HPACK encoder with round-trip tests (RFC 7541)"
```

---

### Task 7: HPACK RFC Vector Tests

Verify against the official RFC test vectors as a comprehensive validation.

**Files:**
- Create: `src/test/java/io/fusionauth/http/hpack/HpackRFCVectorTest.java`

**Step 1: Write HpackRFCVectorTest**

Implement ALL RFC 7541 Appendix C test vectors:
- C.1: Integer encoding examples
- C.2: Header field representation examples
- C.3: Request examples without Huffman
- C.4: Request examples with Huffman
- C.5: Response examples without Huffman
- C.6: Response examples with Huffman

Each test decodes the hex input and verifies the expected headers AND the dynamic table state after each step.

**Step 2: Run tests, verify they pass** (should already pass if Tasks 1-6 are correct)

**Step 3: Commit**
```bash
git add src/test/java/io/fusionauth/http/hpack/HpackRFCVectorTest.java
git commit -m "Add HPACK RFC 7541 Appendix C vector tests"
```

---

### Task 8: HTTP/2 Frame Types and Error Codes

Constants and enums for HTTP/2 frame types and error codes.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameType.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Error.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Flags.java`

**Step 1: Implement frame type enum**

```java
public enum HTTP2FrameType {
  DATA(0x0),
  HEADERS(0x1),
  PRIORITY(0x2),
  RST_STREAM(0x3),
  SETTINGS(0x4),
  PUSH_PROMISE(0x5),
  PING(0x6),
  GOAWAY(0x7),
  WINDOW_UPDATE(0x8),
  CONTINUATION(0x9);

  public final int id;
  // fromId(int) static method for lookup
}
```

**Step 2: Implement error code enum**

All 14 error codes from RFC 9113 Section 7 (NO_ERROR through HTTP_1_1_REQUIRED).

**Step 3: Implement flags constants**

```java
public final class HTTP2Flags {
  public static final byte END_STREAM = 0x1;
  public static final byte END_HEADERS = 0x4;
  public static final byte PADDED = 0x8;
  public static final byte PRIORITY = 0x20;
  public static final byte ACK = 0x1; // For SETTINGS and PING
}
```

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameType.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Error.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Flags.java
git commit -m "Add HTTP/2 frame types, error codes, and flags (RFC 9113)"
```

---

### Task 9: HTTP/2 Frame Reader

Reads 9-byte frame headers and payloads from an InputStream.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Frame.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameReader.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2FrameReaderTest.java`

**Step 1: Write HTTP2FrameReaderTest**

Test:
- Read a SETTINGS frame (type 0x4, stream 0, known payload)
- Read a PING frame (type 0x6, 8-byte payload)
- Read a DATA frame with END_STREAM flag
- Read a HEADERS frame
- Error: frame exceeds SETTINGS_MAX_FRAME_SIZE (should throw)
- Error: truncated frame (EOF mid-read)
- Unknown frame type: read and discard payload

**Step 2: Run tests, verify they fail**

**Step 3: Implement HTTP2Frame and HTTP2FrameReader**

`HTTP2Frame` is a simple data class:
```java
public class HTTP2Frame {
  public int length;       // 24-bit payload length
  public HTTP2FrameType type;
  public byte flags;
  public int streamId;     // 31-bit, mask off reserved bit
  public byte[] payload;
}
```

`HTTP2FrameReader` reads from InputStream:
1. Read 9-byte header
2. Parse length (3 bytes big-endian), type, flags, streamId (4 bytes, mask 0x7FFFFFFF)
3. Validate length <= maxFrameSize
4. Read `length` bytes of payload
5. Return `HTTP2Frame`

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Frame.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameReader.java \
        src/test/java/io/fusionauth/http/http2/HTTP2FrameReaderTest.java
git commit -m "Add HTTP/2 frame reader (RFC 9113 Section 4)"
```

---

### Task 10: HTTP/2 Frame Writer

Writes frames to an OutputStream. Thread-safe (multiple stream threads write concurrently).

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameWriter.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2FrameWriterTest.java`

**Step 1: Write HTTP2FrameWriterTest**

Test:
- Write a SETTINGS frame, read it back with FrameReader, verify equality
- Write a PING frame with ACK
- Write a HEADERS frame (verify HEADERS+CONTINUATION lock — no interleaving)
- Thread safety: 10 concurrent threads writing DATA frames, verify no corruption (frames are intact when read back)

**Step 2: Run tests, verify they fail**

**Step 3: Implement HTTP2FrameWriter**

- `synchronized` write methods (start with lock, optimize later per design spec Section 19.4)
- Methods: `writeSettings(...)`, `writePing(...)`, `writeGoaway(...)`, `writeHeaders(...)`, `writeData(...)`, `writeWindowUpdate(...)`, `writeRstStream(...)`
- Each method constructs the 9-byte header + payload and writes atomically (within the lock)
- HEADERS+CONTINUATION: hold lock for the entire sequence until END_HEADERS is set

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FrameWriter.java \
        src/test/java/io/fusionauth/http/http2/HTTP2FrameWriterTest.java
git commit -m "Add HTTP/2 frame writer (thread-safe, RFC 9113 Section 4)"
```

---

### Task 11: HTTP/1.1 Critical Fix — Date Header Auto-Generation

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/io/HTTPOutputStream.java`
- Add test to existing test file (or create new one)

**Step 1: Write test**

Test that a response without an explicit Date header gets one auto-generated. Test that a response with an explicit Date header is not overwritten.

**Step 2: Run tests, verify the first fails**

**Step 3: Implement**

In `HTTPOutputStream`, during response commit (when headers are written to the socket), check if `response.containsHeader("Date")`. If not, add one with `DateTools.format(ZonedDateTime.now(ZoneOffset.UTC))`.

Look at the existing `commit()` or `writeHeaders()` method in HTTPOutputStream to find the right insertion point.

**Step 4: Run ALL tests** to verify no regressions

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/io/HTTPOutputStream.java
git commit -m "Auto-generate Date header on responses (RFC 9110 Section 6.6.1)"
```

---

### Task 12: HTTP/1.1 Critical Fix — HEAD Method Body Suppression

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/io/HTTPOutputStream.java`
- Add test

**Step 1: Write test**

Test that a HEAD request returns headers (including Content-Length) but no body bytes. The handler should be able to write to the response output stream (as if it were GET), but the server discards the body.

**Step 2: Run tests, verify fails**

**Step 3: Implement**

In `HTTPOutputStream`, when the request method is HEAD:
- Still allow `write()` calls (so the handler doesn't need special logic)
- Count bytes written (for Content-Length)
- But don't actually write body bytes to the socket

This can be done by checking the method in the output stream and skipping the actual socket writes while still tracking content length.

**Step 4: Run ALL tests**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/io/HTTPOutputStream.java
git commit -m "Suppress body output for HEAD responses (RFC 9110 Section 9.3.2)"
```

---

## Phase 2: Connection Handling (~1 week)

### Task 13: HTTP/2 Settings

SETTINGS frame values and negotiation.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Settings.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2SettingsTest.java`

**Step 1: Write HTTP2SettingsTest**

Test:
- Default settings match RFC 9113 defaults
- Parse a SETTINGS frame payload into an HTTP2Settings object
- Serialize an HTTP2Settings to a SETTINGS frame payload
- Round-trip: serialize -> parse -> verify equality
- Individual setting IDs (0x1 through 0x6) mapped correctly

**Step 2: Implement HTTP2Settings**

```java
public class HTTP2Settings {
  // Setting IDs (RFC 9113 Section 6.5.2)
  public static final int HEADER_TABLE_SIZE = 0x1;
  public static final int ENABLE_PUSH = 0x2;
  public static final int MAX_CONCURRENT_STREAMS = 0x3;
  public static final int INITIAL_WINDOW_SIZE = 0x4;
  public static final int MAX_FRAME_SIZE = 0x5;
  public static final int MAX_HEADER_LIST_SIZE = 0x6;

  // Fields with RFC defaults
  private int headerTableSize = 4096;
  private boolean enablePush = false; // We don't support push
  private int maxConcurrentStreams = 100;
  private int initialWindowSize = 65535;
  private int maxFrameSize = 16384;
  private int maxHeaderListSize = 8192;

  // Parse from SETTINGS frame payload (6 bytes per setting: 2-byte ID + 4-byte value)
  public static HTTP2Settings parse(byte[] payload) { ... }

  // Serialize to SETTINGS frame payload
  public byte[] toPayload() { ... }
}
```

**Step 3: Run tests, verify they pass**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Settings.java \
        src/test/java/io/fusionauth/http/http2/HTTP2SettingsTest.java
git commit -m "Add HTTP/2 SETTINGS frame handling (RFC 9113 Section 6.5)"
```

---

### Task 14: HTTP/2 Configuration API

Public configuration class for HTTP/2 settings.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/HTTP2Configuration.java`
- Modify: `src/main/java/io/fusionauth/http/server/HTTPServerConfiguration.java`
- Modify: `src/main/java/io/fusionauth/http/server/Configurable.java`

**Step 1: Implement HTTP2Configuration**

Follow the `MultipartConfiguration` pattern (already in the codebase):

```java
public class HTTP2Configuration {
  private int maxConcurrentStreams = 100;
  private int initialWindowSize = 65535;
  private int maxFrameSize = 16384;
  private int maxHeaderListSize = 8192;
  private int headerTableSize = 4096;
  private int connectionWindowSize = 65535;
  private boolean enabled = true;

  // Builder methods: withMaxConcurrentStreams(int), etc.
  // Getters for each field
}
```

**Step 2: Add to HTTPServerConfiguration**

Add `HTTP2Configuration http2Configuration` field with getter/setter.

**Step 3: Add to Configurable<T>**

Add default method:
```java
default T withHttp2Configuration(HTTP2Configuration http2Configuration) {
  configuration().setHttp2Configuration(http2Configuration);
  return (T) this;
}
```

**Step 4: Run ALL tests** (verify backward compat — no existing test should break)

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/HTTP2Configuration.java \
        src/main/java/io/fusionauth/http/server/HTTPServerConfiguration.java \
        src/main/java/io/fusionauth/http/server/Configurable.java
git commit -m "Add HTTP2Configuration with nested builder pattern"
```

---

### Task 15: ALPN Setup in SecurityTools

Extend TLS configuration to offer `h2` and `http/1.1` via ALPN.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/security/SecurityTools.java`
- Modify: `src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java` (ALPN on SSLServerSocket)

**Step 1: Write test**

Test that when connecting with a TLS client configured for HTTP/2 (ALPN `h2`), the negotiated protocol is `h2`. Test that when connecting with HTTP/1.1 only, the negotiated protocol is `http/1.1`.

Use `javax.net.ssl.SSLSocket.getApplicationProtocol()` to verify.

**Step 2: Implement ALPN setup**

In `HTTPServerThread`, after creating the `SSLServerSocket`, set ALPN parameters:

```java
if (socket instanceof SSLServerSocket sslServerSocket) {
  SSLParameters params = sslServerSocket.getSSLParameters();
  params.setApplicationProtocols(new String[]{"h2", "http/1.1"});
  sslServerSocket.setSSLParameters(params);
}
```

After `accept()`, check the negotiated protocol:
```java
if (clientSocket instanceof SSLSocket sslSocket) {
  sslSocket.startHandshake();
  String protocol = sslSocket.getApplicationProtocol();
  // Route based on protocol
}
```

**Step 3: Run tests, verify pass**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/security/SecurityTools.java \
        src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java
git commit -m "Add ALPN negotiation for h2/http/1.1 on TLS listeners"
```

---

### Task 16: Protocol Detection and Connection Routing

Detect HTTP/2 vs HTTP/1.1 and route to the appropriate handler.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java`

**Step 1: Write test**

Integration test: start a server, connect with h2c prior knowledge (send the 24-byte preface), verify the connection is accepted (we won't have the full handler yet, but we can verify it doesn't crash and the connection isn't routed to HTTPWorker).

Also test: connect with normal HTTP/1.1 request, verify it still works as before.

**Step 2: Implement protocol detection**

For TLS connections: ALPN (from Task 15).

For plaintext connections:
1. Accept socket, wrap in PushbackInputStream
2. Read first 24 bytes
3. Compare against `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n`
4. If match → route to HTTP2ConnectionHandler (stub for now)
5. If no match → push bytes back → route to HTTPWorker (existing)

```java
private static final byte[] HTTP2_PREFACE = "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n".getBytes(StandardCharsets.US_ASCII);
```

**Step 3: Run ALL tests** (critical — HTTP/1.1 must not regress)

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java
git commit -m "Add HTTP/2 protocol detection (ALPN + h2c preface)"
```

---

### Task 17: HTTP2ConnectionHandler — Connection Lifecycle

The main connection-level handler for HTTP/2.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java`

**Step 1: Write test**

Integration test: connect to the server, send the HTTP/2 connection preface + SETTINGS frame, verify the server responds with its own SETTINGS frame + SETTINGS ACK.

Also test PING: send a PING, expect a PING with ACK flag and same payload.

**Step 2: Implement HTTP2ConnectionHandler**

Implements `Runnable` (runs on a virtual thread, like HTTPWorker):

```java
public class HTTP2ConnectionHandler implements Runnable {
  // Constructor takes: socket, configuration, handler, instrumenter, etc.

  @Override
  public void run() {
    // 1. Read client's connection preface (24-byte magic already consumed by detection)
    // 2. Read client's SETTINGS frame
    // 3. Send our SETTINGS frame
    // 4. Send SETTINGS ACK for client's settings
    // 5. Enter frame read loop:
    //    - Read frame via HTTP2FrameReader
    //    - Dispatch by frame type
    //    - SETTINGS → update peer settings, send ACK
    //    - PING → respond with PING ACK
    //    - GOAWAY → initiate graceful shutdown
    //    - HEADERS → create new stream (Task 18+)
    //    - DATA → route to stream (Task 18+)
    //    - WINDOW_UPDATE → update flow control (Task 19)
    //    - RST_STREAM → cancel stream (Task 18+)
    //    - Unknown → ignore (read and discard)
  }
}
```

Start with just SETTINGS exchange and PING handling. Stream handling comes in Phase 3.

**Step 3: Wire into HTTPServerThread**

Update the routing logic from Task 16 to actually spawn `HTTP2ConnectionHandler` instead of a stub.

**Step 4: Run tests**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java \
        src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java
git commit -m "Add HTTP2ConnectionHandler with SETTINGS exchange and PING"
```

---

## Phase 3: Stream Processing (~1.5 weeks)

### Task 18: HTTP2Stream State Machine

Per-stream state tracking and lifecycle.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2StreamState.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2StreamTest.java`

**Step 1: Write HTTP2StreamTest**

Test state transitions:
- idle → open (receive HEADERS)
- open → half-closed(remote) (receive END_STREAM)
- open → closed (receive RST_STREAM)
- half-closed(remote) → closed (send END_STREAM)
- Invalid transitions throw (e.g., HEADERS on closed stream)

**Step 2: Implement HTTP2StreamState enum**

```java
public enum HTTP2StreamState {
  IDLE, OPEN, HALF_CLOSED_LOCAL, HALF_CLOSED_REMOTE, CLOSED
}
```

**Step 3: Implement HTTP2Stream**

```java
public class HTTP2Stream {
  private final int streamId;
  private HTTP2StreamState state = HTTP2StreamState.IDLE;
  private HTTPRequest request;
  private HTTPResponse response;
  // Input buffer for DATA frames (PipedInputStream/PipedOutputStream or custom buffer)
  // Flow control windows
  // Thread reference
}
```

**Step 4: Run tests, verify they pass**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2StreamState.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java \
        src/test/java/io/fusionauth/http/http2/HTTP2StreamTest.java
git commit -m "Add HTTP/2 stream state machine (RFC 9113 Section 5.1)"
```

---

### Task 19: HTTP/2 Flow Control

Connection-level and per-stream window management.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FlowControl.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2FlowControlTest.java`

**Step 1: Write HTTP2FlowControlTest**

Test:
- Initial window sizes are correct (65535 default)
- Sending DATA decreases send window
- Receiving WINDOW_UPDATE increases send window
- Window exhaustion blocks (verify with virtual thread timeout)
- SETTINGS_INITIAL_WINDOW_SIZE change applies delta to all open streams
- Negative window after SETTINGS change is handled correctly (block until positive)

**Step 2: Implement HTTP2FlowControl**

```java
public class HTTP2FlowControl {
  private int connectionSendWindow;
  private int connectionReceiveWindow;

  // Per-stream windows tracked in HTTP2Stream

  // Called before writing DATA: blocks until window available
  public int acquireSendWindow(HTTP2Stream stream, int desired) { ... }

  // Called after writing DATA
  public void consumeSendWindow(HTTP2Stream stream, int bytes) { ... }

  // Called when DATA received
  public void consumeReceiveWindow(HTTP2Stream stream, int bytes) { ... }

  // Called when WINDOW_UPDATE received
  public void updateSendWindow(int streamId, int increment) { ... }

  // Auto WINDOW_UPDATE: send when receive window drops below 50%
  public boolean shouldSendWindowUpdate(int currentWindow, int initialWindow) { ... }
}
```

**Step 3: Run tests, verify they pass**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2FlowControl.java \
        src/test/java/io/fusionauth/http/http2/HTTP2FlowControlTest.java
git commit -m "Add HTTP/2 flow control (RFC 9113 Section 5.2)"
```

---

### Task 20: HEADERS → HTTPRequest Mapping

Decode HEADERS frames into the existing HTTPRequest object.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java`
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java`
- Modify: `src/main/java/io/fusionauth/http/HTTPValues.java` (add pseudo-header constants)

**Step 1: Write test**

Integration test: start a real server with a handler that echoes the request method, path, and a custom header back as JSON. Send an HTTP/2 request using JDK HttpClient with `HttpClient.Version.HTTP_2`. Verify the handler received the correct HTTPRequest fields.

```java
var client = HttpClient.newBuilder().version(HttpClient.Version.HTTP_2).build();
var request = HttpRequest.newBuilder(URI.create("http://localhost:4242/test?foo=bar"))
                         .header("X-Custom", "hello")
                         .GET().build();
var response = client.send(request, HttpResponse.BodyHandlers.ofString());
// Verify response contains the echoed values
```

**Step 2: Implement HEADERS → HTTPRequest mapping**

In HTTP2ConnectionHandler, when a HEADERS frame is received:
1. HPACK-decode the header block → list of (name, value)
2. Create a new HTTPRequest
3. Map pseudo-headers:
   - `:method` → `request.setMethod(HTTPMethod.of(value))`
   - `:path` → `request.setPath(value)` (parse query string)
   - `:scheme` → `request.setScheme(value)`
   - `:authority` → `request.setHost(host)` and `request.setPort(port)`
4. Map regular headers into `request.setHeader(name, value)`
5. Create HTTP2Stream, set state to OPEN
6. If END_STREAM → state = HALF_CLOSED_REMOTE (no body)
7. Spawn virtual thread → call `HTTPHandler.handle(request, response)`

Add to HTTPValues:
```java
public static final String PseudoAuthority = ":authority";
public static final String PseudoMethod = ":method";
public static final String PseudoPath = ":path";
public static final String PseudoScheme = ":scheme";
public static final String PseudoStatus = ":status";
```

**Step 3: Run tests**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java \
        src/main/java/io/fusionauth/http/HTTPValues.java
git commit -m "Map HTTP/2 HEADERS to HTTPRequest via HPACK"
```

---

### Task 21: HTTPResponse → HEADERS + DATA Frames

Write HTTP/2 responses back to the client.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2OutputStream.java`
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java`

**Step 1: Write test**

Test: send an HTTP/2 request, handler sets status 200 and writes "Hello world" to the response. Verify the client receives status 200 and body "Hello world".

Test: large response (1MB) — verify flow control doesn't deadlock and data arrives intact.

Test: empty response (headers only, no body) — verify END_STREAM is set on the HEADERS frame.

**Step 2: Implement HTTP2OutputStream**

This replaces HTTPOutputStream for HTTP/2 streams. When the handler writes to the response:

1. On first write (or `commit()`): HPACK-encode response headers into HEADERS frame
   - `:status` pseudo-header from response status code
   - Regular headers from response headers map
   - Set END_STREAM on HEADERS if no body
2. Body writes → DATA frames through the connection's frame writer
   - Respect flow control (call `flowControl.acquireSendWindow()` before each DATA frame)
   - Split body into chunks ≤ maxFrameSize
   - Set END_STREAM on last DATA frame

**Step 3: Run tests**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2OutputStream.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java
git commit -m "Add HTTP/2 response encoding (HEADERS + DATA frames)"
```

---

### Task 22: DATA Frame → InputStream for Handler

Route incoming DATA frames to the handler's input stream.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java`
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java`

**Step 1: Write test**

Test: send a POST request with body over HTTP/2. Handler reads the body from `request.getInputStream()`. Verify the handler receives the complete body.

Test: large POST body (>64KB) — verify flow control works and WINDOW_UPDATE is sent.

**Step 2: Implement stream input buffer**

HTTP2Stream needs a pipe from the connection reader to the stream's virtual thread:
- Use `PipedOutputStream` (written by connection reader) + `PipedInputStream` (read by handler thread)
- OR a custom bounded buffer (simpler, avoids PipedStream quirks)

When the connection reader receives a DATA frame:
1. Look up stream by streamId
2. Write payload to stream's input buffer
3. Consume receive window
4. If receive window low, send WINDOW_UPDATE
5. If END_STREAM flag, signal EOF on the input buffer

**Step 3: Run tests**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java
git commit -m "Route HTTP/2 DATA frames to stream input buffers"
```

---

### Task 23: Stream Multiplexing

Verify multiple concurrent streams work correctly on one connection.

**Files:**
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2MultiplexTest.java`

**Step 1: Write HTTP2MultiplexTest**

Test: send 10 concurrent HTTP/2 requests on the same connection. Each request hits a handler that sleeps briefly (random 10-50ms) then returns a unique response. Verify all 10 responses arrive correctly.

```java
var client = HttpClient.newBuilder().version(HttpClient.Version.HTTP_2).build();
List<CompletableFuture<HttpResponse<String>>> futures = new ArrayList<>();
for (int i = 0; i < 10; i++) {
  var request = HttpRequest.newBuilder(URI.create("http://localhost:4242/test?id=" + i)).GET().build();
  futures.add(client.sendAsync(request, HttpResponse.BodyHandlers.ofString()));
}
// Wait for all, verify each has correct response
```

Test: exceed MAX_CONCURRENT_STREAMS — verify server sends RST_STREAM with REFUSED_STREAM.

**Step 2: Run tests**

**Step 3: Commit**
```bash
git add src/test/java/io/fusionauth/http/http2/HTTP2MultiplexTest.java
git commit -m "Add HTTP/2 stream multiplexing tests"
```

---

### Task 24: GOAWAY and Graceful Shutdown

Handle connection shutdown cleanly.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java`
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2ErrorTest.java`

**Step 1: Write tests**

Test graceful shutdown:
- Start server, send requests, call `server.close()`, verify GOAWAY is sent, in-flight requests complete, new requests are refused.

Test error GOAWAY:
- Send an invalid frame (e.g., SETTINGS on non-zero stream ID), verify server sends GOAWAY with PROTOCOL_ERROR.

Test RST_STREAM:
- Open a stream, send RST_STREAM from client, verify server cancels the stream's handler thread.

**Step 2: Implement**

In HTTP2ConnectionHandler:
- `sendGoaway(lastStreamId, errorCode)` method
- Wire `HTTPServer.close()` to trigger GOAWAY on all HTTP/2 connections
- Handle received GOAWAY from client (stop creating new streams)
- Handle RST_STREAM (interrupt stream thread, clean up)

**Step 3: Run tests**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java \
        src/test/java/io/fusionauth/http/http2/HTTP2ErrorTest.java
git commit -m "Add HTTP/2 GOAWAY, RST_STREAM, and graceful shutdown"
```

---

## Phase 4: Configuration and Integration (~0.5 weeks)

### Task 25: Instrumenter Extension for HTTP/2

Add HTTP/2 metrics to the monitoring interface.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/Instrumenter.java`
- Modify: `src/main/java/io/fusionauth/http/server/CountingInstrumenter.java`
- Modify: `src/main/java/io/fusionauth/http/server/ThreadSafeCountingInstrumenter.java`

**Step 1: Add default methods to Instrumenter**

```java
// New default methods (backward compatible)
default void http2ConnectionOpened() {}
default void http2ConnectionClosed() {}
default void http2StreamOpened() {}
default void http2StreamClosed() {}
default void http2GoawaySent() {}
```

**Step 2: Implement in CountingInstrumenter and ThreadSafeCountingInstrumenter**

Add counters for each new method.

**Step 3: Wire into HTTP2ConnectionHandler and HTTP2Stream**

Call instrumenter methods at the appropriate lifecycle points.

**Step 4: Run ALL tests**

**Step 5: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/Instrumenter.java \
        src/main/java/io/fusionauth/http/server/CountingInstrumenter.java \
        src/main/java/io/fusionauth/http/server/ThreadSafeCountingInstrumenter.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java \
        src/main/java/io/fusionauth/http/server/internal/http2/HTTP2Stream.java
git commit -m "Add HTTP/2 instrumentation events"
```

---

### Task 26: HTTPServerCleanerThread — HTTP/2 Connection Monitoring

Extend the cleaner thread to handle HTTP/2 connections.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java` (inner HTTPServerCleanerThread class)

**Step 1: Understand current cleaner**

The cleaner monitors HTTPWorker instances for throughput and timeouts. HTTP/2 connections are longer-lived, so the cleaner needs to distinguish between idle connections (no active streams but connection alive) and dead connections.

**Step 2: Implement**

Track HTTP2ConnectionHandler instances in the cleaner. An HTTP/2 connection is considered dead if:
- No active streams AND idle for longer than `keepAliveTimeoutDuration`
- A PING has been sent but no PONG received within timeout

**Step 3: Run ALL tests**

**Step 4: Commit**
```bash
git add src/main/java/io/fusionauth/http/server/internal/HTTPServerThread.java
git commit -m "Extend cleaner thread for HTTP/2 connection monitoring"
```

---

## Phase 5: Testing (~1.5 weeks)

### Task 27: HTTP/2 Core Integration Tests

Full end-to-end tests using JDK HttpClient.

**Files:**
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2CoreTest.java`

Test cases (follow existing test patterns — extend BaseTest, use `schemes()` DataProvider):
- GET request → 200 response
- POST request with body → echo response
- PUT, DELETE methods
- Large response bodies (1MB, 10MB)
- Large request bodies
- Multiple sequential requests on same connection
- Response headers are preserved
- Cookies work over HTTP/2
- Content-Type negotiation
- Query parameters
- Status codes (200, 400, 404, 500)

Each test uses both `http` (h2c) and `https` (h2) schemes.

**Commit after tests pass.**

---

### Task 28: HTTP/2 Socket-Level Tests

Raw frame-level tests for protocol edge cases.

**Files:**
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2SocketTest.java`

Test cases (write raw bytes to socket, read raw frame responses):
- Invalid connection preface → server closes connection
- SETTINGS on non-zero stream → GOAWAY PROTOCOL_ERROR
- DATA on stream 0 → GOAWAY PROTOCOL_ERROR
- Invalid frame size → GOAWAY FRAME_SIZE_ERROR
- HEADERS with invalid stream ID (even number) → GOAWAY PROTOCOL_ERROR
- PING with wrong payload size (not 8 bytes) → GOAWAY FRAME_SIZE_ERROR
- WINDOW_UPDATE with 0 increment → GOAWAY/RST_STREAM PROTOCOL_ERROR
- Unknown frame type → silently ignored

Use `invocationCount = 100` for race detection.

**Commit after tests pass.**

---

### Task 29: Mixed Protocol Tests

Verify HTTP/1.1 and HTTP/2 work simultaneously on the same server.

**Files:**
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2MixedProtocolTest.java`

Test:
- Start server, send HTTP/1.1 request and HTTP/2 request concurrently, both succeed
- Same handler serves both protocols correctly
- HTTP/1.1 connection keeps working after HTTP/2 connections are opened
- Backward compatibility: run a selection of existing HTTP/1.1 tests against the updated server code

**Commit after tests pass.**

---

### Task 30: Backward Compatibility Verification

Run the full existing test suite and verify zero regressions.

**Step 1:** Run `sb test` — all 2505+ existing tests must pass.

**Step 2:** If any failures, investigate and fix. The HTTP/2 changes should not affect any existing behavior.

**Commit any fixes needed.**

---

## Phase 6: Load Testing and Optimization (~1 week)

### Task 31: HTTP/2 Load Test Configuration

Update the load test server to support HTTP/2.

**Files:**
- Modify: `load-tests/self/src/main/java/io/fusionauth/http/load/Main.java`

Add an HTTP/2 listener (either h2c on port 8080 or h2 on a TLS port). The LoadHandler is already protocol-agnostic.

**Commit after working.**

---

### Task 32: Performance Profiling

Profile HTTP/2 under load and optimize bottlenecks.

**Steps:**
1. Run h2load against the load test server
2. Profile with `async-profiler` or JDK Flight Recorder
3. Identify bottlenecks (frame writer contention, HPACK, flow control)
4. Optimize based on profiling data (e.g., frame writer queue if lock is contended)

**Commit optimizations.**

---

## Phase 7: Upgrade Mechanism and WebSocket (~2 weeks)

### Task 33: UpgradeHandler Interface

Public interface for protocol upgrades.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/UpgradeHandler.java`

```java
@FunctionalInterface
public interface UpgradeHandler {
  void handle(HTTPRequest request, Socket socket, InputStream in, OutputStream out) throws Exception;
}
```

**Commit immediately** (interface only).

---

### Task 34: Upgrade Detection in HTTPWorker

Detect `Connection: Upgrade` and dispatch to registered handlers.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/HTTPWorker.java`
- Modify: `src/main/java/io/fusionauth/http/server/HTTPServerConfiguration.java`
- Modify: `src/main/java/io/fusionauth/http/server/Configurable.java`
- Modify: `src/main/java/io/fusionauth/http/HTTPValues.java`

**Step 1: Write test**

Send an HTTP/1.1 request with `Connection: Upgrade` and `Upgrade: test-protocol` to a server with a registered UpgradeHandler for `test-protocol`. Verify the server sends 101 and the handler receives the socket.

**Step 2: Implement**

In HTTPWorker, after parsing request headers:
1. Check for `Connection: Upgrade` header
2. Read the `Upgrade` header value
3. Look up the registered UpgradeHandler for that protocol
4. If found: write 101 Switching Protocols response, call handler with socket
5. If not found: continue normal processing

Add to Configurable:
```java
default T withUpgradeHandler(String protocol, UpgradeHandler handler) { ... }
```

Add to HTTPValues:
```java
public static final String Upgrade = "Upgrade";
public static final int SwitchingProtocols = 101;
```

**Step 3: Run ALL tests**

**Step 4: Commit**

---

### Task 35: h2c Upgrade Handler

HTTP/1.1 → HTTP/2 upgrade via 101 Switching Protocols.

**Files:**
- Modify: `src/main/java/io/fusionauth/http/server/internal/http2/HTTP2ConnectionHandler.java` (add upgrade constructor)
- Create: `src/test/java/io/fusionauth/http/http2/HTTP2UpgradeTest.java`

The h2c upgrade handler:
1. Decodes `HTTP2-Settings` header (base64url-encoded client SETTINGS)
2. Writes 101 response
3. Creates HTTP2ConnectionHandler with the original request pre-loaded as stream 1
4. Runs the connection handler (which processes the original request and then enters normal frame read loop)

**Commit after tests pass.**

---

### Task 36: WebSocket Public API

Handler and session interfaces for WebSocket.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/WebSocketHandler.java`
- Create: `src/main/java/io/fusionauth/http/server/WebSocketSession.java`
- Create: `src/main/java/io/fusionauth/http/server/WebSocketConfiguration.java`

See design spec Section 18 for the complete interface definitions.

**Commit immediately** (interfaces only).

---

### Task 37: WebSocket Frame Reader/Writer

Binary frame encoding and decoding for WebSocket.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/ws/WebSocketFrameReader.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/ws/WebSocketFrameWriter.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/ws/WebSocketCloseCode.java`
- Create: `src/test/java/io/fusionauth/http/ws/WebSocketFrameTest.java`

**Step 1: Write WebSocketFrameTest**

Test:
- Read a text frame (opcode 0x1, masked from client)
- Read a binary frame (opcode 0x2)
- Unmask payload correctly (XOR with 4-byte masking key)
- Write a text frame (unmasked, server→client)
- Payload lengths: 7-bit (0-125), 16-bit extended (126), 64-bit extended (127)
- Read close frame (opcode 0x8) with status code
- Read ping frame, write pong frame

**Step 2: Implement**

Frame reader:
1. Read first 2 bytes (FIN, opcode, MASK, payload length)
2. If payload length = 126: read 2 more bytes (16-bit length)
3. If payload length = 127: read 8 more bytes (64-bit length)
4. If MASK set: read 4-byte masking key
5. Read payload, unmask if needed
6. Return frame object (fin, opcode, payload)

Frame writer:
1. Write first 2 bytes (FIN, opcode, 0 for MASK, payload length)
2. If length > 125: write extended length
3. Write payload (no masking for server→client)

**Step 3: Run tests, verify pass**

**Step 4: Commit**

---

### Task 38: WebSocket Connection Handler

Main WebSocket connection lifecycle.

**Files:**
- Create: `src/main/java/io/fusionauth/http/server/internal/ws/WebSocketConnectionHandler.java`
- Create: `src/main/java/io/fusionauth/http/server/internal/ws/WebSocketSessionImpl.java`
- Create: `src/test/java/io/fusionauth/http/ws/WebSocketConnectionTest.java`

**Step 1: Write WebSocketConnectionTest**

Use JDK 11+ `java.net.http.WebSocket` client:
```java
var wsClient = HttpClient.newHttpClient().newWebSocketBuilder()
    .buildAsync(URI.create("ws://localhost:4242/ws"), new WebSocket.Listener() {
      // Handle onOpen, onText, onClose
    }).join();
wsClient.sendText("hello", true);
// Verify server echoes back
```

Test:
- Open connection, send text message, receive echo
- Send binary message, receive echo
- Send close, verify close handshake
- Ping/pong keepalive
- Multiple messages in sequence
- Large message (>125 bytes, >65535 bytes)
- Fragmented message (continuation frames)

**Step 2: Implement**

WebSocketConnectionHandler:
1. Validate handshake (check Sec-WebSocket-Key, Version 13)
2. Compute Sec-WebSocket-Accept: `Base64(SHA-1(key + "258EAFA5-E914-47DA-95CA-5AB0DC85B11B"))`
3. Write 101 response with Sec-WebSocket-Accept
4. Create WebSocketSessionImpl
5. Call `handler.onOpen(session)`
6. Frame read loop:
   - Read frame via WebSocketFrameReader
   - Text → `handler.onMessage(session, text)` (assemble if fragmented)
   - Binary → `handler.onMessage(session, bytes)`
   - Ping → auto-respond with Pong
   - Close → `handler.onClose(session, code, reason)`, send close back, exit
7. On error → `handler.onError(session, error)`, close

WebSocketSessionImpl:
- `send(String)` → write text frame via WebSocketFrameWriter
- `send(byte[])` → write binary frame
- `ping(byte[])` → write ping frame
- `close(int, String)` → write close frame
- Thread-safe sends (synchronized on writer)

**Step 3: Wire into server**

Register WebSocket upgrade handler in the Upgrade mechanism:
```java
server.withWebSocketHandler("/ws", new MyWebSocketHandler())
```

This registers an UpgradeHandler for protocol `websocket` that path-matches and dispatches to WebSocketConnectionHandler.

Add `withWebSocketHandler(String path, WebSocketHandler handler)` to Configurable.

**Step 4: Run tests**

**Step 5: Commit**

---

### Task 39: WebSocket Error Handling and Edge Cases

**Files:**
- Create: `src/test/java/io/fusionauth/http/ws/WebSocketErrorTest.java`
- Create: `src/test/java/io/fusionauth/http/ws/WebSocketHandshakeTest.java`

Test:
- Missing Sec-WebSocket-Key → 400
- Wrong Sec-WebSocket-Version → 400
- Unregistered path → 404
- Unmasked client frame → close with 1002 (protocol error)
- Invalid UTF-8 in text frame → close with 1007
- Server-initiated close
- Client disconnects abruptly → handler.onError called

**Commit after tests pass.**

---

## Phase 8: Benchmark Framework (~1 week)

### Task 40: Netty Load Test Server

**Files:**
- Create: `load-tests/netty/build.savant`
- Create: `load-tests/netty/src/main/java/io/fusionauth/http/load/NettyLoadServer.java`
- Create: `load-tests/netty/start.sh`

Implement the same 5 endpoints (no-op, no-read, hello, file, load) as an embedded Netty server on port 8080.

**Commit after working.**

---

### Task 41: Jetty Load Test Server

**Files:**
- Create: `load-tests/jetty/build.savant`
- Create: `load-tests/jetty/src/main/java/io/fusionauth/http/load/JettyLoadServer.java`
- Create: `load-tests/jetty/start.sh`

Same 5 endpoints as an embedded Jetty server.

**Commit after working.**

---

### Task 42: JDK HttpServer Load Test

**Files:**
- Create: `load-tests/jdk-httpserver/build.savant`
- Create: `load-tests/jdk-httpserver/src/main/java/io/fusionauth/http/load/JdkLoadServer.java`
- Create: `load-tests/jdk-httpserver/start.sh`

Same 5 endpoints using `com.sun.net.httpserver.HttpServer`.

**Commit after working.**

---

### Task 43: Benchmark Runner Script

**Files:**
- Create: `load-tests/run-benchmarks.sh`

The script:
1. Accepts flags: `--servers`, `--scenarios`, `--protocol`, `--label`, `--output`, `--update-readme`
2. Auto-detects system metadata:
   - macOS: `sysctl -n machdep.cpu.brand_string`, `sysctl -n hw.ncpu`, `sysctl -n hw.memsize`
   - Linux: parse `/proc/cpuinfo`, `/proc/meminfo`
   - Java: `java -version`
3. Prompts for machine description if `--label` not provided
4. For each server: `cd load-tests/<server> && sb clean start`, wait for ready (poll port 8080), run h2load scenarios, kill server
5. Parse h2load output into JSON
6. Write results to `load-tests/results/YYYY-MM-DD-<label>.json`

**Commit after working.**

---

### Task 44: README Auto-Updater

**Files:**
- Create: `load-tests/update-readme.sh`

Reads the latest results JSON from `load-tests/results/`, generates a markdown performance table, and replaces the `## Performance` section in `README.md`.

**Commit after working.**

---

### Task 45: Upgrade Tomcat for HTTP/2

**Files:**
- Modify: `load-tests/tomcat/build.savant` (upgrade Tomcat dependency to 10.x)
- Modify: `load-tests/tomcat/server.xml` (enable HTTP/2 connector)
- Modify: `load-tests/tomcat/src/main/java/io/fusionauth/http/load/LoadServlet.java` (upgrade from javax to jakarta if needed)

**Commit after working.**

---

### Task 46: GitHub Actions Benchmark Workflow

**Files:**
- Create: `.github/workflows/benchmark.yml`

```yaml
name: Benchmark
on:
  workflow_dispatch:
    inputs:
      servers:
        description: 'Servers to benchmark'
        default: 'self'
      scenarios:
        description: 'Scenarios to run'
        default: 'all'

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Install h2load
        run: sudo apt-get install -y nghttp2-client
      - name: Run benchmarks
        run: ./load-tests/run-benchmarks.sh --servers ${{ inputs.servers }} --label "gha-runner" --output load-tests/results/
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: benchmark-results
          path: load-tests/results/
```

**Commit after working.**

---

## Final: Review and Release Preparation

### Task 47: Full Test Suite Pass

Run `sb test` and verify all tests (original + new) pass with zero failures.

### Task 48: Code Review

Review all new code for:
- Consistent code style
- Proper error handling
- Security (HPACK bombs, frame floods, etc.)
- Thread safety
- Resource cleanup (sockets, streams)

### Task 49: Update README

Update the README Todos section:
- Check off `[ ] Support HTTP 2` → `[x] Support HTTP 2`
- Add WebSocket to the roadmap (checked)
- Update performance table with HTTP/2 results

### Task 50: Commit and PR

Create the PR against `FusionAuth/java-http` using:
```bash
gh pr create --title "Add HTTP/2, WebSocket, and benchmark framework" --body "..."
```

---

## Summary

| Phase | Tasks | Estimated Duration |
|---|---|---|
| Phase 1: Foundation | Tasks 1-12 | ~1 week |
| Phase 2: Connection Handling | Tasks 13-17 | ~1 week |
| Phase 3: Stream Processing | Tasks 18-24 | ~1.5 weeks |
| Phase 4: Configuration & Integration | Tasks 25-26 | ~0.5 weeks |
| Phase 5: Testing | Tasks 27-30 | ~1.5 weeks |
| Phase 6: Load Testing & Optimization | Tasks 31-32 | ~1 week |
| Phase 7: Upgrade & WebSocket | Tasks 33-39 | ~2 weeks |
| Phase 8: Benchmark Framework | Tasks 40-46 | ~1 week |
| Final | Tasks 47-50 | ~0.5 weeks |
| **Total** | **50 tasks** | **~10 weeks** |
