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
- Initial maximum concurrent incoming transfers: 2.
- Initial in-flight data budget: configurable, bounded per connection.
- All queues must have explicit capacity limits.

These are conservative initial limits, not proven optimal values. Benchmarks may justify changing them.

When the receiver cannot accept more data, it must stop granting additional capacity. The sender must respect the advertised window rather than accumulate unbounded buffers.

Resource exhaustion must produce a controlled error, not an uncontrolled memory increase or application crash.

## 8. Small-File Efficiency

To reduce latency:

- Reuse an established authenticated connection.
- Avoid repeating TLS handshakes for every file.
- Batch related control metadata when doing so preserves clear error handling.
- Avoid base64 encoding binary data.
- Avoid unnecessary per-chunk allocations and copies.
- Do not perform synchronous disk or network operations on UI threads.
- Avoid mandatory round trips for every chunk where bounded flow control can safely permit pipelining.

Small-file transfers must still receive authorization and integrity verification. Performance optimizations must not bypass security checks.

## 9. Large-File Transfer

- Stream data from source to network and network to destination.
- Never load an entire file into memory.
- Use bounded reusable buffers.
- Apply backpressure when the receiver, disk, or network cannot keep up.
- Compute a standard cryptographic content hash incrementally.
- Track expected file size and bytes accepted.
- Detect premature EOF and excess data.
- Write to a temporary destination.
- Finalize the destination only after content verification and successful completion.
- Clean up partial files safely after cancellation or unrecoverable failure.

The transfer engine must distinguish bytes sent, bytes received, bytes written, and verified completion. These are not interchangeable progress metrics.

## 10. Integrity and Completion

A transfer descriptor includes:

- Unique transfer ID.
- Authorized source and destination references.
- Expected size.
- Agreed integrity algorithm and expected digest.
- Transfer state and permitted recovery metadata.

The receiver must independently verify the received content. An acknowledgment of received bytes is not proof that the complete file is valid.

`TRANSFER_FINISH` must not imply success by itself. Success is reported only after the receiver validates the final size and digest and successfully finalizes the destination.

A connection failure during finalization must not produce an ambiguous success result. The transfer status must be recoverable through a defined status query.

## 11. Cancellation, Reconnect, and Resume

Cancellation is a state transition, not merely closing a socket.

- Stop accepting new chunks for a cancelled transfer.
- Release associated buffers and resources.
- Clean up incomplete destination files unless a valid resume record is retained.
- Ignore or reject late data according to the defined transfer state.
- Return a terminal outcome that accurately reports whether cancellation completed.

Reconnect policy:

- Use bounded exponential backoff with jitter.
- Never retry authentication failures indefinitely.
- Require fresh session authentication after reconnection.
- Do not reuse an old session as proof of a new authenticated connection.

Resume policy:

- Revalidate device identity and transfer authorization.
- Verify the identity of the source content.
- Resume only from a safely verified checkpoint.
- Reconcile receiver-confirmed durable progress.
- Never trust an offset supplied by an unauthenticated peer.
- Do not append blindly to an existing destination.

Resume must be implemented only after normal transfer integrity and state recovery are reliable.

## 12. Error Model

Errors must use stable machine-readable codes, with safe diagnostic descriptions.

Required categories include:

- `UNSUPPORTED_VERSION`
- `INVALID_FRAME`
- `MESSAGE_TOO_LARGE`
- `AUTHENTICATION_FAILED`
- `NOT_AUTHORIZED`
- `TRANSFER_NOT_FOUND`
- `TRANSFER_CANCELLED`
- `INSUFFICIENT_STORAGE`
- `INTEGRITY_FAILED`
- `CONNECTION_LOST`
- `TIMEOUT`
- `RESOURCE_LIMIT`
- `INTERNAL_ERROR`

Never expose secrets, private keys, file contents, or unnecessary filesystem paths in remote error messages or logs.

## 13. Testing and Benchmarks

Required automated tests:

- Header encoding and decoding across both implementations.
- Partial TCP reads and writes.
- Invalid, truncated, oversized, and malformed frames.
- Unsupported versions and invalid state transitions.
- Authentication failure and trust revocation.
- Unauthorized transfer requests.
- Cancellation and disconnect at different transfer stages.
- Integrity mismatch and storage failures.
- Bounded-memory behavior under slow receivers.
- Reconnect and resume consistency.

Required real-device benchmarks:

- Small-file latency at multiple sizes.
- Large-file sustained throughput.
- Peak memory and CPU usage.
- Hashing overhead.
- Behavior under Wi-Fi interruptions and slow storage.
- Multiple simultaneous transfers.

Record the device, OS version, network conditions, file sizes, test repetitions, and measured results. Do not publish performance claims before testing.

## 14. Implementation Gates

1. Agree on and review the wire specification.
2. Implement framing and parser tests.
3. Implement authenticated TLS sessions and peer identity binding.
4. Pass cross-platform interoperability tests.
5. Implement bounded streaming and backpressure.
6. Pass integrity, cancellation, and failure tests.
7. Test on real Windows and Android devices.
8. Add resume only after the preceding stages are reliable.

This document defines intended behavior. It does not certify the implementation as secure, complete, or performant.
