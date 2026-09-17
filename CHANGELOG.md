# Changelog

All notable changes to dap-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `dapmsg` — the load-bearing interface, and the decision is that the
  ENVELOPE IS DECLARED HERE rather than borrowed from jsonrpc-nv. The
  Debug Adapter Protocol travels behind the same `Content-Length` header
  as the Language Server Protocol, so the framing is taken; the message
  inside is a different protocol and typing it as a `JrpcRequest` would
  be wrong in three places at once. `seq` is per SENDER and every
  message consumes one, events included, so a response is tied to its
  request by `request_seq` alone. There are three message types, not
  two, and an event is not a notification: it has a sequence number and
  comes only from the adapter. Failure is a `success` boolean with a
  structured `Message` in the body, not an error object — and that
  message keeps its format string and its variables apart, so a client
  can localise it and group identical errors.
- `dapwire` — the framing is jsonrpc-nv's `jrpcframe` with a decode on
  the end, and `is_framing_fault` keeps the distinction that decides
  whether an adapter answers or closes. `decode` reads the `type` member
  and refuses anything else: inferring the kind from which members are
  present is wrong for a response, which has both a `command` and a
  `success`, and it is wrong silently.
- `dapcmd` — the configuration handshake as a state machine, because its
  order is not the one a reader expects. `initialized` is an EVENT and
  it arrives after `launch` was sent, so
  `phase_after_initialized_event` is a separate function from
  `phase_after`: a state machine that watched only commands never leaves
  `DapInitialized`. `may_complete_launch` is the question an adapter
  that started the program on `launch` never asked, and that bug shows
  up only on a fast machine.
- `dapbrk` — two types, because what the client asked for and what the
  adapter did about it differ constantly. `was_moved` and `corresponds`
  are the two checks a client needs before it redraws: a breakpoint on a
  blank line moves, and an adapter that dropped an unverifiable
  breakpoint from its response rather than returning it unverified
  shifted every breakpoint after it. A source's `source_reference` wins
  over its path, because the path of generated code names a file that
  may hold something else.
- `dapstack` — the variables reference as a handle with a stated
  lifetime. `is_live_reference` reads better than `!= 0` at a call site
  and stops a zero going into a table of handles;
  `dapevent.invalidates_references` and `dapcmd.resumes_execution` are
  the two questions a client asks before it keeps an expanded tree. A
  variable's `value` is TEXT the adapter rendered, because the adapter
  knows how the language prints a value and the client does not.
- `dapevent` — what each event means for what the client is holding.
  `ends_session` reports `terminated` and not `exited`, because a
  program can exit and leave the adapter alive to be restarted.
  `is_program_output` separates the program's streams from the adapter's
  own messages, which an adapter that sent stdout as `"console"` has
  mixed together for good.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  dap-nv.<module>.<fn>`.
- **jsonrpc-nv is a PATH dependency in this staged release.** Nothing is
  on the registry yet, so a sibling directory is the only thing that
  resolves. The publish round rewrites it to `^0.0.1` before the tarball
  is built.
- **The command set is scoped, and what is left out is named** in the
  README's "What is not included" — exception handling, function and
  data breakpoints, `setVariable`, memory and disassembly, modules,
  reverse debugging, and the adapter's own requests to the client among
  them. Each arrives as `DapUnknownCmd` or `DapUnknownEvent` carrying
  its name, so an adapter that implements one reads the arguments
  itself.
