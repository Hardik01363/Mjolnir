# Mjolnir

Mjolnir is a project I've built on top of Electric Fence, a memory debugger that uses guard pages to catch memory bugs. The goal of this project is to take Electric Fence, fix a concurrency bug in its own internal code, and turn it into a fast, thread safe tool that can reliably catch use after free bugs and race conditions in multi threaded programs, while also giving a clear, structured report of how each bug happened.

## Why this project exists

While comparing different memory debugging tools on a deliberately built concurrent use after free bug, I found that Electric Fence consistently failed to catch the bug, even though it is a tool built specifically for this kind of problem. Tools like ASan, TSan, and Helgrind caught the bug reliably, but Electric Fence missed it every single time. The reason turned out to be that Electric Fence keeps an internal table to track every allocation, and this table has no protection at all when multiple threads use it at once. This project exists to fix that problem properly and then push the tool further.

## What my project does

This project has three main parts.

**Part one, fix Electric Fence with locks.** Proper synchronization is added around the internal allocation table so it can be safely used by many threads at once. This is tested and compared against the original unsafe version to show the improvement in detection.

**Part two, make it lock free.** Once the locked version is working correctly, the locks are replaced with atomic, compare and swap based code, so the tool stays fast and close to native speed while still being fully thread safe.

**Part three, add a structured reporting layer.** Instead of Electric Fence just crashing with a plain segmentation fault, an internal log records every allocation, free, and access along with the thread id and a timestamp. When a bug is caught, this log is used to build a clear report showing which thread allocated the memory, which thread freed it, which thread accessed it after, and in what order. This also allows the tool to automatically tell apart two kinds of bugs, one where memory is accessed after being freed but before it is reused, and (a harder one to catch) where the memory was already reused for a new allocation before the bad access happened.

## Project background

Electric Fence gives every allocation its own memory using mmap, and places a guard page next to it using mprotect. When memory is freed, the page can be permanently locked so that any later access causes a real crash. This method is simple and fast, but the original code was written before multi threaded programs were common, so its internal bookkeeping was never made safe for concurrent use. This project fixes exactly that, while keeping the speed and simplicity that made Electric Fence useful in the first place.

## Build

This project builds the same way as the original Electric Fence.

To build, run
```
scons
```

To clean a build, run
```
scons -c
```

## Usage

Electric Fence can be used the same way as before, either by linking the built static library into your application, or by preloading the shared library at runtime.

Example, on Linux
```
LD_PRELOAD=./path/to/library/libefence.so /bin/myapplication
```

The environment variable EF_PROTECT_FREE should be set to 1 so that freed pages are permanently locked instead of being reused right away. This is required for reliable detection of use after free bugs, not just memory overruns.

## Reports

Two reports are part of this project.

**Report one** compares detection results before and after the locked fix is applied.

**Report two** compares speed, detection rate, and overhead between the locked version and the lock free version, along with the overhead compared to running with no protection at all.

Both reports will be added to this repository once complete.

## Credit

This project is built directly on top of Electric Fence. Full credit for the original tool goes to Bruce Perens, the original author, and to Alexander von Gluck IV, who maintains the actively developed fork this project is based on. Without their work, none of this would be possible.

## License

Electric Fence is released under the GPLv2 license.
