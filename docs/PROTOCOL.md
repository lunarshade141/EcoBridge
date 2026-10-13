# EcoBridge Protocol Specification

## 1. Purpose

Define the communication contract shared by Android and Windows implementations of EcoBridge. Both devices are peers. Either peer may initiate a request or transfer.

This document specifies intended behavior, not a claim that the protocol is implemented or secure yet.

## 2. Design Requirements

- Versioned, strictly validated messages.
- Explicit device identity and user-approved pairing.
- Authenticated and encrypted sessions before sensitive operations.
- Bounded message sizes, queues, and concurrent transfers.
- Streaming file transfers with integrity verification.
- Clear errors and safe cancellation.
- No custom cryptographic algorithms.
- BLE discovery is separate from bulk data transport.
- Never trust a device solely because it is nearby or previously discovered.

## 3. Transport Layers

### Discovery

Bluetooth Low Energy is used to discover compatible EcoBridge devices and coordinate connection establishment.

Discovery metadata must be minimal. Do not advertise secrets, private file information, or reusable authentication tokens.

### Data Transport

Bulk data uses an authenticated, encrypted connection over a supported local-network transport.

The transport must not be considered trusted until peer authentication succeeds. The implementation must verify the peer identity against the pairing state.

The exact framing, authentication handshake, and transport details must be documented and reviewed before implementation is declared complete.

## 4. Message Envelope

Every protocol message must contain:

- `version`: protocol version.
- `type`: message type.
- `request_id`: identifier used to correlate requests and responses.
- `payload`: fields defined for that message type.

The actual wire format and maximum encoded message size must be specified before implementation. Parsers must reject unsupported versions, malformed encodings, invalid fields, oversized messages, and unexpected state transitions.

Do not deserialize untrusted input into unrestricted filesystem paths or executable objects.

## 5. Message Families

### Session and capabilities

- `HELLO`: exchange protocol versions and supported capabilities.
- `AUTH`: authenticate the remote peer using the established pairing mechanism.
- `CAPABILITIES`: report supported operations after authentication.

### Pairing and trust

- `PAIR_INIT`: initiate a user-approved pairing flow.
- `PAIR_CONFIRM`: complete pairing only after the required verification and user consent.

Pairing must use a reviewed cryptographic design. The precise key agreement, identity verification, replay protection, key storage, and recovery rules must be documented before implementation.

### File browsing

- `LIST_DIR`: list entries within a user-authorized shared location.
- `FILE_INFO`: return validated metadata for a selected entry.

Use opaque location and entry identifiers where practical. Never expose arbitrary filesystem access through a remote path supplied by another device.

### Transfers

- `TRANSFER_START`: request a transfer with validated metadata.
- `TRANSFER_CHUNK`: carry a bounded piece of file content.
- `TRANSFER_ACK`: acknowledge accepted progress.
- `TRANSFER_FINISH`: finalize only after the receiver verifies the expected content.
- `CANCEL`: request cancellation and safe cleanup.

Transfer requests must define identifiers, direction, size, integrity information, and authorization requirements.

### Errors

- `ERROR`: return a structured error code and safe diagnostic message.

Errors must not reveal secrets, private file contents, or unnecessary local filesystem details.

## 6. Transfer Requirements

- Stream content; never load an entire file into memory.
- Bound chunk size and concurrent transfers.
- Calculate and verify a cryptographic content hash using a standard library.
- Write incoming content to a temporary destination.
- Publish the completed file only after successful verification.
- Handle insufficient storage, permission denial, disconnects, cancellation, and corrupted content.
- Never report success before the destination has been finalized.
- Resume interrupted transfers only when identity, authorization, and verified offsets can be safely established.

## 7. Authorization and Privacy

- Require explicit trust establishment before sensitive operations.
- Recheck current permissions when required by the operating system.
- Respect Android Storage Access Framework grants and Windows filesystem permissions.
- Limit browsing and transfer to authorized locations.
- Allow users to revoke trusted devices.
- Do not log file contents, credentials, private keys, or authentication tokens.

## 8. State Machine

A connection follows these conceptual states:

1. `DISCOVERED`
2. `PAIRING` when pairing is needed
3. `TRUSTED` after verified user-approved pairing
4. `CONNECTING`
5. `AUTHENTICATING`
6. `CONNECTED` after authentication succeeds
7. `TRANSFERRING` during an authorized transfer
8. `DISCONNECTED` or `ERROR` when the session cannot continue

A device's trust status is independent of its proximity and connection status. Invalid transitions must be rejected.

## 9. Compatibility and Testing

- Both implementations must use the same protocol version and message definitions.
- Maintain shared test vectors for serialization, validation, and error handling.
- Test malformed messages, replay attempts, unauthorized requests, interrupted transfers, and integrity failures.
- Test on real Windows and Android hardware.
- A successful build alone does not establish protocol correctness or security.

## 10. Implementation Status

This specification describes required behavior. A message type is not considered implemented until it has working code, relevant automated tests, and device-level verification where applicable.
