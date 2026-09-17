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
connections must prove ownership of the node key that signed the ENR. How the proof is
created depends on the peer types. The server distinguishes the two cases by the presence
of a nonce in the CONNECT request.

#### Browser Connections

The dialer generates a random 32 byte nonce and sends it in the CONNECT request, in the
`WT-Available-Protocols` header. Here the nonce is hex-encoded string.

The server opens the first bidirectional stream and sends the `id-proof` as its first
message.

    id-proof = sign(nodekey, "ENR-key-proof-v1" || nonce)

The dialer recovers the public key from the signature and checks it against the node key
in the ENR it dialed.

#### Node-to-Node Connections

The dialer omits the nonce from the CONNECT request. Instead, both sides must create a
binding signature over the negotiated key material of the QUIC connection.

To get the key material, use a 'key exporter' with the `ENR key binding v1` label and a
length of `32`.

Both sides send the `id-proof` as their first message on stream zero. The dialer sends
first, without waiting for the proof of the server.

    id-proof = sign(nodekey, "ENR-key-proof-v1" || tls-exported-key)

Both sides must verify the `id-proof` of the peer. The dialer checks the recovered public
key against the node key in the ENR it dialed. The server has no ENR for the dialer, and
uses the recovered public key as the node ID of the peer.
