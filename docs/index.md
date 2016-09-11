# \_

\_ is for utilities

\_.stat(path)
: Returns `err`, `statinfo` for the file pointed to by path where `statinfo` is
  a `Stat` object.

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
local h = require("levee")

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

## Network

.stream:dial(port, [host])
: Establishes a streamed network connection with `port` and `host`. `host`
  defaults to `localhost`. Returns `err`, [io.RW](#io)

# io

## R

:read(buf, len)
: Reads up to `len` bytes into `buf`. Returns `err`, `n` where `n` is the
  number of bytes actually read

## W

:write(buf, len)
: Writes `len` bytes of `buf`. Returns `err`.
