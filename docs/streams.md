# Streams

Three RPCs stream. Everything else is a plain unary call.

## `EventService.Connect`: the push channel

The stream is bidirectional, and each signed-in client keeps one open per server.

```
client                                   server
  │── Connect (authorization: Bearer …) ──▶│
  │◀──────────── ready ────────────────────│  user id, protocol version, heartbeat_interval
  │◀──────────── events … ─────────────────│
  │── heartbeat ──────────────────────────▶│  every heartbeat_interval
  │── resubscribe ────────────────────────▶│  e.g. after the app resumes
  │◀──────────── ready ────────────────────│  client re-reads everything
  │◀─────── status + ErrorInfo.reason ─────│  how it ended (see errors.md)
```

- **`ready` comes first**, before any other event, and again after every `Resubscribe`. Nothing is
  replayed, so a client re-reads whatever it shows after each `ready`. That is also how a reconnect
  catches up.
- **Events are notifications.** Each carries the full current state of what it is about (a whole
  `ChannelMessage`, a whole `VoiceParticipant`), so a client merges by id. An event a client does not
  understand is a `oneof` case it does not know, and it ignores it.
- **Audiences** are decided per event by the same permission checks as every call. A channel the caller
  cannot see produces no events at all.
- **Heartbeats** are sent every `ready.heartbeat_interval`. They do two jobs:
  - They keep the session alive (it otherwise lapses after its idle timeout).
  - They make presence honest. A member is online while they have a live stream. A stream with no
    heartbeat for the server's timeout (a few intervals) is ended as `connection_lost`, and the member
    goes offline.
- **The server never waits for a slow client.** Each stream has a bounded outbox. A client that falls
  behind it is cut off with `too_slow`, and reconnecting re-reads what it missed.
- **The client sends only heartbeats and resubscribes.** Actions (sending a message, committing,
  joining a call) stay unary calls, where errors and retries are per action.

## `AttachmentService.Upload`: client-streaming

```
UploadRequest{header: {attachment_id, offset}}
UploadRequest{data: ≤ upload_frame_bytes}
UploadRequest{data: …}
⇒ UploadResponse{status}
```

- The header comes first, exactly once. Its `offset` must equal the upload's `received_size`; otherwise
  the call fails with `upload_offset_mismatch`.
- Each data frame is written as it arrives. If the stream breaks, what arrived is kept: the client asks
  `GetUploadStatus` and opens a new stream from `received_size`.
- A frame larger than `upload_frame_bytes` is refused with `chunk_too_large`. Sending more than the
  declared `ciphertext_size` is refused too.
- `CompleteUpload` then checks the whole ciphertext's SHA-256.

## `AttachmentService.Download`: server-streaming

```
DownloadRequest{attachment_id, offset}
⇒ DownloadResponse{header: {ciphertext_size, offset}}
⇒ DownloadResponse{data: …} …
```

- A client resuming a download asks from the size of the partial file it already has, and appends.
- Every failure is `attachment_not_found`, whether the file never existed, is in a channel the caller
  cannot see, or is not attached yet. Ids therefore cannot be probed.
