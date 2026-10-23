# Boundless rustls patches

Base: upstream rustls v0.23.46
(`f04804fa13341a9b1dc02b5d1dd8a746f2c4b7fb`). This includes the fixes for
RUSTSEC-2026-0346, RUSTSEC-2026-0347, RUSTSEC-2026-0348 and RUSTSEC-2026-0349,
and the TLS 1.3 record-boundary correction previously supplied by v0.23.45.

## EMS-bound TLS 1.2 channel binding

Rebuild `b179bd2117ae5d768d53558286e2c4eefa972e5a`: upstream still does not
expose `CommonState::tls_unique()`, which Boundless needs for SASL channel
binding. Retain the first Finished value, including the server-first Finished
on resumed sessions. The public accessor returns it only after a completed
TLS 1.2 handshake using Extended Master Secret. TLS 1.3 and TLS 1.2 without EMS
return no value. Boundless's TLS 1.3 exporter path is unchanged.

The accessor is the check point for both client and server handshakes; its
retained state and handshake completion flags are private to rustls.
`runtime_tls_unique_matches_peer_and_changes_for_resumption` checks initial
absence, completed handshakes, resumption and TLS 1.3 absence. Boundless also
exercises independent OpenSSL peers.

## TLS 1.2 record exhaustion

Adapt `191863d776f4f170841082ff2b5cfb14ecb7ef74`. Upstream v0.23.46 now
checks the record limit of every fragment before unbuffered encryption, rejects
sequence numbers at or beyond the soft/hard limits, fuses writes and flushes
after fatal connection errors, and updates TLS 1.3 keys within fragmented
writes. Keep those upstream changes.

The remaining defect is acceptance accounting and persistent exhaustion:
upstream's buffered write crossing the TLS 1.2 limit sends no application data
but reports the entire plaintext length as accepted. The retained 64 KiB
ordinary/vectored regression reproduced on v0.23.46 with only the channel-binding
patch: it reported 65536 bytes instead of the expected delivered 32768-byte
prefix.

Count only successfully encrypted fragments. Ordinary and vectored writes
return that prefix; subsequent writes and flushes return `EncryptError`.
Unbuffered encryption preflights every fragment and retains exhaustion even
when a smaller later write would otherwise fit. A handshake exhausting buffered
plaintext also returns `EncryptError`. The reserved close_notify record remains
sendable below the hard sequence limit. No reconnect or replay is introduced.

`CommonState::check_pre_encrypt_action` is the shared check point for buffered
fragment encryption and unbuffered preflight. The exhaustion state is owned by
CommonState; private encryption methods prevent application-data paths from
bypassing it. `ConnectionCommon::check_error` exposes it to ordinary/vectored
writes and flushes, and packet processing exposes handshake exhaustion.

Drop the old `record_layer.rs` hunk because upstream's stricter comparisons
already contain its hard-limit ordering. Replace the old standalone writer
error helper with upstream's `check_error`. Preserve upstream's TLS 1.3
preflight and within-write key updates instead of rebuilding the old key-update
mechanism. The documentation-only v0.23.45 baseline commit is superseded by
these notes; neither behavior patch can be retired yet.

Regression evidence on macOS arm64 with Rust 1.97.0:

```sh
cargo test -p rustls --locked --no-default-features --features ring,std,tls12 --test api
cargo test -p rustls --locked --no-default-features --features ring,std,tls12 --test unbuffered
cargo test -p rustls --locked --no-default-features --features ring,std,tls12 --lib
```

234 API, 28 unbuffered and 244 unit tests pass. Coverage includes
`tls12_fragmented_writes_report_only_the_prefix_delivered_before_exhaustion`,
`tls12_write_reports_the_prefix_before_confidentiality_limit`,
`tls12_connection_fails_after_key_reaches_confidentiality_limit`,
`tls12_fragmented_encryption_keeps_exhaustion_on_later_calls`,
`runtime_tls_unique_matches_peer_and_changes_for_resumption`, and upstream's
buffered/unbuffered automatic TLS 1.3 key-update and record-limit tests.

## Reject distrusted certificate paths (#312)

Raise the workspace rustls-webpki minimum to 0.103.16 and update both the
workspace and fuzz lockfiles from 0.103.15. This is the smallest patched release
for RUSTSEC-2026-0382: certificate verification must reject paths containing
the distrusted emSign intermediate public keys. No rustls API or Boundless
channel-binding/record-limit patch changes.

Regression coverage: the ring/std/tls12 rustls library, API and unbuffered
suites, plus Boundless TLS, network and HTTP host tests against this pin.
