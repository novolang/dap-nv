# dap-nv

The Debug Adapter Protocol lets a development tool and a debugger talk
to each other, so that one editor can drive many debuggers. It is
specified by Microsoft; this package implements
[version 1.x](https://microsoft.github.io/debug-adapter-protocol/specification).
It brings the protocol's messages to novo-lang as values an adapter
reads and writes.

The messages travel behind the same `Content-Length` header the Language
Server Protocol uses, and this package takes that framing from
[jsonrpc-nv](https://novo-lang.org/packages/jsonrpc-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the protocol is

A **debug adapter** is a separate process that speaks this protocol on
one side and to a real debugger on the other. The editor is the
**client**.

There are three kinds of message, and each carries a `seq`, a sequence
number.

| Message | Direction | Answered |
| --- | --- | --- |
| Request | Either way | Yes, with a response |
| Response | The other way | — |
| Event | Adapter to client | No |

The sequence is **per sender**. Each party numbers its own messages from
1 upwards, and every message it sends takes the next number, responses
and events included. A response names the request it answers in a
separate member, `request_seq`. The two parties' sequences are
unrelated.

A response says whether the request succeeded with a `success` boolean.
A failed one carries a short `message` and, in its body, a structured
error with a **format string** such as `"cannot open {path}"` and its
variables separately, so that a client can localise it.

A session starts with a handshake of four steps, and their order is not
the one a reader expects.

1. The client sends `initialize`. The adapter answers with its
   **capabilities**, a list of what it supports.
2. The client sends `launch` or `attach`. The adapter does **not**
   finish this request yet.
3. The adapter sends the `initialized` **event** when it is ready for
   configuration. The client sends its breakpoints and finishes with
   `configurationDone`.
4. Only now does the adapter complete step 2 and start the program.

Once the program stops, the adapter sends a `stopped` event, and the
client asks for threads, then a stack trace, then scopes, then
variables.

A variable whose value is structured does not carry its children. It
carries a **variables reference**, a non-zero integer the client sends
back in a `variables` request. Zero means there is nothing to expand.
**Every reference is invalidated when execution resumes**, and the
adapter may reuse the numbers for other values.

Line numbers in this protocol are **one-based**.

Every function in this package performs no input or output. Starting
processes, reading a debuggee's memory and talking over a pipe are the
work of the adapter built on it.

## Install

```
novo pkg add dap-nv
```

## Example

```novo
use dapmsg
use dapwire
use dapcmd
use dapevent

fn main() [io]
    // A reader holds the bytes that have arrived but are not yet a
    // whole message. The framing is jsonrpc-nv's.
    let r = dapwire.reader()

    // The adapter announces what it supports. Everything is off until
    // it is implemented, so the default cannot lie.
    let caps = dapcmd.default_capabilities()

    // A message that arrived.
    match dapwire.feed(r, dapwire.frame(
              DapRequestMsg(dapmsg.request(1, "threads", None))))
        Err(f)   =>
            // A framing fault means close; a message fault does not.
            if dapwire.is_framing_fault(f)
                println("closing: ${f.message()}")
            else
                println("refused: ${f.message()}")
        Ok(step) =>
            match step.message
                None    => println("a partial message; read more")
                Some(m) =>
                    match m
                        DapRequestMsg(q) =>
                            let command = dapcmd.command_of(q.command)
                            // Refuse anything out of order rather than
                            // answering it in a state where the answer
                            // is meaningless.
                            if not dapcmd.allows(DapRunning, command)
                                let e = dapmsg.error_message(1, "not now")
                                println(dapwire.encode(DapResponseMsg(
                                    dapmsg.failed(2, q.seq, q.command, "badState", e))))
                            else
                                println("handling ${q.command}")
                        // An event only ever comes from the adapter.
                        DapEventMsg(e) =>
                            if dapevent.invalidates_references(dapevent.event_of(e.event))
                                println("every variables reference is now dead")
                        DapResponseMsg(_) => println("an answer")

    println("${caps.supports_configuration_done}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: dap-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `dapmsg` | The three-message envelope, the sequence numbers, and the structured error a failed response carries. |
| `dapwire` | Text to messages and back, and the framing reader built on jsonrpc-nv's. |
| `dapcmd` | The commands, the adapter's capabilities, and the configuration handshake as a state machine. |
| `dapbrk` | Sources and breakpoints: what the client asked for, and where the breakpoint actually landed. |
| `dapstack` | Threads, stack frames, scopes, variables and evaluations. |
| `dapevent` | The events, and what each one means for what the client is holding. |

## How to choose an entry point

**`dapwire.feed` takes bytes and answers at most one message.**
`dapwire.take` answers the next message already buffered, adding
nothing. A read that delivered three messages gives one from `feed`.

**`dapcmd.command_of` is where an adapter's dispatcher starts.** It
turns a command name into one arm per command this package types, plus
an arm carrying the name of anything else.

**`dapcmd.allows` is the handshake in one call.** Ask it before handling
anything, and `dapcmd.may_complete_launch` before starting the program.

**`dapmsg.failed` is how a request is refused.** It carries the same
`command` and `request_seq` as a success, so the client's table is
cleared either way.

## The rules a user needs

1. **`seq` is per sender, not per request.** Each party numbers its own
   messages from 1, and every message takes the next number — responses
   and events included. A response is tied to its request by
   `request_seq` and by nothing else.
2. **The sequence starts at 1.** A sender that started at 0 sends a
   message whose number the specification says no message has.
3. **A failed response is still a response.** An adapter that answers a
   failure by sending nothing leaves the client waiting for ever.
4. **An error is a format string and its variables, sent separately.**
   An adapter that interpolates before sending throws away the client's
   ability to localise the message and to group identical errors.
   `showUser` decides whether it is shown at all.
5. **The message's kind is read from its `type` member.** Inferring it
   from which members are present is wrong for a response, which has
   both a `command` and a `success`.
6. **`initialized` is an event, and it arrives after `launch` was
   sent.** The client sends its breakpoints in response to it, and the
   adapter completes the launch only after `configurationDone`. An
   adapter that started the program on `launch` runs past every
   breakpoint the client had not sent yet, and only on a fast machine.
7. **`setBreakpoints` replaces every breakpoint in that source.** There
   is no add and no remove. A client that sends one breakpoint to add
   one has deleted the others.
8. **The breakpoint response corresponds one to one, in order, with the
   request.** A breakpoint has no identifier going in, so that
   correspondence is the only thing tying a result to a request. An
   adapter that drops an unverifiable breakpoint instead of returning it
   unverified shifts every breakpoint after it.
9. **A breakpoint moves.** The response's `line` is where it actually
   is, which for a blank line or a comment is not where it was asked
   for. The client redraws from the response.
10. **Line numbers are one-based**, unlike the Language Server
    Protocol's.
11. **A source with a non-zero `sourceReference` is not on disk**, and
    the reference wins over any path beside it. The content is fetched
    with a `source` request.
12. **A variables reference of 0 means a leaf.** Any other value is a
    handle the adapter issued.
13. **Every variables reference dies when execution resumes.** A
    `continue`, a `next`, a `stepIn` or a `stepOut` makes every handle
    meaningless, and the adapter may reuse the numbers. A client that
    keeps its expanded tree shows the user other values, with no error,
    because the numbers are still valid and simply mean something else.
14. **A thread id is the exception**: it is stable for the thread's life
    and stepping does not invalidate it.
15. **A `stopped` event's `threadId` decides what everything afterwards
    is about**, and `allThreadsStopped` decides whether the client may
    ask about another thread at all.
16. **An output event's category decides where the text goes.**
    `stdout` and `stderr` are the program's and belong in its terminal;
    `console` is the adapter's own; `telemetry` is not shown.
17. **`exited` is not `terminated`.** A program can exit and leave the
    adapter alive to be restarted. A client that shuts the session down
    on `exited` loses the restart.
18. **A tracepoint does not stop the program.** A breakpoint with a log
    message prints and continues.
19. **A `hover` evaluation must have no side effects.** The user only
    moved the mouse. An adapter that calls a function on a hover runs
    the user's code every time the cursor passes over it.

## What is not included

The protocol is large and this package carries the commands a debugger
for a compiled language needs. These are named rather than left to be
discovered:

- **Exception handling** — `setExceptionBreakpoints`,
  `exceptionInfo`, and the exception filters and options.
- **The other breakpoint kinds** — `setFunctionBreakpoints`,
  `setDataBreakpoints`, `setInstructionBreakpoints`,
  `breakpointLocations`.
- **Changing state** — `setVariable`, `setExpression`, `goto`,
  `gotoTargets`, `restartFrame`.
- **Termination and restart** — `terminate`, `restart`,
  `terminateThreads`.
- **Memory and disassembly** — `readMemory`, `writeMemory`,
  `disassemble`, `stepInTargets`, and stepping by instruction.
- **Source fetching** — the `source` request itself. `DapSource`
  carries the reference and `needs_fetch` says when one must be
  fetched; making the request is the adapter's.
- **Modules and loaded sources** — `modules`, `loadedSources`, and
  their events.
- **Reverse debugging** — `stepBack`, `reverseContinue`.
- **Completions in the debug console** — `completions`.
- **The adapter's requests to the client** — `runInTerminal`,
  `startDebugging`. The envelope supports them, because a request may
  go either way; the argument types are not here.
- **`cancel`** and the progress events.

All of them arrive as `DapUnknownCmd` or `DapUnknownEvent` carrying the
name, so an adapter that implements one reads the arguments itself and
this package stays out of the way.

Also not included: **a transport**, **a debugger**, and **a value
formatter**. An adapter reads its own pipe, drives its own debugger, and
renders values itself — a variable's `value` is text the adapter
produced, because the adapter knows how the language prints a value and
the client does not.

## Related packages

- [jsonrpc-nv](https://novo-lang.org/packages/jsonrpc-nv) supplies the
  `Content-Length` framing. This package depends on it for that and
  nothing else: the Debug Adapter Protocol looks like JSON-RPC and is
  not, so the message envelope is declared here.
- [lsp-nv](https://novo-lang.org/packages/lsp-nv) is the Language Server
  Protocol, behind the same framing. Its positions are zero-based and in
  negotiated code units; this protocol's lines are one-based.

## Tests

```bash
novo test tests/dapmsg_tests.nv       # the envelope, the codec and the framing
novo test tests/dapsession_tests.nv   # the handshake, breakpoints, stack, events
```

The normative source is the Debug Adapter Protocol specification, and
each assertion names the rule it comes from. The reference
implementations are the TypeScript package `@vscode/debugprotocol`, for
the shape of the types, and the Python package `debugpy`, for the
handshake.

The suite asserts that the sequence is per sender and starts at 1, that
a response is tied to its request by `request_seq` and the command, that
a failed response carries both, that an error keeps its format string
and its variables apart, that the message kind is read from `type`, that
a launch is not completed before `configurationDone`, that a breakpoint
response reports where the breakpoint landed and corresponds one to one
with the request, that a reference of 0 is a leaf, that resuming
invalidates every reference, and that `exited` does not end the session.

The tests compile today and fail at run, each on the
`not implemented: dap-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `dapmsg` — the envelope and the sequence | the types are declared; every body is a `todo()` |
| `dapwire` — the codec and the framing | the types are declared; every body is a `todo()` |
| `dapcmd` — commands, capabilities, handshake | the types are declared; every body is a `todo()` |
| `dapbrk` — sources and breakpoints | the types are declared; every body is a `todo()` |
| `dapstack` — threads, frames, scopes, variables | the types are declared; every body is a `todo()` |
| `dapevent` — the events | the types are declared; every body is a `todo()` |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
