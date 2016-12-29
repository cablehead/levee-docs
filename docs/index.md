# \_

\_ is for utilities

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

:is_reg()
: Returns `true` if this is a regular file.

:is_dir()
: Returns `true` if this is a directory.

# d

d is for data structure like things

## buffer

# p

p is for parsing / protocol jobbies

## json

p.json.decode(buf, len)
: Decode the JSON compliant string `buf`, `len`. `buf` can be a Lua string.
  Returns `err`, `data`.

```lua
local p = require("levee").p

local err, data = p.json.decode([[{"foo": "bar"}]])
print(data.foo)  -- "bar"
```

# IO

## R

:read(buf, len)
: Reads up to `len` bytes into `buf`. Returns `err`, `n` where `n` is the
  number of bytes actually read

:stream()
: Returns an [io.Stream](#io)

## W

:write(buf, len)
: Writes `len` bytes of `buf`. Returns `err`.

## Stream

A stream is a combination of an `io.R` and a `d.Buffer`.

:readin(n)
: Reads additional bytes into this stream's buffer. `n` is optional. If
  supplied this call will block until at least `n` bytes are available in the
  buffer. If that many bytes are already available, it will return immediately.
  Otherwise this call will block until one successful read has been made.

# Hub

## Coroutines

:sleep(ms)
: Suspends the current green thread until at least `ms` milliseconds in the
  future, when it will be resumed.

:spawn(f)
: Spawns and queues to run the callable `f` as a new green thread. The current
  thread will yield to be resumed after 1 tick of the event loop.

:spawn_later(ms, f)
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

.io:stdin()
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

.io:stdout()
: Returns an [io.W](#w) object to work with this processes stdout


## Network

.stream:dial(port, [host])
: Establishes a streamed network connection with `port` and `host`. `host`
  defaults to `localhost`. Returns `err`, [io.RW](#io)
