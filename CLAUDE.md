# go-psrpcore

Sans-IO PSRP protocol implementation. No transport. Consumers provide io.ReadWriter.

## Package Layout

runspace/      — Pool lifecycle state machine (main package)
pipeline/      — Pipeline execution state machine
fragments/     — Fragmenter + Assembler
messages/      — 41 PSRP message types, encode/decode
serialization/ — CLIXML encode/decode
objects/       — PSObject, PSCredential, SecureString, ErrorRecord
host/          — Host callback interface (Read-Host, prompts)
outofproc/     — OutOfProcess transport framing (stdin/stdout)

## Key Types

**runspace.Pool** — central type:
- `New(transport, id)` — transport is io.ReadWriter
- `Open(ctx)` — full handshake (send SESSION_CAPABILITY, INIT_RUNSPACEPOOL, wait for Opened). Do NOT call after CreateShell with creationXml.
- `ReceiveHandshakeWSMan(ctx)` — receive-only wait for Opened. Call this after CreateShell.
- `GetHandshakeFragments()` → `[]byte` — SESSION_CAPABILITY + INIT_RUNSPACEPOOL fragments for creationXml. Calling this increments objectIDs 1 and 2 in the fragmenter.
- `CreatePipeline(cmd)` / `CreatePipelineBuilder()` → `*pipeline.Pipeline`
- `SendMessage(ctx, msg)` — low-level
- `SetTransport(t)` — dynamic transport swap (reconnect)

**Pool states**: `BeforeOpen → Opening → Opened → Closing → Closed` (Broken from any)

**fragments.Fragmenter**:
- Fragment header: 21 bytes — ObjectID (8B BE), FragmentID (8B BE), Flags (1B), BlobLength (4B BE)
- `objectID` increments on each `Fragment()` call — IDs are consumed permanently
- Start flag=0x1, End flag=0x2, both=0x3 (single-fragment message)

**messages.Message**:
- Header: 40 bytes — Destination (4B LE), Type (4B LE), RunspaceID (16B .NET GUID), PipelineID (16B .NET GUID)
- .NET mixed-endian GUID: first 3 groups LE, last 2 groups BE (not RFC 4122)
- Key types: `SESSION_CAPABILITY`, `INIT_RUNSPACEPOOL`, `RUNSPACEPOOL_STATE`,
`CREATE_PIPELINE`, `PIPELINE_OUTPUT`, `PIPELINE_STATE`, `ERROR_RECORD`, `SIGNAL`

**pipeline.Pipeline**:
- `Invoke(ctx)` — executes pipeline
- `GetCreatePipelineData()` → `[]byte` — serialized CREATE_PIPELINE for embedding in Command request
- Output channels: `Output()`, `Error()`, `Warning()`, `Verbose()`, `Debug()`,
`Progress()`, `Information()`

## Fragment Byte Order

Fragments: **big-endian** (network byte order)
Messages: **little-endian** (.NET convention)