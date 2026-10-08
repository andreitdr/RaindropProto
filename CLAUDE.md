# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Documentation wiki

[`docs/`](docs/README.md) is a per-topic documentation wiki for this project, meant to stay current.
**Whenever you change a `.proto` file — a new RPC, a new field, a new event, a new error reason —
update the matching file(s) under `docs/` as part of that same change**, not as a follow-up.

RaindropProto is the wire protocol of Raindrop, a decentralised, self-hosted chat platform: every
message a client and a server exchange is defined here, and nowhere else. It is shared by:

- [RaindropServer](../RaindropServer/CLAUDE.md) (.NET), which compiles these files with `Grpc.Tools`
  (server stubs) inside `RaindropServer.Grpc`, and implements every service there.
- [RaindropClient](../RaindropClient/CLAUDE.md) (Flutter), which generates Dart from these files into
  `lib/core/network/grpc/generated/` and commits the result.

A change here is a change to both. Regenerate the client's Dart, rebuild the server, and update the
matching docs in **both** projects (`RaindropServer/docs/`, `RaindropClient/docs/`) in the same change.
[RaindropCrypto](../RaindropCrypto/CLAUDE.md) never sees this protocol: its byte formats travel inside
these messages as opaque `bytes` (see its `docs/wire-contracts.md`).

## Commands

From this folder:

```bash
buf lint
```

```bash
buf format -w
```

Once there is a commit on `main` to compare against, check that a change does not break clients already
in the field:

```bash
buf breaking --against '.git#branch=main'
```

Regenerate the client's Dart (needs `protoc` and `dart pub global activate protoc_plugin`; the script
lives in the client):

```bash
../RaindropClient/tool/gen_protos.sh
```

The server needs nothing run by hand: `dotnet build` regenerates its C#.

## Layout and rules

Everything is package `raindrop.v1`, in `raindrop/v1/`, one file per domain and one service per file
(`payload.proto` has no service: it is the plaintext *inside* an encrypted message, which only clients
read). Full conventions: [`docs/conventions.md`](docs/conventions.md). The ones that matter most:

- **Wire compatibility is permanent.** Never renumber, retype or reuse a field number; delete a field by
  `reserved`-ing its number and name. Renaming is safe on the wire but breaks generated code on both
  sides, so it is a coordinated change. `buf breaking` (WIRE rules) is the check.
- **Every RPC has its own `<Rpc>Request` and `<Rpc>Response`**, even when empty (buf STANDARD), so either
  can grow a field later without touching another RPC.
- **UUIDs are canonical lower-case strings**, never 16-byte `bytes`: device certificates and message
  attestations sign ids *as text*, and a reformatted id would not verify.
- **Open sets are strings, closed sets are enums.** Permissions, channel types, voice leave reasons,
  transport names, capabilities and payload types can be extended by plugins or a newer build and stay
  strings that a client ignores when unknown. Presence, user state, role kind, attachment kind and upload
  state are enums with a `*_UNSPECIFIED = 0` that an unknown value decodes to.
- **Presence carries meaning: use `optional` or a wrapper message.** Absent and empty must differ in
  `UpdateRoleRequest.permissions` (`PermissionSet`), `AttachmentPolicy.allowed_extensions`
  (`ExtensionList`) and `ReportMessageRequest.attested_files` (`AttestedFiles`, which selects the v2
  attestation).
- **Errors are not messages.** A failed call is a gRPC status whose `google.rpc.ErrorInfo.reason` is the
  error code clients branch on; see [`docs/errors.md`](docs/errors.md). Never add an `error` field to a
  response.
- **Events are a `oneof`.** A new built-in event is a new case in `ServerEvent` with the next free number;
  plugins publish through `PluginEvent` and never get a case of their own.
