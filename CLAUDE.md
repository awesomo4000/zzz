# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

zzz is a high-performance networking framework for Zig focused on HTTP/HTTPS servers. It's built on top of Tardy (an async runtime) and uses BearSSL for TLS support. The framework emphasizes modularity, performance through minimal allocations, and flexibility in async implementation.

## Build Commands

```bash
# Build all examples
zig build

# Build specific example
zig build basic
zig build tls
zig build middleware
zig build fs
zig build form
zig build cookies
zig build sse
zig build unix  # (not available on Windows)

# Run specific example
zig build run_basic
zig build run_tls
zig build run_middleware
# ... etc

# Run tests
zig build test

# Fetch dependencies (when updating)
zig fetch --save git+https://github.com/mookums/zzz#v0.3.0
```

## Architecture

### Core Structure
- **src/lib.zig**: Main entry point, exports HTTP module and Tardy runtime
- **src/http/lib.zig**: HTTP module aggregating all HTTP-related components
- **src/core/**: Low-level utilities (secure sockets, typed storage, wrapping, pseudoslice)
- **src/tls/**: TLS implementation using BearSSL

### Key Components

1. **Router System** (src/http/router.zig, src/http/router/*)
   - Uses a Routing Trie for efficient path matching
   - Supports middleware layers
   - Route definitions with HTTP method chaining
   - Capture groups and path parameters

2. **Server** (src/http/server.zig)
   - Built on Tardy async runtime
   - Configurable connection limits, keepalive, buffer sizes
   - Support for plain and TLS connections via SecureSocket abstraction
   - Memory pooling to minimize allocations

3. **Request/Response Cycle**
   - **Context** (src/http/context.zig): Contains request, response, captures, and typed storage
   - **Request** (src/http/request.zig): HTTP request parsing and handling
   - **Response** (src/http/response.zig): Response building with Respond union type
   - **Middleware** (src/http/router/middleware.zig): Layered middleware system with Next function

4. **Async Runtime Integration**
   - Uses Tardy for async I/O (io_uring, epoll, kqueue, poll)
   - Socket abstraction for network operations
   - Task-based concurrency model

## Common Patterns

### Route Handler Signature
```zig
fn handler(ctx: *const Context, data: DataType) !Respond {
    return ctx.response.apply(.{
        .status = .OK,
        .mime = http.Mime.HTML,
        .body = "response",
    });
}
```

### Server Setup Pattern
```zig
// 1. Initialize Tardy runtime
var t = try Tardy.init(allocator, .{ .threading = .auto });

// 2. Create router with routes
var router = try Router.init(allocator, &.{
    Route.init("/").get({}, handler).layer(),
}, .{});

// 3. Create and bind socket
var socket = try Socket.init(.{ .tcp = .{ .host = host, .port = port } });
try socket.bind();
try socket.listen(4096);

// 4. Start server in Tardy entry
try t.entry(params, entry_function);
```

### Middleware Pattern
Middlewares wrap handlers and can modify request/response or terminate early. They receive a Next function to call the next layer.

## Dependencies

- **tardy**: Async runtime (git dependency)
- **bearssl**: TLS implementation (git dependency)
- Minimum Zig version: 0.13.0

## Testing Approach

Use `zig build test` to run unit tests. Test files are in src/unit_test.zig.

## Important Configuration

### Server Config Options
- `stack_size`: Task stack size (default 1MB)
- `connection_count_max`: Max concurrent connections per runtime
- `keepalive_count_max`: Max keepalive requests per connection
- `socket_buffer_bytes`: Buffer size for socket operations
- `arena_retain_bytes`: Memory retained after arena clear

### Security Modes
- Plain: No encryption
- TLS: Using BearSSL with certificate and key files