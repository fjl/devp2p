# ethp2p QUIC transport

This document specifies a QUIC-based protocol that nodes on the execution layer
implement to enable connectivity from browsers, and between each other.

## Connection Setup

### TLS Certificate

Implementations must generate a new, self-signed TLS certificate with a lifetime of two
weeks. If the node is to run for longer than two weeks, a new certificate must be created
and announced (see below) ahead of time.

Certificates should support key material for a P256-curve based key exchange. The TLS
configuration must include support for the `TLS_AES_128_GCM_SHA256` cipher suite, and may
include support for more cipher suites.

### Certificate Hashes in Discovery

For node implementations that support QUIC, the ENR must include the SHA256 hashes of the
current certificate (and next certificate). This is to be stored in the `qh` ENR key,
which should have a size of either 32 bytes (for one certificate) or 64 bytes (for two
certificates).

The certificate hashes are computed as the SHA256 hash of the DER encoding of the
certificate. This feature exists primarily for compatibility with WebTransport, but
non-browser implementations should also verify the presented certificate upon connection
to an ENR.

### Node ID Binding

Since the public key used by TLS does not match the key used for signing the ENR, fresh
connections must prove ownership of the node key that signed the ENR.

To do this, the dialer generates a random 32 byte nonce and sends it in the CONNECT
request, in the `WT-Available-Protocols` header. Here the nonce is hex-encoded string.

The server opens the first bidirectional stream and sends the `id-proof` as its first
message.

    id-proof = sign(nodekey, "ENR-key-proof-v1" || nonce)

The dialer recovers the public key from the signature and checks it against the node key
in the ENR it dialed. For connections between nodes, the server may challenge the dialer
the same way (TBD)
