---
name: go-delve-debugging
description: Debug Go programs and tests with Delve from the terminal. Use when stepping through failing tests, setting breakpoints, inspecting variables, tracing goroutines or deadlocks, attaching to a running Go process, examining a core dump, or diagnosing race conditions and state changes in any Go repository.
---

# Debug Go with Delve

Use `dlv` as the default debugger for Go CLI debugging. Prefer terminal workflows over IDE-specific instructions, and choose the launch mode that matches the artifact being debugged.

## Choose the launch mode

Pick the narrowest entrypoint that reproduces the bug:

| Command | Use for |
| --- | --- |
| `dlv debug` | Build and debug the `main` package in the current directory |
| `dlv debug ./cmd/server` | Debug a specific `main` package |
| `dlv debug ./cmd/server -- --port 8080` | Pass program arguments after `--` |
| `dlv test` | Build and debug tests in the current directory |
| `dlv test ./pkg/auth` | Debug tests in a specific package |
| `dlv test ./pkg/auth -- -run TestLogin` | Debug one test or a filtered test set |
| `dlv exec ./binary` | Debug a prebuilt binary |
| `dlv exec ./binary -- --flag value` | Debug a prebuilt binary with arguments |
| `dlv attach <pid>` | Attach to a running process |
| `dlv core ./binary ./core` | Inspect a core dump |

If the target is a test failure, prefer `dlv test`. If the target is a CLI or service entrypoint, prefer `dlv debug` on the exact `main` package instead of the repo root.

## Drive the REPL

Use these commands to control execution and inspect state.

### Set breakpoints

```text
break main.go:42                  # file:line
break main.main                   # function entry
break pkg.(*T).Method             # method on type
break /Handler/                   # regex match on function name
break mybp main.go:42             # named breakpoint
break main.go:42 if x > 10        # conditional breakpoint

condition 2 i == 5                # add condition to breakpoint #2
condition -hitcount 2 > 3         # stop after the 3rd hit
condition -per-g-hitcount 2 == 1  # stop once per goroutine
condition -clear 2                # clear breakpoint condition

watch v                           # break on read or write
watch -w v                        # break on write only
watch -r v                        # break on read only

trace main.go:42                  # print without stopping
on 2 print x                      # run a command when breakpoint #2 hits

breakpoints                       # list breakpoints
clear 2                           # delete breakpoint #2
clearall                          # delete all breakpoints
toggle 2                          # enable or disable breakpoint #2
```

### Control execution

```text
continue      (c)   # run until a breakpoint or program exit
next          (n)   # step over
step          (s)   # step into
stepout       (so)  # step out
restart       (r)   # restart and keep breakpoints
rebuild             # rebuild, restart, and keep breakpoints
call f(x)           # call a function inside the debugged process
```

### Inspect state

```text
print x             (p x)   # evaluate an expression
print %x myVar              # print with a Go fmt verb
print *myPtr                # dereference a pointer
print mySlice[2:5]          # inspect a slice range
print myMap["key"]          # inspect a map value
print myStruct.Field        # inspect a struct field

locals                      # list local variables
locals -v                   # include type information
locals err                  # filter locals by regex
args                        # list function arguments
args -v                     # include type information
whatis x                    # print the type of x
regs                        # print CPU registers
```

### Navigate stacks and goroutines

```text
stack             (bt)      # stack trace
stack 20                    # deeper stack trace
stack -full                 # include locals for each frame
frame 3                     # switch to frame #3
up                          # move up one frame
down                        # move down one frame
deferred 2                  # inspect deferred call #2

goroutines                  # list goroutines
goroutines -t               # include stack traces
goroutines -l               # include labels
goroutines -with userloc Handler
goroutines -group userloc
goroutine 5                 # switch to goroutine #5
goroutine 5 bt              # show goroutine #5 stack
goroutine 5 print x         # inspect x in goroutine #5 context

threads                     # list OS threads
thread 3                    # switch to thread #3
```

### Navigate source

```text
list              (ls)      # show source near the current line
list main.go:42             # show source near a line
list main.main              # show source for a function
funcs                       # list functions
funcs test.Test*            # filter functions by regex
sources                     # list source files
sources auth                # filter source files
types                       # list types
types Request               # filter types
```

## Run common workflows

### Debug a failing test

```bash
dlv test ./pkg/auth -- -run TestCreateUser -v
```

```text
(dlv) break TestCreateUser
(dlv) continue
(dlv) next
(dlv) print user
(dlv) locals
```

Use this flow when a test reproduces reliably and you want to stop before the bad assertion or state transition.

### Find where a value becomes wrong

```text
(dlv) break pkg/service.go:88 if result == nil
(dlv) continue
(dlv) stack
(dlv) frame 1
(dlv) locals
```

Add a condition to stop at the first suspicious state instead of stepping through every call.

### Investigate goroutine issues

```text
(dlv) break deadlock_suspect.go:45
(dlv) continue
(dlv) goroutines -t 5
(dlv) goroutines -with userloc mu
(dlv) goroutine 8 bt
```

Use this flow for deadlocks, blocked channels, mutex contention, or worker pools that appear stuck.

### Keep an edit-debug loop tight

```text
# After editing source:
(dlv) rebuild
(dlv) continue
```

Use `rebuild` when source changed and you want to keep existing breakpoints.

## Apply practical tips

- Use tab completion in the REPL for commands and symbols.
- Use `on <bp> print <expr>` when you want lightweight tracing without stopping.
- Use `restart` to rerun quickly and `rebuild` when code changed.
- Use `print` for most Go expressions, including field access, indexing, slicing, and type assertions.
- Use `dlv debug --build-flags='-race'` or `dlv test --build-flags='-race'` when reproducing race conditions.
- Use `dlv debug --build-flags='-gcflags=all=-N -l'` if optimized code makes variables hard to inspect.
- Use `--headless --api-version=2` when running Delve as a server for remote debugging.
- Use `help <command>` inside Delve when a REPL command's flags are unclear.
- Use `~/.config/dlv/config.yml` to define defaults such as aliases or output limits.
