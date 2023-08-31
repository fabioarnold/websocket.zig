A zig websocket server.

Zig 0.12-dev is rapidly changing and constantly breaking features. I'll try to keep this up to date as much as possible. Note that 0.12-dev currently does not support async, so I've reverted to threads.

See the [v0.10.1-compat](https://github.com/karlseguin/websocket.zig/tree/v0.10.1-compat) tag for a release that's compatible with Zig 0.10.1.

## Example
```zig
const websocket = @import("websocket");
const Conn = websocket.Conn;
const Message = websocket.Message;
const Handshake = websocket.Handshake;

// Define a struct for "global" data passed into your websocket handler
const Context = struct {

};

pub fn main() !void {
    var general_purpose_allocator = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = general_purpose_allocator.allocator();

    // this is the instance of your "global" struct to pass into your handlers
    var context = Context{}; 

    try websocket.listen(Handler, allocator, &context, .{
        .port = 9223,
        .max_headers = 10,
        .address = "127.0.0.1",
    });
}

const Handler = struct {
    conn: *Conn,
    context: *Context,

    pub fn init(h: Handshake, conn: *Conn, context: *Context) !Handler {
        // `h` contains the initial websocket "handshake" request
        // It can be used to apply application-specific logic to verify / allow
        // the connection (e.g. valid url, query string parameters, or headers)

        _ = h; // we're not using this in our simple case

        return Handler{
            .conn = conn,
            .context = context,
        };
    }

    // optional hook that, if present, will be called after initialization is complete
    pub fn afterInit(self: *Handler) !void {}

    pub fn handle(self: *Handler, message: Message) !void {
        const data = message.data;
        try self.conn.write(data); // echo the message back
    }

    // called whenever the connection is closed, can do some cleanup in here
    pub fn close(_: *Handler) void {}
};
```

### init
The `init` method is called with a `websocket.Handshake`, a `*websocket.Conn` and whatever `context` value was passed as the 3rd parameter to `websocket.listen`.

The websocket specification requires the initial "handshake" to contain certain headers and values. The library validates these headers. However applications may have additional requirements before allowing the connection to be "upgraded" to a websocket connection. For example, a one-time-use token could be required in the querystring. Applications should use the provided `websocket.Handshake` to apply any application-specific verification and optionally return an error to terminate the connection.

The `websocket.Handshake` exposes the following fields:

* `url: []const u8` - URL of the request in its original casing
* `method: []const u8` - Method of the request in its original casing
* `raw_headers: []const u8` - The raw "key1: value1\r\nkey2: value2\r\n" headers. Keys are lowercase.

If you set the `max_headers` configuration value to > 0, then you can use `req.headers.get("HEADER_NAME")` to extract a header value from the given name:

```zig
// get returns a ?[]const u8
// the name is lowercase
// the value is in its original case
const token = handshake.headers.get("authorization") orelse {
    return error.NotAuthorized;
}
...
```

Memory referenced by the `websocket.Handshake`, including headers from `handshake.headers` will be freed after the call to `init` completes. Application that need these values to exist beyond the call to `init` must make a copy.

### handle
The `handle` function takes a `message` which has a `type` and `data` field. The `type` will either be `text` or `binary`, but in 99% of cases, you can ignore it and just use `data`. This is an unfortunate part of the spec which differentiates between arbitrary byte data (binary) and valid UTF-8 (text). This library _does not_ reject text messages with invalid UTF-8. Again, in most cases, it doesn't matter and just wastes cycles. If you care about this, do like the autobahn test does and validate it in your handler.

If `handle` returns an error, the connection is closed.

### close
Called whenever the connection terminates. By the time `close` is called, there is no guarantee about the state of the underlying TCP connection (it may or may not be closed).

## websocket.Conn
The call to `init` includes a `*websocket.Conn`. It is expected that handlers will keep a reference to it. The main purpose of the `*Conn` is to write data via `conn.write([]const u8)` and `conn.writeBin([]const u8)`. The websocket protocol differentiates between a "text" and "binary" message, with the only difference that "text" must be valid UTF-8. This library does not enforce this. Which you use really depends on what your client expects. For browsers, text messages appear as strings, and binary messages appear as a Blob or ArrayBuffer (depending on how the client is configured).

`conn.close()` can also be called to close the connection. Calling `conn.close()` **will** result in the handler's `close` callback being called.

## Autobahn
Every mandatory [Autobahn Testsuite](https://github.com/crossbario/autobahn-testsuite) case is passing. (Three fragmented UTF-8 are flagged as non-strict and as compression is not implemented, these are all flagged as "Unimplemented").

You can see `autobahn.zig` in the root of the project for the handler that's used for the autobahn tests.

## Config
The 4th parameter to `websocket.listen` is a configuration object. 

* `port` - Port to listen to. Default: `9223`.
* `max_size` - Maximum incoming message size to allow. The server will dynamically allocate up to this much space per request. Default: `65536`.
* `buffer_size` - Size of the static buffer that's available per connection for incoming messages. While there's other overhead, the minimal memory usage of the server will be `# of active connections * buffer_size`. Default: `4096`.
* `address` - Address to bind to. Default: `"127.0.0.1"`.
* `handshake_pool_size` - The number of buffers to create and keep for reading the initial handshake. Default: `50`
* `handshake_max_size` - The maximum size of the initial handshake to allow. Default: `1024`.
* `max_headers` - The maximum size of headers to store in `handshake.headers`. Requests with more headers will still be processed, but `handshake.headers` will only contain the first `max_headers` headers. Default: `0`.
* `handshake_timeout_ms` - The time, in milliseconds, to wait for the handshake to complete. This essentially prevents a client from opening a connection and "hanging" the thread while it waits for data. If a client slowly sends a few bytes at a time, the actual timeout might happen up to 2x longer than specified. Generally speaking, it might be better to let a proxy (e.g. nginx) handle this. Default 10_000;

Setting `max_size == buffer_size` is valid and will ensure that no dynamic memory allocation occurs once the connection is established.

The server allocates and keep around `handshake_pool_size * handshake_max_size` bytes at all times. If the handshake buffer pool is empty, new buffers of `handshake_max_size` are dynamically created.

The handshake pool/buffer is separate from the main `buffer_size` to reduce the memory cost of invalid handshakes. Unless you're expecting a very large handshake request (a large URL, querystring or headers), the initial handshake is usually < 1KB. This data is also short-lived. Once the websocket is established however, you may want a larger buffer for handling incoming requests (and this buffer is generally much longer lived). By using a small pooled buffer for the initial handshake, invalid connections don't use up [as much] memory and thus the server may be more resilient to basic DOS attacks. Keep in mind that the websocket protocol requires a number of mandatory headers, so `handshake_max_size` can't be set to a super small value. Also keep in mind that the current pool implementation does not block when empty, but rather created dynamic buffers of `handshake_max_size`.

Websockets have their own fragmentation "feature" (not the same as TCP fragmentation) which this library could handle more efficiently. However, I'm not aware of any client (e.g. browsers) which use this feature at all.

## Advanced

### Pre-Framed Comptime Message
Websocket message have their own special framing. When you use `conn.write` or `conn.writeBin` the data you provide is "framed" into a correct websocket message. Framing is fast and cheap (e.g., it DOES NOT require an O(N) loop through the data). Nonetheless, there may be be cases where pre-framing messages at compile-time is desired. The `websocket.frameText` and `websocket.frameBin` can be used for this purpose:

```zig
const UNKNOWN_COMMAND = websocket.frameText("unknown command");
...

pub fn handle(self: *Handler, message: Message) !void {
    const data = message.data;
    if (std.mem.startsWith(u8, data, "join: ")) {
        self.handleJoin(data)
    } else if (std.mem.startsWith(u8, data, "leave: ")) {
        self.handleLead(data)
    } else {
        try self.conn.writeFramed(UNKNOWN_COMMAND);
    }
}
```

## Testing
The library comes with some helpers for testing:

```zig
cosnt wt = @import("websocket").testing;

test "handler invalid message" {
    var wtt = wt.init();
    defer wtt.deinit();

    var handler = MyAppHandler{
        .conn = &wtt.conn,
    }

    handler.handle(wtt.textMessage("hack"));
    try wtt.expectText("invalid message");
}
```

For testing websockets, you usually care about two things: emulating incoming messages and asserting the messages sent to the client.

`wtt.conn` is an instance of a `websocket.Conn` which is usually passed to your handler's `init` function. For testing purposes, you can inject it directly into your handler. The `wtt.expectText` asserts that the expected message was sent to the conn. 

The `wtt.textMessage` generates a message that you can pass into your handle's `handle` function.

## http.zig
I'm also working on an HTTP 1.1 server for zig: [https://github.com/karlseguin/http.zig](https://github.com/karlseguin/http.zig).
