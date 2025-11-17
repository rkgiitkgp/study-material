# HTTP Request End-to-End Cycle in Node.js
## Prompt Plan: From Networking Layer to Node.js Application Layer

### Objective
Comprehensive exploration of how Node.js handles HTTP requests from the network layer through the operating system kernel, event loop phases, and finally to the application code.

---

## 1. Networking Layer & Socket Connection

### TCP/IP Stack and HTTP

HTTP requests travel over TCP/IP, which provides reliable, ordered delivery. When you create an HTTP server in Node.js, here's what happens:

```javascript
const http = require('http');
const server = http.createServer((req, res) => {
  res.end('Hello World');
});
server.listen(3000);
```

When `server.listen(3000)` is called, Node.js doesn't immediately start accepting HTTP requests. Instead, it sets up a TCP socket that will receive HTTP data.

### Socket Creation and Port Binding

**Step 1: Socket Creation**
- Node.js (via libuv) calls the OS `socket()` system call to create a TCP socket file descriptor
- This creates a socket but doesn't bind it to a port yet

**Step 2: Port Binding**
- `bind()` system call binds the socket to port 3000 (or any specified port)
- The socket is now associated with port 3000 on the local machine
- At this point, the socket is in CLOSED state

**Step 3: Listening**
- `listen()` system call puts the socket into LISTEN state
- The socket is now ready to accept incoming connections
- Node.js also sets a backlog parameter (default ~511) - this is the queue size for pending connections

### Socket States During HTTP Connection Lifecycle

When a client makes an HTTP request, the socket goes through these states:

1. **LISTEN**: Server socket waiting for connections (your HTTP server)
2. **SYN_RECEIVED**: Client sends SYN packet, server receives it
3. **ESTABLISHED**: TCP handshake complete (3-way handshake: SYN → SYN-ACK → ACK)
4. **CLOSE_WAIT**: Client closed connection, server waiting to close
5. **CLOSED**: Connection fully closed

### Connection Queue and Backlog

When multiple HTTP requests arrive simultaneously:

- **Accepted connections**: Connections that completed the TCP handshake and are ready for HTTP processing
- **Backlog queue**: Connections in SYN_RECEIVED state waiting to complete handshake
- If backlog is full, new connection attempts are rejected (ECONNREFUSED)

The backlog size affects how many pending connections your server can handle. Node.js uses `server.listen(port, hostname, backlog)` where backlog defaults to 511.

### HTTP Over TCP: What Actually Happens

HTTP is just text sent over TCP. When a client sends:
```
GET /api/users HTTP/1.1
Host: localhost:3000
```

This text is sent as TCP packets. The TCP layer ensures:
- All packets arrive (retransmission if lost)
- Packets arrive in order
- No duplicate packets

Node.js receives these TCP packets and reconstructs the HTTP request text from them.

---

## 2. Operating System Kernel Interaction

### System Calls for HTTP Server Setup

When Node.js starts an HTTP server, it makes these system calls (through libuv):

1. **`socket()`**: Creates a TCP socket file descriptor
2. **`bind()`**: Binds socket to IP address and port
3. **`listen()`**: Marks socket as passive (listening for connections)
4. **`accept()`**: Accepts incoming connections (called when client connects)

These are blocking system calls by default, but libuv makes them non-blocking.

### Non-blocking I/O for HTTP

The key to Node.js handling thousands of HTTP connections is non-blocking I/O:

**Traditional blocking approach (bad for Node.js):**
```c
// This would block the thread
int client_fd = accept(server_fd, ...);  // Waits here until connection arrives
read(client_fd, buffer, ...);            // Waits here until data arrives
```

**Node.js non-blocking approach:**
```c
// Set socket to non-blocking mode
fcntl(socket_fd, F_SETFL, O_NONBLOCK);
int client_fd = accept(server_fd, ...);  // Returns immediately (EWOULDBLOCK if no connection)
```

When `accept()` or `read()` would block, they return immediately with an error code (like EAGAIN or EWOULDBLOCK) meaning "try again later when data is ready."

### I/O Multiplexing: epoll, kqueue, IOCP

Node.js can't poll thousands of sockets individually (too slow). Instead, it uses OS-level I/O multiplexing:

**Linux - epoll:**
- `epoll_create()`: Creates an epoll instance
- `epoll_ctl()`: Registers sockets to watch
- `epoll_wait()`: Blocks until ANY registered socket has data (efficient!)

**macOS/BSD - kqueue:**
- Similar to epoll but uses different API
- `kqueue()`: Creates event queue
- `kevent()`: Registers events and waits for them

**Windows - IOCP (I/O Completion Ports):**
- Different model: completion-based rather than readiness-based
- Operations are submitted, and completion callbacks fire when done

### How Event Notification Works for HTTP

Here's the flow when an HTTP request arrives:

1. **Client sends TCP packet** → OS kernel receives it
2. **Kernel buffers data** → Stores in TCP receive buffer (SO_RCVBUF)
3. **Kernel marks socket as readable** → Updates internal data structures
4. **libuv calls epoll_wait()** → OS returns "socket X has data ready"
5. **libuv reads from socket** → Non-blocking read, gets HTTP data
6. **libuv triggers callback** → Node.js event loop processes the HTTP request

The magic is that `epoll_wait()` can watch thousands of sockets simultaneously and efficiently return only the ones with data ready.

### Kernel Buffering for HTTP

**TCP Receive Buffer (SO_RCVBUF):**
- Default: ~200KB on Linux
- Stores incoming HTTP request data until Node.js reads it
- If buffer fills up, TCP flow control kicks in (client slows down sending)

**TCP Send Buffer (SO_SNDBUF):**
- Default: ~200KB on Linux
- Stores outgoing HTTP response data until OS sends it
- If buffer fills up, `res.write()` may block or return false (backpressure)

**Why buffering matters:**
- Network speed ≠ application read speed
- Buffers smooth out the difference
- Too small = frequent blocking, too large = memory waste

### The Accept Loop

When your HTTP server is running, libuv continuously:

1. Calls `epoll_wait()` to wait for events
2. When a new connection arrives, `accept()` is called
3. New socket is added to epoll watch list
4. When HTTP data arrives on any socket, `read()` is called
5. Data is passed to Node.js HTTP parser

This all happens in the event loop's poll phase, which we'll cover next.

---

## 3. Node.js Event Loop Phases (HTTP-Specific)

The Node.js event loop has multiple phases, but for HTTP requests, two phases are most critical: **Poll** (where HTTP requests are processed) and **Close** (where connections are cleaned up).

### 3.1 Poll Phase (HTTP Request Processing)

The poll phase is where all HTTP I/O happens. This is the heart of Node.js HTTP processing.

#### What Happens in Poll Phase

**Step 1: Check for I/O Events**
```javascript
// Simplified pseudocode of what libuv does
while (poll_phase_active) {
  // Wait for I/O events (epoll_wait, kqueue, etc.)
  int num_events = epoll_wait(epoll_fd, events, MAX_EVENTS, timeout);
  
  for (each event in events) {
    if (event.type == NEW_CONNECTION) {
      accept_new_http_connection();
    } else if (event.type == DATA_READY) {
      read_http_data_from_socket();
    }
  }
}
```

**Step 2: Read HTTP Data from Socket**
- libuv performs non-blocking `read()` on sockets with data
- HTTP request bytes are read from kernel buffer into Node.js
- Data is passed to the HTTP parser (llhttp)

**Step 3: HTTP Request Parsing**
The llhttp parser processes data incrementally:

```javascript
// What happens internally (simplified)
parser.onHeadersComplete = function(version, method, url, headers) {
  // Request line and headers are complete
  // Create req object (IncomingMessage)
  const req = new IncomingMessage(socket);
  req.method = method;
  req.url = url;
  req.headers = headers;
  
  // NOW your handler is called!
  server.emit('request', req, res);
};

parser.onBody = function(chunk) {
  // Body data arrives incrementally
  req.emit('data', chunk);
};

parser.onMessageComplete = function() {
  // Entire HTTP request is complete
  req.emit('end');
};
```

**Step 4: Invoke HTTP Request Handler**
- Once headers are parsed, your `(req, res) => {...}` handler is called
- This happens **synchronously** during the poll phase
- If your handler does async work (database, file I/O), it returns control to event loop

#### Important: Poll Phase Execution Model

```javascript
const server = http.createServer((req, res) => {
  console.log('Handler 1');  // Runs in poll phase
  setTimeout(() => {
    console.log('Handler 2');  // Runs in timers phase (later)
  }, 0);
  res.end('Done');
});

// When HTTP request arrives:
// 1. Poll phase: Handler 1 executes, setTimeout scheduled
// 2. Poll phase: res.end() sends response
// 3. Event loop continues to next phases
// 4. Later: Timers phase executes Handler 2
```

**Key Point**: Your HTTP handler runs synchronously in the poll phase. If it takes 100ms, no other HTTP requests are processed during that time (unless they're on different connections that already have data buffered).

#### Poll Phase Timeout

The poll phase has a timeout:
- If there are pending timers, timeout = time until next timer
- If no timers, timeout = infinity (waits until I/O event)
- This ensures timers execute on time while still processing I/O

### 3.2 Close Phase (HTTP Connection Cleanup)

After an HTTP request/response completes, the connection may close. The close phase handles cleanup.

#### When Close Phase Executes

The close phase runs when:
1. HTTP response is fully sent (`res.end()` called and data flushed)
2. Client closes connection (sends FIN packet)
3. Server closes connection (timeout, error, etc.)

#### What Happens in Close Phase

```javascript
// Simplified flow
socket.on('close', () => {
  // This callback runs in CLOSE phase
  
  // 1. Remove socket from epoll watch list
  epoll_ctl(epoll_fd, EPOLL_CTL_DEL, socket_fd);
  
  // 2. Close file descriptor
  close(socket_fd);
  
  // 3. Free memory (socket object, buffers)
  // 4. Trigger any 'close' event handlers user registered
});
```

#### TCP Connection Termination

When an HTTP connection closes:

1. **Client sends FIN** → Server receives it, socket enters CLOSE_WAIT state
2. **Server sends ACK** → Acknowledges FIN
3. **Server sends FIN** → When server is done sending response
4. **Client sends ACK** → Connection fully closed
5. **Close phase callbacks execute** → Cleanup happens

#### Resource Cleanup

The close phase ensures:
- Socket file descriptors are closed (prevent file descriptor leaks)
- Memory buffers are freed
- Socket removed from epoll/kqueue watch lists
- Any user-registered 'close' handlers are called

**Important**: If you don't properly close HTTP connections, you'll leak file descriptors and memory. Always call `res.end()` or `req.destroy()`.

---

## 4. HTTP Streams Processing

HTTP requests and responses in Node.js are streams. This allows handling large payloads without loading everything into memory.

### HTTP Request Stream (IncomingMessage)

The `req` object is a Readable stream. HTTP request body data flows through it.

#### How Request Body Streaming Works

```javascript
const server = http.createServer((req, res) => {
  // req is a Readable stream
  let body = '';
  
  // Data arrives in chunks as it's read from socket
  req.on('data', (chunk) => {
    body += chunk.toString();
    console.log('Received chunk:', chunk.length, 'bytes');
  });
  
  // All data received
  req.on('end', () => {
    console.log('Complete body:', body);
    res.end('OK');
  });
});
```

**What happens internally:**
1. HTTP parser (llhttp) receives body chunks from socket
2. Each chunk triggers `req.emit('data', chunk)`
3. When parser finishes, `req.emit('end')` is called
4. Your handlers process the streamed data

#### Request Stream States

- **Flowing mode**: Data automatically flows to handlers (default for HTTP)
- **Paused mode**: You call `req.pause()` to stop data flow
- **Ended**: Stream is done, no more data (`req.readableEnded === true`)

### HTTP Response Stream (ServerResponse)

The `res` object is a Writable stream. You write data to it, and it sends it over the network.

#### Writing HTTP Responses

```javascript
const server = http.createServer((req, res) => {
  // Write headers first
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  
  // Write body in chunks
  res.write('Hello');
  res.write(' ');
  res.write('World');
  
  // End the response (sends final chunk and closes connection)
  res.end();
});
```

**What happens internally:**
1. `res.write()` adds data to internal buffer
2. If socket is ready, data is written immediately (non-blocking)
3. If socket buffer is full, data waits in Node.js buffer
4. `res.end()` sends final chunk and signals end of response

### Backpressure in HTTP Streams

Backpressure occurs when the receiver (client) can't keep up with the sender (server).

#### How Backpressure Works

```javascript
const server = http.createServer((req, res) => {
  // Writing faster than client can receive
  const canContinue = res.write('some data');
  
  if (!canContinue) {
    // Buffer is full! Wait for drain event
    res.once('drain', () => {
      console.log('Buffer drained, can write more');
      // Continue writing...
    });
  }
});
```

**Backpressure flow:**
1. Server writes data → Goes to socket send buffer (SO_SNDBUF)
2. Socket buffer fills up → OS signals "can't send more"
3. Node.js buffer fills up → `res.write()` returns `false`
4. Client reads data → Socket buffer drains
5. OS signals ready → `res.emit('drain')` fires
6. Server can write more

**Why it matters:**
- Prevents server from overwhelming slow clients
- Prevents unbounded memory growth
- Automatic flow control built into TCP

### Chunked Transfer Encoding

For responses where size isn't known upfront, HTTP uses chunked encoding:

```javascript
const server = http.createServer((req, res) => {
  res.writeHead(200, {
    'Transfer-Encoding': 'chunked'  // Node.js sets this automatically if needed
  });
  
  // Write chunks
  res.write('chunk1');
  res.write('chunk2');
  res.end();  // Sends final chunk (size 0)
});
```

**How chunked encoding works:**
- Each chunk: `HEX_SIZE\r\nDATA\r\n`
- Example: `5\r\nHello\r\n` (5 bytes, then "Hello")
- Final chunk: `0\r\n\r\n` (size 0, end of response)

Node.js handles this automatically when you don't set `Content-Length`.

### HTTP Stream Events

Both request and response streams emit events:

**Request (Readable) events:**
- `'data'`: Chunk of request body received
- `'end'`: Request body complete
- `'error'`: Error reading request
- `'close'`: Underlying socket closed

**Response (Writable) events:**
- `'drain'`: Buffer drained, can write more
- `'finish'`: All data flushed to socket
- `'error'`: Error writing response
- `'close'`: Connection closed

### Stream Piping for HTTP

You can pipe streams directly to HTTP responses:

```javascript
const fs = require('fs');

const server = http.createServer((req, res) => {
  // Pipe file directly to HTTP response
  const fileStream = fs.createReadStream('large-file.txt');
  fileStream.pipe(res);  // Handles backpressure automatically!
});
```

**What happens:**
- File data flows through pipe to response
- If response buffer fills, file stream pauses automatically
- When buffer drains, file stream resumes
- Backpressure handled automatically by streams

This is efficient for large files - data flows without loading entire file into memory.

---

## 5. HTTP Application Layer (Developer Perspective)

This is where your code lives. Understanding how the HTTP module works helps you write better handlers.

### HTTP Module: `http.createServer()` Internals

When you call `http.createServer()`, here's what happens:

```javascript
const http = require('http');

// This creates a Server instance
const server = http.createServer((req, res) => {
  // Your handler
});

// Internally, this is roughly:
class Server extends EventEmitter {
  constructor(requestListener) {
    super();
    this.on('request', requestListener);  // Store your handler
  }
  
  listen(port) {
    // Create TCP socket, bind, listen (via libuv)
    // Start accepting connections
    // When HTTP request arrives, emit 'request' event
  }
}
```

**Key points:**
- Server is an EventEmitter
- Your handler is registered as a 'request' event listener
- When HTTP request is parsed, server emits 'request' with `(req, res)`
- Your handler is called synchronously in the poll phase

### HTTP Request Object (`req` - IncomingMessage)

The `req` object contains all information about the incoming HTTP request.

#### Key Properties

```javascript
const server = http.createServer((req, res) => {
  // HTTP method (GET, POST, etc.)
  console.log(req.method);  // 'GET'
  
  // Request URL path
  console.log(req.url);  // '/api/users?id=123'
  
  // HTTP version
  console.log(req.httpVersion);  // '1.1'
  
  // Request headers (lowercase keys)
  console.log(req.headers);  // { 'content-type': 'application/json', ... }
  
  // Raw headers (preserves case)
  console.log(req.rawHeaders);  // ['Content-Type', 'application/json', ...]
  
  // Socket (underlying TCP connection)
  console.log(req.socket);  // Socket object
});
```

#### Request Body Reading

```javascript
// Method 1: Stream (for large bodies)
req.on('data', (chunk) => {
  // Process chunk
});

req.on('end', () => {
  // Body complete
});

// Method 2: Collect entire body
let body = '';
req.on('data', chunk => body += chunk);
req.on('end', () => {
  const data = JSON.parse(body);
});

// Method 3: Using helper (if available)
// Note: Node.js doesn't have built-in body parser
// Frameworks like Express add this
```

#### Request Stream Methods

```javascript
req.pause();   // Pause data flow
req.resume();  // Resume data flow
req.destroy(); // Destroy the request stream
```

### HTTP Response Object (`res` - ServerResponse)

The `res` object is how you send data back to the client.

#### Setting Headers

```javascript
// Method 1: Individual headers
res.setHeader('Content-Type', 'application/json');
res.setHeader('X-Custom-Header', 'value');

// Method 2: Multiple headers at once
res.writeHead(200, {
  'Content-Type': 'application/json',
  'X-Custom-Header': 'value'
});

// Method 3: Status code + headers
res.statusCode = 200;
res.setHeader('Content-Type', 'text/plain');
```

#### Writing Response Body

```javascript
// Method 1: Write in chunks
res.write('Hello');
res.write(' ');
res.write('World');
res.end();  // Must call end() to finish

// Method 2: Write everything at once
res.end('Hello World');

// Method 3: Write JSON
res.setHeader('Content-Type', 'application/json');
res.end(JSON.stringify({ message: 'Hello' }));
```

#### Response Status Codes

```javascript
res.statusCode = 404;
res.statusMessage = 'Not Found';  // Optional
res.end('Page not found');

// Or use writeHead
res.writeHead(404, 'Not Found');
res.end('Page not found');
```

### HTTP Middleware Pattern (Express Example)

Frameworks like Express build on top of Node.js HTTP module:

```javascript
const express = require('express');
const app = express();

// Middleware 1: Logging
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url}`);
  next();  // Pass to next middleware
});

// Middleware 2: Body parsing
app.use(express.json());

// Route handler
app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

// Express wraps http.createServer()
app.listen(3000);
```

**How middleware works:**
1. Express creates HTTP server internally
2. Each middleware is a function `(req, res, next) => {}`
3. Middleware chain executes in order
4. `next()` calls next middleware
5. Final handler sends response

### Async HTTP Handlers and Event Loop

Your HTTP handlers can be async, but understand how they interact with the event loop:

```javascript
const server = http.createServer(async (req, res) => {
  // This runs synchronously in poll phase
  console.log('Handler started');
  
  // This returns a Promise immediately
  const data = await fetch('https://api.example.com/data');
  
  // Control returns to event loop while waiting
  // Other HTTP requests can be processed!
  
  // When Promise resolves, this continues
  res.json(data);
});
```

**Execution flow:**
1. Poll phase: Handler starts, `await` is hit
2. Handler returns Promise, control back to event loop
3. Other HTTP requests can be processed
4. When Promise resolves, handler continues (in next tick)
5. Response is sent

**Important**: If you do CPU-intensive work synchronously, it blocks the event loop:

```javascript
// BAD: Blocks event loop
const server = http.createServer((req, res) => {
  // This blocks for 100ms - no other requests processed!
  const result = heavyComputation();
  res.end(result);
});

// GOOD: Use async or worker threads
const server = http.createServer(async (req, res) => {
  const result = await heavyComputationAsync();
  res.end(result);
});
```

### HTTP Error Handling

Errors can occur at different levels:

```javascript
const server = http.createServer((req, res) => {
  // Handle request errors
  req.on('error', (err) => {
    console.error('Request error:', err);
    res.statusCode = 400;
    res.end('Bad Request');
  });
  
  // Handle response errors
  res.on('error', (err) => {
    console.error('Response error:', err);
    // Response might already be sent
  });
  
  try {
    // Your handler code
    throw new Error('Something went wrong');
  } catch (err) {
    // Handle synchronous errors
    res.statusCode = 500;
    res.end('Internal Server Error');
  }
});

// Handle server-level errors
server.on('error', (err) => {
  console.error('Server error:', err);
});

// Handle client errors (malformed requests, etc.)
server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
```

**Error propagation:**
- Request parsing errors → `clientError` event
- Handler errors → Should be caught in try/catch
- Socket errors → `error` event on req/res
- Server errors → `error` event on server

**Best practice**: Always handle errors, or they'll crash your server!

---

## 6. Complete Request Lifecycle Flow

Here's the detailed step-by-step execution of a single HTTP request from network packet to response.

### Phase 1: Network and OS Layer

**Step 1: Network Packet Arrives**
- Client sends TCP packet containing HTTP request data
- Packet travels through network (routers, switches)
- Arrives at server's network interface card (NIC)
- OS kernel's network stack receives the packet

**Step 2: TCP Processing**
- Kernel validates TCP packet (checksum, sequence number)
- If packet is part of existing connection, adds to connection's receive buffer
- If new connection, initiates TCP handshake (SYN → SYN-ACK → ACK)
- Socket enters ESTABLISHED state

**Step 3: Kernel Buffering**
- HTTP request bytes stored in TCP receive buffer (SO_RCVBUF)
- Buffer size typically ~200KB (configurable)
- If buffer fills, TCP flow control slows down client sending

**Step 4: Event Notification**
- Kernel marks socket as "readable" in its internal data structures
- libuv has registered this socket with epoll/kqueue/IOCP
- When libuv calls `epoll_wait()`, OS returns this socket as ready
- This happens in the event loop's poll phase

### Phase 2: Event Loop Poll Phase

**Step 5: Poll Phase Activation**
- Event loop enters poll phase
- libuv calls `epoll_wait()` (or equivalent) with timeout
- OS returns list of sockets with data ready
- For each socket, libuv processes events

**Step 6: Socket Read Operation**
- libuv calls non-blocking `read()` on socket
- Data copied from kernel buffer to Node.js buffer
- Read returns number of bytes read (or EAGAIN if no data)
- Multiple reads may be needed for large requests

**Step 7: HTTP Parsing Begins**
- Raw bytes passed to llhttp parser
- Parser is state machine that processes HTTP incrementally
- First, parses request line: `GET /api/users HTTP/1.1\r\n`
- Extracts: method='GET', url='/api/users', version='1.1'

**Step 8: Header Parsing**
- Parser continues reading headers line by line
- Each header: `Header-Name: value\r\n`
- Headers end with empty line `\r\n\r\n`
- When headers complete, parser fires `onHeadersComplete` callback

**Step 9: Request Object Creation**
- Node.js creates `IncomingMessage` object (req)
- Sets properties: `req.method`, `req.url`, `req.headers`, `req.httpVersion`
- Creates `ServerResponse` object (res)
- Both objects are streams (Readable and Writable)

**Step 10: Handler Invocation**
- Server emits 'request' event with `(req, res)`
- Your handler function is called **synchronously** in poll phase
- Handler receives req and res objects
- If handler is async and hits `await`, control returns to event loop

### Phase 3: Application Processing

**Step 11: Handler Execution**
- Your code processes the request
- May read request body: `req.on('data', ...)` or `await` body parsing
- May perform async operations: database queries, API calls, file I/O
- While waiting for async operations, other HTTP requests can be processed

**Step 12: Response Preparation**
- Handler sets response headers: `res.setHeader()` or `res.writeHead()`
- Handler writes response body: `res.write()` or `res.end()`
- Response data goes into Node.js internal buffer

### Phase 4: Response Sending

**Step 13: Response Writing**
- `res.write()` or `res.end()` adds data to response buffer
- If socket is ready, libuv writes data immediately (non-blocking)
- If socket buffer is full, data waits in Node.js buffer (backpressure)
- When buffer drains, 'drain' event fires

**Step 14: Socket Write Operation**
- libuv calls non-blocking `write()` on socket
- Data copied from Node.js buffer to kernel send buffer (SO_SNDBUF)
- Write may be partial if buffer is full
- Remaining data waits for next write opportunity

**Step 15: Kernel Network Stack**
- Kernel's TCP stack processes send buffer
- TCP segments data into packets
- Adds TCP headers (sequence numbers, checksums)
- Sends packets to network interface
- Handles retransmission if packets are lost

**Step 16: Network Transmission**
- Packets travel over network to client
- Client receives packets, reassembles TCP stream
- Client's HTTP client parses response
- Client processes response (browser renders, API client parses JSON, etc.)

### Phase 5: Connection Cleanup

**Step 17: Response Completion**
- When `res.end()` is called and data flushed, response is complete
- If `Connection: close` header or HTTP/1.0, connection closes
- If HTTP/1.1 with `Connection: keep-alive`, connection stays open for reuse

**Step 18: Connection Closure (if closing)**
- Client or server sends FIN packet
- TCP connection enters CLOSE_WAIT or TIME_WAIT state
- Four-way handshake completes: FIN → ACK → FIN → ACK
- Socket enters CLOSED state

**Step 19: Close Phase Cleanup**
- Event loop enters close phase
- Socket 'close' event handlers execute
- libuv removes socket from epoll/kqueue watch list
- File descriptor is closed (`close()` system call)
- Memory buffers are freed
- Socket object is garbage collected

### Timeline Example

For a typical HTTP GET request:

```
0ms:    Client sends TCP packet
1ms:    Kernel receives, buffers data
2ms:    epoll_wait() returns socket as ready
3ms:    Poll phase: libuv reads data
4ms:    HTTP parser processes request line + headers
5ms:    Handler invoked (synchronously)
6ms:    Handler executes (quick, no async work)
7ms:    res.end() called, response written
8ms:    Data sent to kernel send buffer
9ms:    Kernel sends TCP packets
10ms:   Client receives response
11ms:   Connection closes (if not keep-alive)
12ms:   Close phase: cleanup
```

For async handler:
```
0-5ms:  Same as above
6ms:    Handler starts, hits await database.query()
7ms:    Control returns to event loop (other requests can process)
50ms:   Database query completes
51ms:   Handler continues, sends response
52ms:   Response sent, connection closes
```

This is why Node.js can handle thousands of concurrent connections - while one request waits for I/O, others are processed.

---

## 7. HTTP-Specific Key Concepts

### HTTP Event-Driven Architecture

Node.js handles multiple HTTP connections concurrently using an event-driven model:

**Traditional Multi-Threaded Model (Apache, old servers):**
- Each HTTP connection = 1 thread
- 1000 connections = 1000 threads
- Threads consume memory (~2MB each)
- Context switching overhead
- Limited scalability

**Node.js Event-Driven Model:**
- All HTTP connections = 1 main thread
- Event loop processes all connections
- I/O operations are non-blocking
- When one request waits for I/O, others are processed
- Can handle 10,000+ connections on single thread

**How it works:**
```javascript
// Simplified concept
while (true) {
  // Check all sockets for data
  const readySockets = epoll_wait(all_sockets);
  
  for (socket of readySockets) {
    // Process HTTP request on this socket
    processHttpRequest(socket);
  }
  
  // Process other events (timers, etc.)
  processOtherEvents();
}
```

### Single-Threaded HTTP Model

Node.js uses one main thread for JavaScript execution, but handles thousands of HTTP connections:

**Main Thread Responsibilities:**
- JavaScript execution (your HTTP handlers)
- Event loop coordination
- HTTP parsing (llhttp)
- Response generation

**Worker Threads (Optional):**
- CPU-intensive tasks (not HTTP I/O)
- File I/O operations (can use thread pool)
- Crypto operations

**libuv Thread Pool:**
- Default: 4 threads
- Used for: file system operations, DNS lookups
- **NOT used for**: Network I/O (HTTP uses non-blocking I/O)

**Why Single-Threaded Works:**
- HTTP I/O is mostly waiting (network latency)
- While waiting, thread can process other requests
- CPU-bound work should be offloaded to worker threads

### Libuv for HTTP

libuv is the C++ library that provides the event loop and handles all I/O:

**What libuv does for HTTP:**
1. **Socket Management**: Creates, binds, listens on sockets
2. **I/O Multiplexing**: Uses epoll/kqueue/IOCP to watch multiple sockets
3. **Non-blocking I/O**: Makes all socket operations non-blocking
4. **Event Loop**: Coordinates when to process which events
5. **Thread Pool**: Manages worker threads for file I/O

**libuv Event Loop Structure:**
```
┌─────────────────┐
│   Timers        │  setTimeout, setInterval
├─────────────────┤
│   Pending       │  I/O callbacks
├─────────────────┤
│   Idle/Prepare   │  Internal use
├─────────────────┤
│   Poll          │  ← HTTP requests processed here!
├─────────────────┤
│   Check         │  setImmediate
├─────────────────┤
│   Close         │  ← HTTP cleanup here!
└─────────────────┘
```

**HTTP-specific libuv operations:**
- `uv_tcp_bind()`: Bind socket to port
- `uv_listen()`: Start accepting connections
- `uv_read_start()`: Start reading from socket
- `uv_write()`: Write data to socket
- `uv_close()`: Close socket and cleanup

### HTTP Performance Considerations

#### Throughput
- **Requests per second**: How many HTTP requests can be processed
- Limited by: CPU, memory, network bandwidth
- Typical: 10,000-50,000 req/s on modern hardware
- Bottleneck is usually application logic, not Node.js

#### Latency
- **Time to first byte**: How quickly server responds
- **Total response time**: End-to-end request/response
- Factors: network latency, processing time, I/O wait time
- Keep handlers fast, use async I/O

#### Connection Limits

**OS Limits:**
- File descriptors: Default ~1024, can be increased
- `ulimit -n` to check/change
- Each HTTP connection uses 1 file descriptor

**Node.js Limits:**
- No hard limit, but memory is constraint
- Each connection uses ~2-4KB memory (buffers)
- 10,000 connections ≈ 20-40MB memory

**C10K Problem:**
- Challenge: Handle 10,000 concurrent connections
- Node.js solves this with event-driven architecture
- Can handle 100K+ connections with proper tuning

#### Performance Optimization Tips

```javascript
// 1. Keep handlers async
const server = http.createServer(async (req, res) => {
  const data = await db.query();  // Non-blocking
  res.json(data);
});

// 2. Use connection pooling
// Keep-alive connections reuse sockets

// 3. Stream large responses
const server = http.createServer((req, res) => {
  fs.createReadStream('large-file.txt').pipe(res);
});

// 4. Avoid blocking operations
// Don't do CPU-intensive work in handlers
```

### HTTP Buffer Management

**Request Buffering:**
- TCP receive buffer (kernel): ~200KB
- Node.js internal buffer: ~16KB per connection
- If buffer fills, backpressure slows client

**Response Buffering:**
- Node.js internal buffer: ~16KB per connection
- TCP send buffer (kernel): ~200KB
- If buffer fills, `res.write()` returns false

**Memory Usage:**
- Small buffers = more system calls (slower)
- Large buffers = more memory usage
- Node.js balances automatically

**Buffer Configuration:**
```javascript
// Increase socket buffer sizes (if needed)
server.on('connection', (socket) => {
  socket.setReadableHighWaterMark(64 * 1024);  // 64KB
  socket.setWritableHighWaterMark(64 * 1024);
});
```

### HTTP Debugging Tools

#### async_hooks (Node.js Built-in)
```javascript
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    if (type === 'HTTPCLIENTREQUEST' || type === 'HTTPINCOMINGMESSAGE') {
      console.log(`HTTP ${type}`, asyncId);
    }
  }
});

hook.enable();
```

#### perf (Linux Performance Profiler)
```bash
# Record HTTP server
perf record -g node server.js

# Analyze
perf report
```

#### dtrace (macOS/Solaris)
```bash
# Trace HTTP requests
dtrace -n 'node*:::http-server-request { printf("%s", copyinstr(arg0)); }'
```

#### Node.js Built-in Tracing
```bash
# Trace HTTP events
node --trace-event-categories=node.http server.js

# View with Chrome DevTools
# chrome://tracing
```

#### Custom Logging
```javascript
const server = http.createServer((req, res) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} ${res.statusCode} ${duration}ms`);
  });
  
  // ... handler
});
```

These tools help you understand where time is spent in the HTTP request lifecycle and identify bottlenecks.

---

## 8. HTTP Request/Response Practical Examples

### Example 1: Simple HTTP Request Flow

Let's trace a simple GET request through all layers:

```javascript
// server.js
const http = require('http');

const server = http.createServer((req, res) => {
  console.log(`[Handler] ${req.method} ${req.url}`);
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello World');
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');
});

// Add event listeners to trace flow
server.on('connection', (socket) => {
  console.log('[Connection] New TCP connection');
  
  socket.on('data', (chunk) => {
    console.log(`[Socket] Received ${chunk.length} bytes`);
  });
  
  socket.on('close', () => {
    console.log('[Connection] TCP connection closed');
  });
});

server.on('request', (req, res) => {
  console.log('[Server] Request event emitted');
  
  res.on('finish', () => {
    console.log('[Response] Response finished');
  });
  
  res.on('close', () => {
    console.log('[Response] Response closed');
  });
});
```

**Execution flow when client sends `GET / HTTP/1.1`:**
```
[Connection] New TCP connection          ← TCP handshake complete
[Socket] Received 78 bytes               ← HTTP request data arrives
[Server] Request event emitted           ← Headers parsed, handler called
[Handler] GET /                           ← Your handler executes
[Response] Response finished             ← res.end() called, data sent
[Connection] TCP connection closed       ← Connection closes
[Response] Response closed                ← Close phase cleanup
```

### Example 2: Async Handler with Database Query

```javascript
const http = require('http');

// Simulate database query
function queryDatabase() {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve([{ id: 1, name: 'John' }]);
    }, 100);
  });
}

const server = http.createServer(async (req, res) => {
  console.log('[Handler] Started');
  
  // This await returns control to event loop
  const data = await queryDatabase();
  
  console.log('[Handler] Database query complete');
  res.json(data);
});

server.listen(3000);
```

**Execution timeline:**
```
0ms:   [Handler] Started                    ← Poll phase: handler invoked
1ms:   await queryDatabase()                ← Promise created, control returns
2ms:   Event loop processes other requests  ← Other HTTP requests handled here
100ms: Promise resolves                     ← Database query completes
101ms: [Handler] Database query complete     ← Handler continues
102ms: Response sent                          ← res.json() sends data
```

**Key insight**: While waiting for database, other HTTP requests are processed!

### Example 3: Streaming Large Response

```javascript
const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  console.log('[Handler] Streaming file');
  
  const fileStream = fs.createReadStream('large-file.txt');
  
  fileStream.on('data', (chunk) => {
    console.log(`[Stream] Sending chunk: ${chunk.length} bytes`);
  });
  
  fileStream.on('end', () => {
    console.log('[Stream] File stream ended');
  });
  
  // Pipe handles backpressure automatically
  fileStream.pipe(res);
});

server.listen(3000);
```

**What happens:**
1. File stream starts reading
2. Data chunks flow to response stream
3. If response buffer fills, file stream pauses (backpressure)
4. When buffer drains, file stream resumes
5. All handled automatically by streams

### Example 4: Request Body Parsing

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.method === 'POST') {
    let body = '';
    
    // Data arrives in chunks
    req.on('data', (chunk) => {
      console.log(`[Request] Received chunk: ${chunk.length} bytes`);
      body += chunk.toString();
    });
    
    // All data received
    req.on('end', () => {
      console.log(`[Request] Complete body: ${body.length} bytes`);
      try {
        const data = JSON.parse(body);
        res.json({ received: data });
      } catch (err) {
        res.statusCode = 400;
        res.end('Invalid JSON');
      }
    });
  } else {
    res.end('Send POST request');
  }
});

server.listen(3000);
```

**Flow for POST request:**
```
[Request] Received chunk: 1024 bytes    ← First chunk of body
[Request] Received chunk: 512 bytes     ← Second chunk
[Request] Complete body: 1536 bytes     ← All chunks received
[Handler] Processing JSON               ← Handler processes data
[Response] Sending response             ← Response sent
```

### Example 5: Error Handling

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // Handle request errors
  req.on('error', (err) => {
    console.error('[Request Error]', err);
    res.statusCode = 400;
    res.end('Bad Request');
  });
  
  // Handle response errors
  res.on('error', (err) => {
    console.error('[Response Error]', err);
  });
  
  try {
    // Simulate error
    if (req.url === '/error') {
      throw new Error('Something went wrong');
    }
    res.end('OK');
  } catch (err) {
    console.error('[Handler Error]', err);
    res.statusCode = 500;
    res.end('Internal Server Error');
  }
});

// Handle server-level errors
server.on('error', (err) => {
  console.error('[Server Error]', err);
});

// Handle malformed requests
server.on('clientError', (err, socket) => {
  console.error('[Client Error]', err);
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});

server.listen(3000);
```

### Example 6: Connection Keep-Alive

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  // HTTP/1.1 keeps connection alive by default
  res.writeHead(200, {
    'Content-Type': 'text/plain',
    'Connection': 'keep-alive'  // Explicit (default in HTTP/1.1)
  });
  
  res.end('Response 1');
  
  // Same connection can be reused for next request
});

server.listen(3000);
```

**Keep-alive behavior:**
- First request: TCP connection established
- Response sent, connection stays open
- Second request: Reuses same TCP connection
- Faster (no TCP handshake)
- Connection closes after timeout or explicit close

### Example 7: Tracing with Custom Logging

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  const requestId = Math.random().toString(36).substr(2, 9);
  const startTime = process.hrtime.bigint();
  
  console.log(`[${requestId}] Request started: ${req.method} ${req.url}`);
  
  // Log when headers are received
  console.log(`[${requestId}] Headers:`, Object.keys(req.headers));
  
  // Log response
  res.on('finish', () => {
    const duration = Number(process.hrtime.bigint() - startTime) / 1_000_000;
    console.log(`[${requestId}] Response sent: ${res.statusCode} in ${duration.toFixed(2)}ms`);
  });
  
  res.end('OK');
});

server.listen(3000);
```

**Output example:**
```
[abc123] Request started: GET /api/users
[abc123] Headers: [ 'host', 'user-agent', 'accept' ]
[abc123] Response sent: 200 in 2.45ms
```

### Example 8: Understanding Backpressure

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  let chunksWritten = 0;
  
  function writeChunk() {
    const chunk = 'x'.repeat(1000);  // 1KB chunk
    const canContinue = res.write(chunk);
    chunksWritten++;
    
    if (!canContinue) {
      console.log(`[Backpressure] Buffer full after ${chunksWritten} chunks`);
      // Wait for drain event
      res.once('drain', () => {
        console.log('[Backpressure] Buffer drained, continuing');
        writeChunk();  // Continue writing
      });
    } else {
      // Continue writing if buffer has space
      if (chunksWritten < 1000) {
        setImmediate(writeChunk);  // Use setImmediate to avoid blocking
      } else {
        res.end();
        console.log(`[Complete] Wrote ${chunksWritten} chunks`);
      }
    }
  }
  
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  writeChunk();
});

server.listen(3000);
```

This example demonstrates how backpressure works - when the buffer fills, writing pauses until it drains.

### Common Pitfalls and Best Practices

**Pitfall 1: Blocking the Event Loop**
```javascript
// BAD: Blocks event loop
const server = http.createServer((req, res) => {
  // This blocks for 100ms - no other requests processed!
  const result = heavyComputation();
  res.end(result);
});

// GOOD: Use async or worker threads
const server = http.createServer(async (req, res) => {
  const result = await heavyComputationAsync();
  res.end(result);
});
```

**Pitfall 2: Not Handling Errors**
```javascript
// BAD: Unhandled errors crash server
const server = http.createServer(async (req, res) => {
  const data = await db.query();  // Might throw!
  res.json(data);
});

// GOOD: Always handle errors
const server = http.createServer(async (req, res) => {
  try {
    const data = await db.query();
    res.json(data);
  } catch (err) {
    res.statusCode = 500;
    res.end('Internal Server Error');
  }
});
```

**Pitfall 3: Memory Leaks with Event Listeners**
```javascript
// BAD: Event listeners accumulate
const server = http.createServer((req, res) => {
  req.on('data', () => {});  // Listener never removed
});

// GOOD: Use once() or remove listeners
const server = http.createServer((req, res) => {
  req.once('data', () => {});  // Auto-removed after firing
});
```

**Best Practice: Always Call res.end()**
```javascript
// BAD: Response never ends
const server = http.createServer((req, res) => {
  res.write('Hello');
  // Missing res.end() - connection hangs!
});

// GOOD: Always end response
const server = http.createServer((req, res) => {
  res.write('Hello');
  res.end();  // Always call end()
});
```

These examples demonstrate the complete HTTP request/response cycle and common patterns you'll encounter when building HTTP servers in Node.js.

---

## HTTP Request/Response Questions to Answer
- How does Node.js handle thousands of concurrent HTTP connections?
- What happens when an HTTP request arrives while processing another?
- How does backpressure work in HTTP request/response streams?
- How does the event loop poll phase process HTTP requests?
- What role does libuv play in HTTP request/response processing?
- How does HTTP request parsing work incrementally?
- How are HTTP responses sent back to clients?