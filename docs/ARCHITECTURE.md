# EcoBridge Architecture

## Vision

EcoBridge is an open, cross-platform device ecosystem connecting Android and Windows devices without requiring a proprietary ecosystem, cloud account, root access, or developer tools for normal users.

## MVP

1. Discover nearby trusted-device candidates using Bluetooth Low Energy (BLE).
2. Pair devices with explicit user consent and establish persistent trust.
3. Connect over the local network for data transfer.
4. Browse only user-authorized shared folders.
5. Transfer files in both directions with progress, cancellation, integrity verification, and safe error handling.

## Architecture

- **Android:** Kotlin, Android SDK, BLE APIs, Storage Access Framework, Android Keystore.
- **Windows:** Rust backend with Tauri as the desktop application framework.
- **Discovery:** BLE is used for finding devices and coordinating connections, not bulk file transfers.
- **Transport:** An authenticated, encrypted local-network connection.
- **Protocol:** Versioned, documented messages shared by both implementations.
- **Builds:** GitHub Actions for automated builds and tests.
- **Storage:** Stream file contents; do not load entire files into memory.

## Security

- Require explicit consent before pairing.
- Authenticate trusted devices and encrypt data in transit.
- Never treat proximity alone as proof of identity.
- Limit file access to locations authorized by the user.
- Validate incoming messages and file metadata.
- Support trust revocation and safe transfer cancellation.
- Never implement custom cryptographic algorithms.

## Reliability

- Use bounded queues and limited concurrent transfers.
- Verify file integrity before marking a transfer complete.
- Use temporary destination files and finalize them only after successful verification.
- Handle disconnects, permission revocation, and malformed messages explicitly.
- Keep diagnostics useful without logging file contents or secrets.

## Development Principles

- Build and test in GitHub Actions wherever practical.
- Keep local development lightweight.
- Test on real Windows and Android devices.
- Prioritize correctness, security, and reliability over feature count.
- Do not claim support until the feature has been tested.

## Initial Milestone

Prove that the Android and Windows prototypes can discover one another over BLE. Then implement pairing, authenticated connections, and file transfers incrementally.
