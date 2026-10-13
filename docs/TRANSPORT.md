# EcoBridge Secure Transport Specification

**Status:** Proposed design, not yet implemented or security-certified  
**Protocol family:** EBP  
**Initial wire version:** 1

## 1. Goals

EcoBridge transport must provide:

- Confidentiality and peer authentication.
- Low overhead for small files.
- Sustained throughput for large files.
- Bounded memory usage and backpressure.
- Safe cancellation, disconnect handling, and resumable transfers.
- Equivalent behavior on Windows and Android.
- Clear separation between networking, file access, and user interface.

A passing build is not proof of interoperability, performance, or security.

## 2. Initial Transport

The first implementation uses a persistent TCP connection protected by TLS 1.3.

Requirements:

- Use maintained TLS implementations and standard cryptographic libraries.
- Require TLS 1.3 for the initial profile.
- Authenticate the remote device against its previously established EcoBridge identity.
- Never disable certificate or peer verification to make a connection succeed.
- Never trust a device based only on BLE discovery, an IP address, or a device name.
- Keep the transport behind an interface so an alternative such as QUIC can be evaluated later.
- Use a bounded connection timeout and explicit connection state.

The Windows and Android implementations must pass the same interoperability tests.

## 3. Device Identity and Pairing

Pairing and transport authentication are separate stages.

1. BLE discovery identifies a possible EcoBridge peer.
2. Both devices perform a reviewed, user-approved pairing ceremony.
3. The devices establish and store persistent identities and trust records.
4. The transport establishes a fresh TLS session.
5. The TLS peer identity is cryptographically bound to the paired EcoBridge identity.
6. Only after verification succeeds may the session access authorized application operations.

The pairing design must specify key agreement, identity binding, user verification, replay resistance, key rotation, revocation, and recovery.

Do not invent a cryptographic handshake. Select reviewed primitives and libraries, document the exact construction, and obtain appropriate security review before claiming the design is secure.

Private keys must never be transmitted as pairing credentials or written to diagnostic logs.

## 4. Connection Lifecycle

Conceptual states:

- `IDLE`
- `CONNECTING`
- `TLS_HANDSHAKE`
- `AUTHENTICATING`
- `READY`
- `CLOSING`
- `CLOSED`
- `FAILED`

Application operations are forbidden before `READY`.

A session enters `READY` only after TLS validation, paired identity verification, protocol negotiation, and authorization checks succeed.

Every state transition must be explicit. Duplicate initialization, unexpected messages, authentication failures, and invalid transitions must be rejected safely.

## 5. Framing

TCP is a byte stream and does not preserve application message boundaries. EcoBridge therefore requires explicit framing.

### Version 1 frame header

The initial fixed header is 24 bytes, encoded in network byte order:

| Offset | Size | Field |
|---|---:|---|
| 0 | 4 bytes | Magic: ASCII `EBP1` |
| 4 | 1 byte | Wire version |
| 5 | 1 byte | Frame type |
| 6 | 2 bytes | Flags, reserved bits initially zero |
| 8 | 4 bytes | Stream ID |
| 12 | 4 bytes | Sequence number |
| 16 | 4 bytes | Payload length |
| 20 | 4 bytes | Reserved, must be zero |

The header is followed by exactly the declared number of payload bytes.

All integer fields use big-endian encoding. Receivers must read the complete header before processing the payload.

Rules:

- Reject incorrect magic, unsupported versions, unknown frame types, nonzero reserved fields, invalid flags, and illegal state transitions.
- Validate payload lengths against the limit for that frame type before allocating memory.
- Do not permit integer overflow during length calculations.
- Reject truncated frames and unexpected end-of-stream.
- Do not attempt to resynchronize an authenticated stream by searching arbitrary file bytes for magic.
- TLS protects transport records; the EBP header is application framing, not a replacement for TLS authentication or integrity.

The header layout is fixed for wire version 1. Any incompatible change requires a new version or an explicitly negotiated extension.

## 6. Frame Types

Initial frame types:

- `HELLO`: negotiate protocol and supported capabilities.
- `AUTH`: perform the specified application identity-binding checks.
- `CONTROL`: carry a bounded, versioned CBOR control message.
- `DATA`: carry raw file bytes for an authorized transfer.
- `WINDOW_UPDATE`: update permitted in-flight data.
- `CANCEL`: cancel an identified operation.
- `ERROR`: report a defined failure code.

The protocol document must define the mapping from existing application messages to these frame types.

Control frames must not be mixed into arbitrary binary file payloads. A `DATA` frame must identify an existing authorized transfer through its stream ID and transfer state.

## 7. Limits and Resource Control

Initial implementation limits:

- Control payload: at most 64 KiB.
- Data payload: at most 1 MiB.
- Initial target data chunk: 256 KiB.
- Initial maximum concurrent outgoing transfers: 2.
- Initial
