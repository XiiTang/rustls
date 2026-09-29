# Boundless TLS channel binding

Base: upstream rustls v0.23.45 (2976d90fd1c2db6b518700dd101b714069cfcb17).
This includes the upstream TLS 1.3 record-boundary correction in 0.23.45.

The sole code patch exposes `CommonState::tls_unique()` for a completed TLS 1.2
handshake using Extended Master Secret. It retains the first Finished value,
including the server-first Finished on resumed sessions. TLS 1.3 and TLS 1.2
without EMS return no tls-unique value. Boundless uses this for SASL channel
binding; its existing TLS 1.3 exporter path is unchanged.

Regression evidence (macOS):

```
cargo test -p rustls --locked --no-default-features --features ring,std,tls12 --test api
```

223 tests pass, including `runtime_tls_unique_matches_peer_and_changes_for_resumption`
and `client_rejects_encrypted_extensions_in_server_hello_record`.
The Boundless repository separately exercises independent OpenSSL peers.
