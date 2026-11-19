# Errors

A failed call ends with a gRPC status, never with an error field in a response:

- **status code**: the broad class (below), for generic handling and logs.
- **status message**: a human-readable sentence, safe to show to the user.
- **`grpc-status-details-bin`** trailer: a `google.rpc.Status` whose `details` holds one
  `google.rpc.ErrorInfo` with
  - `domain = "raindrop"`
  - `reason =` the error code, e.g. `username_taken`. **This is what clients branch on.**

  Codes are open strings: a client that does not know one shows the message.

The `google/rpc/status.proto` and `google/rpc/error_details.proto` definitions come from
[googleapis](https://github.com/googleapis/googleapis). They are not imported by any file here, because no
Raindrop message contains them; each side generates or ships them for decoding the trailer.

## Status codes

The server's domain layer throws `ApiException` with a kind; the gRPC layer maps the kind to a status:

| Kind | gRPC status | Typical reasons |
|---|---|---|
| bad request | `INVALID_ARGUMENT` | `invalid_username`, `invalid_mls_message`, `message_too_long` |
| unauthorized | `UNAUTHENTICATED` | `missing_token`, `invalid_session`, `invalid_credentials` |
| forbidden | `PERMISSION_DENIED` | `missing_permission`, `device_required`, `account_left` |
| not found | `NOT_FOUND` | `channel_not_found`, `attachment_not_found` |
| conflict | `FAILED_PRECONDITION` | `username_taken`, `epoch_conflict`, `stale_epoch`, `upload_offset_mismatch`, `camera_limit_reached`, `screen_share_limit_reached` |
| too large | `RESOURCE_EXHAUSTED` | `chunk_too_large` |
| unavailable | `UNAVAILABLE` | `voice_unavailable` |
| anything unexpected | `INTERNAL` | `internal_error` |

Clients must not branch on the status code where a reason exists. Several reasons share a status, and
the reason is the contract.

## How the event stream ends

`EventService.Connect` never ends with OK while the server is running. Its final status says what to do:

| Reason | Status | Client should |
|---|---|---|
| `session_ended` | `UNAUTHENTICATED` | stop and ask for the password again |
| `invalid_session`, `missing_token` | `UNAUTHENTICATED` | the same, when the stream was refused at the start |
| `too_slow` | `RESOURCE_EXHAUSTED` | reconnect (it fell more than the outbox behind) |
| `server_stopping` | `UNAVAILABLE` | reconnect with backoff |
| `connection_lost` | `UNAVAILABLE` | reconnect; no heartbeat arrived within the timeout |
| `invalid_frame` | `INVALID_ARGUMENT` | a client bug: it sent something other than a heartbeat or resubscribe |

A transport failure with no `ErrorInfo` at all (network gone, TLS refused) is handled by the client's own
connection logic: backoff for the network, and certificate-change handling for a refused pin.

## Reasons

These are every reason the server produces today, by area. The exact meaning of each is documented with
the endpoint that produces it in `RaindropServer/docs/`.

**Requests in general**
`invalid_payload` (a malformed field, e.g. an id that is not a UUID), `missing_identifier`,
`invalid_frame` (a stream frame out of order), `internal_error`

**Sessions and accounts**
`missing_token`, `invalid_session`, `invalid_credentials`, `invalid_user_id`, `invalid_username`,
`weak_password`, `username_taken`, `already_registered`, `already_active`, `account_preserved`,
`account_left`, `no_preserved_account`, `invalid_challenge` (a sign-in nonce that is unknown, used, lapsed
or issued for another UUID), `auth_unavailable` (the server has no identity key for devices to sign
against), `auth_busy` (too many sign-in challenges outstanding), `rate_limited` (too many session-less calls from
one address this minute; `RESOURCE_EXHAUSTED`), `authenticator_required`, `invalid_authenticator_code`,
`authenticator_locked` (too many wrong codes; wait), and for a device waiting to get in without an
authenticator code (`RequestRecovery`): `recovery_pending` (the wait is still running; `CompleteAuth`
without a code), `recovery_cancelled` (one of the account's devices cancelled this device's wait),
`recovery_unavailable` (this server lets nobody in without a code), `recovery_not_needed` (the account has
no authenticator, so the device can simply sign in), `too_many_recoveries` (five devices are already
waiting)

**Roles and members**
`invalid_role_name`, `role_name_taken`, `role_not_found`, `invalid_permission`, `role_immutable`,
`role_not_assignable`, `last_owner`, `member_not_found`, `missing_permission`

**Channels and messages**
`invalid_channel_type`, `channel_type_unsupported`, `invalid_channel_name`, `channel_name_taken`,
`channel_not_found`, `wrong_channel_type`, `invalid_message`, `message_too_long`, `message_not_found`,
`invalid_cursor`, `encryption_required`, `not_encrypted`, `invalid_reason`, `invalid_attestation`

**End-to-end encryption**
`device_required`, `invalid_certificate`, `identity_mismatch`, `device_exists`, `device_not_found`,
`too_many_devices`, `too_many_users`, `no_challenge`, `invalid_signature`, `invalid_backup`,
`invalid_link`, `link_not_found`, `invalid_vault`, `vault_not_found`, `invalid_authenticator`,
`invalid_pairing`, `pairing_not_found`, `pairing_busy`, `invalid_mls_message`, `wrong_group`, `group_not_found`, `group_exists`,
`epoch_conflict`, `stale_epoch`, `not_allowed_in_group`, `too_many_key_packages`, `invalid_welcome`,
`certificate_mismatch`

**Attachments**
`attachments_disabled`, `invalid_attachment`, `file_type_not_allowed`, `file_too_large`,
`too_many_pending_uploads`, `upload_offset_mismatch`, `chunk_too_large`, `upload_incomplete`,
`upload_finished`, `hash_mismatch`, `too_many_attachments`, `attachment_already_sent`,
`attachment_not_found`, `attachment_unavailable`, `invalid_offset` (a download asked to start past the
end of the file)

**Voice**
`voice_unavailable`, `channel_full`, `not_in_voice`, `camera_limit_reached` (every camera place in the
call is taken; turning a camera on without the `video` permission is `missing_permission` instead),
`screen_share_limit_reached` (every sharing place in the call is taken; sharing without the
`screen_share` permission is `missing_permission` instead). A share with no kind, or a screen share
asking for sound, is `invalid_payload`.

**Event stream**
`session_ended`, `too_slow`, `server_stopping`, `connection_lost`

## Client-side codes

A client also produces codes of its own that never come from a server: `certificate_mismatch` (the pin
refused the certificate), `network_error` and `unknown_error`. They share the same `code` field in the
client's `ApiException`, so UI code treats every failure the same way.
