# Core: Hub

## Coroutines

h:sleep(ms)
: Suspends the current green thread until at least `ms` milliseconds in the
  future, when it will be resumed.

h:spawn(f)
: Spawns and queues to run the callable `f` as a new green thread. The current
  thread will yield to be resumed after 1 tick of the event loop.

h:spawn_later(ms, f)
: Schedules the callable `f` to run at least `ms` milliseconds in the future,
  in a new green thread.

```lua
local h = require("levee").Hub()

h:spawn(function()
  while true do
    print("tick")
    h:sleep(1000)
  end
end)

h:sleep(500)

while true do
  print("tock")
  h:sleep(1000)
end
```

## IO

h.io:stdin()
: Returns an [io.R](#ior) object to work with this processes stdin

```lua
local h = require("levee").Hub()

local stdin = h.io:stdio()
local stream = stdin:stream()

while true do
  local err, line = stream:line()
  if err then break end
  print(line)
end
```

h.io:stdout()
: Returns an [io.W](#w) object to work with this processes stdout


## Network

h.stream:dial(endpoint|options)
: Establishes a streamed network connection based on the supplied `endpoint`
  string or `options` (`endpoint` TBD). Returns `err`, [io.RW](#ior)

Options are:

- `unix`: path to unix domain socket (TBD) *OR*

- `port`: port to connect to

- `host`: host to connect to (default: localhost)

- `timeout`: timeout for reads and writes on this connection

- `connect_timeout`: timeout to apply to connection

- `tls`: upgrade this connection to use tls. Note `timeout` also applies to the
  tls handshake. The value is a table of [TLS Options](#misc-tls-options).

h.stream:listen(endpoint|options)
: Binds to `endpoint` string or as specified with `options` and listens for
  streamed connections. Returns `err`, [Recver](#TDB). The Recver yields `err`,
  [io.RW](#ior) for each accepted connection.

Options are:

- `unix`: path to unix domain socket (TBD) *OR*

- `port`: port to bind to (default: 0)

- `host`: host to bind to (default: localhost)

- `backlog`: the maximum length for the queue of pending connections (default:
  256)

- `timeout`: sets the read / write timeout for accepted connections

- `tls`: upgrade accepted connections to use tls. Note `timeout` also
  applies to the tls handshake. The value is a table of [TLS
  Options](#misc-tls-options).

```lua
local h = require("levee").Hub()

-- a basic echo server
local err, serve = h.stream:listen({port=9000})

while true do
    local err, conn = serve:recv()
    if err then break end
    h:spawn(function()
        local buf = levee.d.Buffer(4096)
        conn:readinto(buf:tail())
        conn:write(buf:value())
        conn:close()
    end)
end
```

## Process

h.process:spawn(path, [options])
: Spawns the binary specified by `path`, and returns a `child` object to
  interact with the child process.

Options are:

- `argv`: list of command line arguments

- `io`: a table that describes how to treat the child process' IO. By
  default the `STDIN`, `STDOUT` and `STDERR` of the child process are
  captured and available to the parent process, for writing and reading,
  through the variables `stdin`, `stdout` and `stderr`. Alternatively, it's
  possible to assign the child's `STDIN`, `STDOUT` and `STDERR` to a
  specific file descriptor. By setting `STDIN = 0`, `STDOUT = 1` or `STDERR
  = 2`, the child's `stdin`, `stdout` or `stderr` will be left
  unchanged. Addtionally, it is possible to assign an arbitrary file
  descriptor to another file descriptor. This is useful when the child
  wants to write to the parent without using `STDOUT` or `STDERR`. In this
  case, the child must know the value of the writing file descriptor ahead
  of time.

```lua
local levee = require("levee")
local _ = levee._

local h = levee.Hub()

local parent_r, parent_w = _.pipe()
local stdout_r, stdout_w = _.pipe()

local cmd_fd_no = 1020
local cmd_fd_path = _.path.join("/dev/fd", tostring(cmd_fd_no))

local cmd = "tee"
local cmd_args = {cmd_fd_path}
local io_args = {[cmd_fd_no]=parent_w, STDOUT=stdout_w}
local child = h.process:spawn(cmd, {argv=cmd_args, io=io_args})

child.stdin:write("foo")
print(_.reads(parent_r)) -- foo
print(_.reads(stdout_r)) -- foo

child.stdin:close()
child.done:recv()
```

## HTTP

h.http:dial([endpoint|host], options)
: `endpoint|host` is a string. Returns `err`, [HTTP.Client](#httpclient)

Options are:

- `unix`: path to unix domain socket (TBD) *OR*

- `port`: port to connect to

- `host`: host to connect to (default: localhost)

- `timeout`: timeout for reads and writes on this connection

- `connect_timeout`: timeout to apply to connection

- `tls`: upgrade this connection to use tls. Note `timeout` also applies to the
  tls handshake. The value is a table of [TLS Options](#misc-tls-options).

h.http:listen(endpoint|options)
: Binds to `endpoint` string or as specified with `options` and listens for
  streamed connections. Returns `err`, [Recver](#TDB). The Recver yields `err`,
  [HTTP.Server](#httpserver) for each accepted connection.

Options are:

- `unix`: path to unix domain socket (TBD) *OR*

- `port`: port to bind to (default: 0)

- `host`: host to bind to (default: localhost)

- `backlog`: the maximum length for the queue of pending connections (default:
  256)

- `timeout`: sets the read / write timeout for accepted connections

- `tls`: upgrade accepted connections to use tls. Note `timeout` also
  applies to the tls handshake. The value is a table of [TLS
  Options](#misc-tls-options).

# Objects: IO

## io.R

r:read(char*, len)
: Reads up to `len` bytes into `char*`. Returns `err`, `n` where `n` is the
  number of bytes actually read

r:readn(char*, n, [len])
: Reads *at least* `n` bytes into `char*`, but *no more* then `len`. `len`
  defaults to `n`. Returns `err`, `n` where `n` is the number of bytes actually
  read

r:readinto(buf, n)
: Reads *at least* `n` bytes into [buf](#dbuffer). Handles ensuring the `buf`
  is large enough to accommodate `n` and bumps the contents marker. Returns
  `err`.

r:stream()
: Returns an [io.Stream](#iostream)

r:recvfd():
: Receives a file descriptor and possibly a `d.Iovec`. `r` must be a Unix
  socket. For single and forked processes, `io.socketpair` can be used to
  create this socket along with the sending socket (the one that calls
  `sendfd`). Returns `err`, `fd`, `iov`.

```lua

local levee = require("levee")
local d = require("levee").d

local h = levee.Hub()

local r, w = h.io:socketpair(C.AF_UNIX, C.SOCK_STREAM)
local iov = d.Iovec(1)
local out = h.io:stdout()

out:write("My name is what?\n")
iov:write("Slim Shady\n")
w:sendfd(out.no, iov)

err, fd, iov = r:recvfd()
out = h.io:w(fd)
out:write(iov:string())
```

## io.W

w:write(buf, [len])
: Writes `buf`. `buf` can either be a `char *` or a `string`. `len` defaults to
  `#buf`.  Returns `err`.

w:sendfd(fd, [iov]):
: Sends a file descriptor, `fd`, and an optional `d.Iovec`, `iov`.  `w`
  must be a Unix socket. For single and forked processes, `io.socketpair`
  can be used to create this socket along with the receiving socket (the
  one that calls `recvfd`). Returns `err`

## io.Stream

A stream is a combination of an [io.R](#ior) and a [d.Buffer](#dbuffer).

stream:readin([n])
: Reads additional bytes into this stream's buffer. `n` is optional. If
  supplied this call will block until *at least* `n` bytes are available in the
  buffer. If that many bytes are already available, it will return immediately.
  If `n` is not supplied this call will block until one successful read has
  been made.

# Objects: HTTP

## HTTP.Client

## HTTP.Server

# Misc: TLS Options

```
  Certificate Authority:
    ca = BYTES            # root certificates from string
    ca_path = DIRECTORY   # directory searched for root certificates
    ca_file = FILE        # file containing the root certificates
  Certificate:
    cert = BYTES          # public certificate from string
    cert_file = FILE      # file containing the public certificate
  Key:
    key = BYTES           # private key from string
    key_file = FILE       # file containing the private key
  Ciphers:
    ciphers = "secure"    # use the secure ciphers only (default)
    ciphers = "compat"    # OpenSSL compatibility
    ciphers = "legacy"    # (not documented)
    ciphers = "insecure"  # all ciphers available
    ciphers = "all"       # same as "insecure"
    ciphers = STRING      # see CIPHERS section of openssl(1)
  DHE Params:
    dheparams = STRING    # (not documented)
  ECDHE Curve:
    ecdhecurve = STRING   # (not documented)
  Protocols:
    protocols = "TLSv1.0" # only TLSv1.0
    protocols = "TLSv1.1" # only TLSv1.1
    protocols = "TLSv1.2" # only TLSv1.2
    protocols = "ALL"     # all supported protocols
    protocols = "DEFAULT" # currently TLSv1.2
    protocols = LIST      # any combination of the above strings
  Verfiy Depth:
    verify_depth = NUMBER # limit verification depth (?)
  Server:
    server = {
      prefer_ciphers = "server"  # prefer client cipher list (less secure)
      prefer_ciphers = "client"  # prefer server cipher list (more secure, default)
      verify_client = true       # require client to send certificate
      verify_client = "optional" # enable client to send certificate
    }
  Insecure:
    insecure = {
      verify_cert = false        # disable certificate verification
      verify_name = false        # disable server name verification for client
      verify_time = false        # disable validity checking of certificates
    }
```

# require("levee").\_

**\_** is for utilities

## Network

\_.endpoint_in(host, port)
: Returns `ep`, an IPV4 [Endpoint](#endpoint) for the given `host` and `port`.

\_.endpoint_unix(name)
: Returns `ep`, a Unix Domain [Endpoint](#endpoint) for the given path `name`.

\_.socket(domain, socktype, [protocol])
: Returns `err`, `no`. `protocol` defaults to `0`

_.connect = function(no, endpoint)
: Connects socket `no` to endpoint `endpoint`. Returns `err`, `no`

_.bind = function(no, endpoint)
: Binds socket `no` to endpoint `endpoint`. Returns `err`, `no`

_.listen = function(no, endpoint, [backlog])
: Binds and listens socket `no` to endpoint `endpoint`. Returns `err`, `no`

\_.sendto(no, who, buf, len)
: Sends `buf`, `len` over file descriptor `no` to [Endpoint](#endpoint) `who`. Returns
  `err`, `n`, where `n` is the number of characters sent on success.

\_.recvfrom(no, buf, len)
: Receives from file descriptor `no` into `buf`, `len`. Returns `err`, `who`,
  `n` where `who` is an [Endpoint](#endpoint) object of the sender and `n` is
  the number of bytes received.

## File System

\_.stat(path)
: Returns `err`, `statinfo` for the file pointed to by path where `statinfo` is
  a [Stat](#stat) object.

## Stat

stat:is_reg()
: Returns `true` if this is a regular file.

stat:is_dir()
: Returns `true` if this is a directory.

# require("levee").d

**d** is for data structure thingies

## d.Buffer

A `Buffer` is designed to be a reusable scratch pad of memory. It can grow
dynamically if it's initial sizing is too small, but eventually you usually
want the size of the buffer to reach a steady state. It's the work horse data
structure of the Levee library. It is used, for example, to create streaming
protocol parsers. The parser reads bytes into the `Buffer` until the next token
in the protocol is reached and the parser can then yield the next portion of
the protocol and then `:trim` the `Buffer` to reset the memory allocation for
reuse.

d.Buffer([bytes])
: allocates and returns a new `buf`. `bytes` is a sizing hint for the initial
  allocation of memory for this `Buffer`

buf:ensure([bytes])
: ensures the `Buffer` has *at least* `bytes` available of allocated space, in
  addition to what's currently in use.

buf:write(buf, [len])
: copies `buf` into the tail of the `Buffer`. `buf` can either be a `char *` or a
  `string`. `len` defaults to `#buf`. Write ensures the buffer is large enough to
  hold the write and bumps the `Buffer`'s content marker

buf:value([[off], len])
: If `len` is supplied it should be less than the current length of the
  `Buffer`. `len` defaults to the entire `Buffer`'s contents. The optional
  `off` offsets the returned `char *` from the beginning of the `Buffer`'s
  contents.  Returns `char*`, `len`

buf:tail()
: returns `char*`, `len` to the tail of the allocated `Buffer` that's not
  currently in use

buf:bump(len)
: moves the marker for in use bytes by `len`

buf:trim([len])
: marks `len` bytes of the `Buffer` as available for reuse. If `len` is not
  supplied to entire `Buffer` is marked. Returns `n`, the number of bytes
  trimmed

```lua
local buf = d.Buffer()

buf:ensure(3)
ffi.copy(buf:tail(), "foo")
buf:bump(3)
ffi.string(buf:value())  -- "foo"

buf:write("bar")
ffi.string(buf:value())  -- "foobar"

ffi.string(buf:value(3))  -- "foo"
ffi.string(buf:value(3, 1))  -- "b"

buf:trim()
ffi.string(buf:value())  -- ""
```

## d.Iovec

d.Iovec([size])
: *todo*


# require("levee").p

**p** is for parsing / protocol jobbies

## p.json

p.json.decode(buf, len)
: Decode the JSON compliant string `buf`, `len`. `buf` can be a Lua string.
  Returns `err`, `data`.

```lua
local p = require("levee").p

local err, data = p.json.decode([[{"foo": "bar"}]])
data.foo  -- "bar"
```
