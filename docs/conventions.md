# Conventions

## Packages and files

- One package, `raindrop.v1`, in `raindrop/v1/`. A breaking redesign would be `raindrop.v2` beside it,
  served alongside v1 for as long as clients need it. It would not be an edit to v1.
- One service per domain, one domain per file. `payload.proto` is the exception: it has no service,
  because it describes the plaintext inside an MLS message, which never crosses the wire unencrypted.
- `option csharp_namespace = "RaindropServer.Grpc.Protocol"` on every file, so the server's generated
  types never collide with its domain types of the same name (`UserService`, `Channel`, …).

## Requests and responses

- Every RPC has its own `<Rpc>Request` and `<Rpc>Response` (buf STANDARD), including empty ones, so either
  can grow a field without affecting another RPC.
- A response that returns one entity wraps it (`CreateChannelResponse { Channel channel = 1; }`), which
  leaves room for metadata next to it later.
- Lists are returned in full. Message history is the only paged call: it uses an opaque
  `next_cursor` string that clients pass back untouched and never parse.
- Nothing sensitive goes in metadata except the session token (`authorization: Bearer <token>`).
  Usernames, ids and passwords travel in request bodies, inside TLS.

## Types

| Kind of value | Wire type | Why |
|---|---|---|
| UUID | `string`, canonical lower case | Certificates and attestations sign ids as text, so the exact text matters |
| Instant | `google.protobuf.Timestamp` | |
| Duration | `google.protobuf.Duration`, or `int64 …_ms` inside signed or client-only payloads | |
| Opaque crypto blob (MLS message, certificate, signature, backup, ciphertext hash) | `bytes` | Produced and checked by RaindropCrypto; the server never parses most of them |
| Size, count, epoch | `int64` (sizes, epochs), `int32` (small counts, positions) | |

## Open and closed sets

**Open sets are strings.** A plugin or a newer build can add to them, and a client ignores or
generically renders a value it does not know:
- permission names (`view_channel`, …), driven by the server's `PermissionNames` table. Each server
  build lists the ones it knows in `RoleService.ListPermissions`, with a `title` and a `category` (itself
  an open set: `general`, `text`, `voice`, `channels`, `moderation`, `administration`), so a role editor
  draws only from that list and can show a permission the client was never built with. A server older
  than those two fields sends neither: derive a title from the name and file it under "other". A role
  carries only the names its server's build knows; one saved by a newer build keeps the rest on the
  server, unsent and ungranted, until that build runs again
- channel types (`text`, `voice`)
- voice leave reasons and transport names
- capability names
- payload types (`text`, `files`)
- plugin event types

**Closed sets are enums** that only a protocol change can extend: `Presence`, `UserState`, `RoleKind`,
`AttachmentKind` and `UploadState`.
- Each has `<ENUM>_UNSPECIFIED = 0`. proto3 enums are open, so a value from a newer server decodes to
  something a client does not recognise, which it treats as `UNSPECIFIED`.
- `Presence` is sorted by the client's own display order, never by its numbers. A new presence value
  takes the next free number, whatever its place in a roster.

## Absent versus empty

proto3 scalars and repeated fields cannot say "absent". Where absent means something different from
empty, the field is `optional` (scalars) or a wrapper message (lists):

| Field | Absent means | Empty means |
|---|---|---|
| `UpdateRoleRequest.permissions` (`PermissionSet`) | leave the role's permissions alone | the role grants nothing |
| `AttachmentPolicy.allowed_extensions` (`ExtensionList`) | any extension that is not blocked | no extension is allowed |
| `ReportMessageRequest.attested_files` (`AttestedFiles`) | a v1 attestation | a v2 attestation over no files |
| Any `optional` scalar | not set / unknown / not applicable | — |

## Evolving the schema

- Add fields with new numbers. Never renumber, retype or reuse a number. Delete a field with
  `reserved <number>; reserved "<name>";`.
- Renaming a field or message does not change the wire, but it breaks the generated code on both sides.
  Do it only as a coordinated server and client change.
- New built-in events get the next free `ServerEvent` number below 100. Number 100 is `PluginEvent`; leave
  room around it.
- `buf lint` must pass. `buf breaking --against '.git#branch=main'` uses WIRE rules, because only
  the binary encoding reaches clients already in the field.
- Bump `GetServerInfoResponse.protocol_version` only for a change old clients cannot survive. Additive
  features are announced as a new capability name instead.
