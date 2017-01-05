# require("levee").\_

**\_** is for utilities

## Network

\_.endpoint_in(host, port)
: Returns `ep` an IPV4 [Endpoint](#endpoint) for the given `host` and `port`.

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

buf:value()
: returns `char*`, `len` of the current contents of the `Buffer`

buf:tail()
: returns `char*`, `len` to the tail of the allocated `Buffer` that's not
  currently in use.

buf:bump([len])
: moves the marker for in use bytes by `len`

buf:trim([len])
: marks `len` bytes of the `Buffer` as available for reuse. If `len` is not
  supplied to entire `Buffer` is marked. Returns `n`, the number of bytes
  trimmed.

```lua
local buf = d.Buffer()

buf:ensure(3)

ffi.copy(buf:tail(), "foo")
buf:bump(3)
ffi.string(buf:value())  -- "foo"

buf:trim()
ffi.string(buf:value())  -- ""
```

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

# Objects: IO

## io.R

r:read(buf, len)
: Reads up to `len` bytes into `buf`. Returns `err`, `n` where `n` is the
  number of bytes actually read

r:readn(buf, n, [len])
: Reads *at least* `n` bytes into `buf`, but *no more* then `len`. `len`
  defaults to `n`. Returns `err`, `n` where `n` is the number of bytes actually
  read

r:stream()
: Returns an [io.Stream](#iostream)

## io.W

w:write(buf, len)
: Writes `len` bytes of `buf`. Returns `err`.

## io.Stream

A stream is a combination of an [io.R](#ior) and a [d.Buffer](#dbuffer).

stream:readin([n])
: Reads additional bytes into this stream's buffer. `n` is optional. If
  supplied this call will block until *at least* `n` bytes are available in the
  buffer. If that many bytes are already available, it will return immediately.
  If `n` is not supplied this call will block until one successful read has
  been made.

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

- `local`: path to unix domain socket (TBD) *OR*

- `port`: port to connect to (default: 0)

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

- `local`: path to unix domain socket (TBD) *OR*

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
