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

# require("levee").p

**p** is for parsing / protocol jobbies

## p.json

p.json.decode(buf, len)
: Decode the JSON compliant string `buf`, `len`. `buf` can be a Lua string.
  Returns `err`, `data`.

```lua
local p = require("levee").p

local err, data = p.json.decode([[{"foo": "bar"}]])
print(data.foo)  -- "bar"
```

# Objects: IO

## io.R

r:read(buf, len)
: Reads up to `len` bytes into `buf`. Returns `err`, `n` where `n` is the
  number of bytes actually read

r:stream()
: Returns an [io.Stream](#iostream)

## io.W

w:write(buf, len)
: Writes `len` bytes of `buf`. Returns `err`.

## io.Stream

A stream is a combination of an [io.R](#ior) and a [d.Buffer](#dbuffer).

stream:readin(n)
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
: Returns an [io.R](#r) object to work with this processes stdin

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

h.stream:dial(port, [host])
: Establishes a streamed network connection with `port` and `host`. `host`
  defaults to `localhost`. Returns `err`, [io.RW](#io)
