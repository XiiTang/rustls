# Boundless TLS channel binding

Base: upstream rustls v0.23.45 (2976d90fd1c2db6b518700dd101b714069cfcb17).
This includes the upstream TLS 1.3 record-boundary correction in 0.23.45.

The channel-binding patch exposes `CommonState::tls_unique()` for a completed TLS 1.2
handshake using Extended Master Secret. It retains the first Finished value,
including the server-first Finished on resumed sessions. TLS 1.3 and TLS 1.2
without EMS return no tls-unique value. Boundless uses this for SASL channel
binding; its existing TLS 1.3 exporter path is unchanged.

Regression evidence (macOS):

```
cargo test -p rustls --locked --no-default-features --features ring,std,tls12 --test api
```

224 API tests pass, including `runtime_tls_unique_matches_peer_and_changes_for_resumption`
and `client_rejects_encrypted_extensions_in_server_hello_record`.
The Boundless repository separately exercises independent OpenSSL peers.

## TLS 1.2 record exhaustion (2026-10-03)

Keep the stable 0.23.45 baseline. Upstream issue #3301 and main-line #3313
(453374fa) identify the writer failure, but returning an error only when zero
bytes were accepted does not repair a write that straddles exhaustion.

Count only successfully encrypted fragments. Buffered and vectored writes return
the accepted prefix; a connection-level exhausted flag rejects all subsequent
writes and flushes. Unbuffered encryption checks every fragment before accepting
the write and cannot bypass exhaustion on a later call. Processing a handshake
that exhausts previously buffered plaintext surfaces EncryptError. The reserved
close_notify record remains sendable below the hard sequence limit. Preserve
TLS 1.3's existing single KeyUpdate trigger and tls_unique/EMS behavior.

The lowered-limit regression sends a 64 KiB ordinary/vectored write with only
two records left: both report 32768 bytes, and the peer receives exactly that
prefix followed by close_notify. Later writes, vectored writes and flush fail.
224 API and 27 unbuffered tests pass with ring,std,tls12 on macOS arm64,
including automatic TLS 1.3 key updates and channel-binding resumption. No
automatic reconnect or replay is introduced.
