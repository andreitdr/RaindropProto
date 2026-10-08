# RaindropProto

The Raindrop wire protocol: gRPC services and protobuf messages, shared by
[RaindropServer](../RaindropServer) and [RaindropClient](../RaindropClient).

Everything a client and a server say to each other is in `raindrop/v1/`, over gRPC on HTTP/2 and TLS 1.3,
with the server's self-signed certificate pinned on first use. The one exception is voice media, which is
WebRTC through LiveKit. Its signalling is relayed by the server as a WebSocket, because that is the
protocol LiveKit's client SDK speaks.

| File | Service | What it covers |
|---|---|---|
| `server.proto` | `ServerService` | The handshake: protocol version, TLS fingerprint, identity key, capabilities |
| `users.proto` | `UserService` | Registration, login, sessions, profile, leaving |
| `members.proto` | `MemberService` | The roster and presence |
| `roles.proto` | `RoleService` | Roles and permissions |
| `channels.proto` | `ChannelService` | Channel lifecycle |
| `text.proto` | `TextChannelService` | Messages, history, deletion, reports |
| `voice.proto` | `VoiceService` | Voice call control (the media is WebRTC) |
| `devices.proto` | `DeviceService` | E2EE identity: devices, session binding, key backup, device linking |
| `delivery.proto` | `DeliveryService` | The MLS delivery service: groups, commits, key packages, Welcomes |
| `attachments.proto` | `AttachmentService` | Encrypted file upload (client stream) and download (server stream) |
| `events.proto` | `EventService` | The bidirectional push stream |
| `payload.proto` | — | The plaintext inside an end-to-end encrypted message (clients only) |

Start with [`docs/`](docs/README.md).
