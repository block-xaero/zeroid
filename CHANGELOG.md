# Changelog

All notable changes to XaeroID will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0-rc3]

### Added

#### Ring Buffer Memory Pool Integration
- **XaeroIDPoolManager** - High-performance ring buffer allocation for XaeroID instances
- **Ring buffer pooling** using rusted-ring crate for zero-allocation identity management
- **Memory pool sizing** with automatic size estimation (XaeroID fits in L pool: 4096 bytes)
- **Pool error handling** with dedicated PoolError enum for allocation failures
- **Zero-copy access** to pooled XaeroID instances via RingPtr smart pointers
- **Thread-safe allocation** using static EventAllocator reference
- **Device-aware optimization** support for mobile/tablet/desktop memory constraints

#### Memory Management Features
- **RingPtr<XaeroID>** smart pointer for reference-counted XaeroID access
- **Automatic pool selection** based on XaeroID size (~2453 bytes → L pool)
- **Type-safe transmutation** from RingPtr<PooledEvent<SIZE>> to RingPtr<XaeroID>
- **From<PooledEvent<SIZE>>** trait implementation for XaeroID deserialization
- **bytemuck integration** for Pod-safe serialization to/from ring buffer storage
- **Stack overflow protection** with 16MB stack requirement for test environments

#### Performance Optimizations
- **Zero heap allocation** for XaeroID storage in high-frequency scenarios
- **Predictable memory usage** through pre-allocated ring buffer pools
- **Reference counting** enables efficient XaeroID sharing across components
- **Cache-friendly access** patterns via sequential ring buffer layout
- **Reduced GC pressure** by eliminating frequent XaeroID allocations/deallocations

#### P2P Integration Preparedness
- **Bandwidth optimization** support for network protocols (store author_id as RingPtr)
- **Peer identity caching** with ring buffer-backed XaeroID storage
- **Memory-efficient gossip** protocols through shared XaeroID references
- **Scalable peer management** with bounded memory usage via ring buffer pools

### Technical Implementation

#### Ring Buffer Architecture
- **Pool allocation strategy**: Automatic sizing from XS (64B) to XL (16KB) pools
- **XaeroID pool placement**: L pool (4096 bytes) with ~1643 bytes padding efficiency
- **Memory layout preservation**: Pod-safe structures maintain deterministic layout
- **Reference counting**: Atomic operations for thread-safe XaeroID sharing
- **Type safety**: Compile-time guarantees for pool size compatibility

#### Integration Points
- **rusted-ring dependency**: EventAllocator integration for ring buffer management
- **Stack size requirements**: 16MB minimum for ring buffer initialization in tests
- **Device compatibility**: Configurable pool sizes for iOS/Android/Desktop platforms
- **FFI readiness**: Prepared for Dart/Flutter integration via C-compatible interfaces

#### Test Coverage
- **Basic allocation/retrieval**: XaeroID round-trip through ring buffer pools
- **Data integrity verification**: Byte-level equality after pool storage
- **Reference counting validation**: Multiple RingPtr instances sharing same data
- **Concurrent access testing**: Multi-threaded allocation and access patterns
- **Memory layout verification**: Size and alignment assumptions validation
- **Pool error handling**: Allocation failure scenarios and error propagation

### Dependencies

#### New Dependencies
- `rusted-ring` - Custom ring buffer implementation with smart pointer semantics
- `thiserror` ^2.0 - Enhanced error handling for pool allocation failures

#### Updated Test Requirements
- **Stack size**: 16MB minimum for test execution (RUST_MIN_STACK=16777216)
- **Memory constraints**: Device-aware testing for mobile platform compatibility

### Breaking Changes

#### API Additions (Non-breaking)
- All existing XaeroID APIs remain unchanged
- New pooling functionality is additive and optional
- Backward compatibility maintained for all identity operations

### Known Limitations

#### Memory Requirements
- **Initialization stack**: 16MB minimum for ring buffer allocation
- **Pool exhaustion**: No graceful degradation when ring buffer pools are full
- **Device scaling**: Pool sizes need manual tuning for optimal mobile performance

#### Testing Environment
- **Stack overflow risk**: Tests require RUST_MIN_STACK=16777216 environment variable
- **CI/CD requirements**: Build systems must configure larger stack sizes for test runs

### Future Roadmap

#### Performance Enhancements
- **Pool size auto-tuning**: Dynamic pool sizing based on runtime usage patterns
- **Heap fallback**: Graceful degradation to heap allocation when pools exhausted
- **Memory pressure handling**: Advanced pool management for constrained environments

#### Platform Optimization
- **iOS/Android optimization**: Device-specific pool sizing and initialization strategies
- **WASM compatibility**: Ring buffer adaptation for WebAssembly environments
- **Embedded support**: Ultra-low memory footprint variants for IoT devices

### Migration Guide

#### For Existing Users
No code changes required - all existing XaeroID functionality remains identical.

#### For Performance-Critical Applications
```rust
use xaeroid::pool::{XaeroIDPoolManager, PoolError};

// Initialize pool manager (requires 16MB stack)
let pool_manager = XaeroIDPoolManager::new(allocator);

// Allocate XaeroID in ring buffer (zero heap allocation)
let ring_ptr = pool_manager.allocate_xaero_id(xaero_id)?;

// Access XaeroID (zero-copy)
let retrieved_id = &*ring_ptr;

// Share across components (reference counting)
let shared_ptr = ring_ptr.clone();
```

#### For Test Environments
```bash
# Required for running tests
RUST_MIN_STACK=16777216 cargo test

# Or add to .cargo/config.toml
[env]
RUST_MIN_STACK = "16777216"
```

## [0.2.0-rc1] - 2025-06-10

### Added

#### Core Identity System
- **XaeroID** - Pod-safe decentralized identity structure with embedded Falcon-512 keypairs
- **XaeroIdentityManager** - Identity generation, challenge signing, and verification
- **DID:peer** support with Falcon-512 public keys and multibase Base58BTC encoding
- **Post-quantum security** using Falcon-512 signatures (897-byte public key, 1281-byte secret key)

#### Wallet System
- **XaeroWallet** - Container for identity and zero-knowledge proofs
- **WalletProofEntry** - Extended proof storage with metadata and context
- **WalletProofType** enum supporting 9 different proof categories
- **Pod-safe serialization** with bytemuck for zero-copy operations
- **Proof cleanup** functionality to remove expired proofs

#### Zero-Knowledge Proof Circuits
- **Membership Circuit** - Prove group membership without revealing identity
- **Role Circuit** - Prove authority levels using bit decomposition constraints
- **Object Creation Circuit** - Prove object creation rights with role validation
- **Workspace Creation Circuit** - Prove workspace creation authority
- **Identity Circuit** - Challenge-response authentication with ZK proofs

#### Proof System Infrastructure
- **Groth16 SNARKs** on BN254 curve via Arkworks libraries
- **ProofBytes** container for variable-length proof storage (up to 8KB)
- **Deterministic proof parameters** using fixed seeds for reproducible setups
- **Compressed proof serialization** using Arkworks canonical serialization

#### Verifiable Credentials
- **FalconCredentialIssuer** - Issue and verify credentials with Falcon-512 signatures
- **XaeroCredential** - Pod-safe credential container with embedded proofs
- **CredentialClaims** - Fixed-size claims structure for email and birth year
- **Hash-based proof verification** for credential integrity

#### Event Integration System
- **WalletEventSink** trait for optional event emission
- **WalletCrdtOp** enum for wallet state change events
- **IdentityEvent** enum for identity-related events
- **BlackholeEventSink** - No-op implementation for standalone usage
- **Event sink injection** in all proof generation methods via `_with_sink` variants

#### API Design
- **Backward compatibility** - All original methods preserved alongside event-enabled variants
- **Zero-overhead abstractions** - Blackhole sink compiles away when unused
- **Type safety** - Trait-based event contracts with compile-time verification
- **Memory safety** - Fixed-size structures prevent buffer overflows

### Technical Implementation

#### Cryptographic Primitives
- Falcon-512 detached signatures with variable-length encoding (up to 690 bytes)
- Blake3 hashing for proof integrity and DID derivation
- BN254 elliptic curve for efficient SNARK operations
- Groth16 proving system with compressed proof serialization

#### Memory Layout
- **XaeroID**: 2,702 bytes (Pod-safe, fixed layout)
- **XaeroWallet**: ~45KB total with 16 proof slots
- **WalletProofEntry**: ~8.5KB each with extended proof storage
- All structures maintain deterministic sizes across platforms

#### Circuit Constraints
- **Membership**: 2 constraints (commitment + group validation)
- **Role**: Variable constraints based on bit decomposition (8-bit roles)
- **Object Creation**: Role comparison + root derivation constraints
- **Workspace Creation**: Similar to object creation with multiplication instead of addition

#### Proof Generation Performance
- **Setup time**: ~100ms per circuit (one-time cost)
- **Proving time**: ~500ms average per proof
- **Verification time**: ~5ms per proof
- **Proof size**: ~200 bytes compressed

### Dependencies

#### Core Dependencies
- `ark-bn254` ^0.4.0 - BN254 curve implementation
- `ark-groth16` ^0.4.0 - Groth16 SNARK system
- `ark-r1cs-std` ^0.4.0 - R1CS constraint system
- `pqcrypto-falcon` ^0.2.0 - Falcon-512 post-quantum signatures
- `bytemuck` ^1.13.0 - Pod-safe serialization
- `blake3` ^1.4.0 - Cryptographic hashing
- `multibase` ^0.9.0 - DID encoding

#### Development Dependencies
- `rand` ^0.8.0 - Random number generation for tests
- `tempfile` ^3.0.0 - Temporary file handling in tests
- `ark-std` ^0.4.0 - Standard Arkworks utilities

### Testing

#### Test Coverage
- **Identity generation and serialization**: 100%
- **Challenge signing and verification**: 100%
- **Wallet proof storage and retrieval**: 100%
- **ZK proof generation and verification**: 100%
- **Credential issuance and verification**: 100%
- **Event sink integration**: 100%

#### Test Categories
- **Unit tests**: 47 tests covering core functionality
- **Integration tests**: Cross-component interaction testing
- **Circuit tests**: Proof generation and verification for all circuits
- **Serialization tests**: Pod-safety and cross-platform compatibility
- **Event tests**: Event sink behavior and integration patterns

### Known Limitations

#### Circuit Setup
- Uses deterministic seeds instead of trusted setup ceremony
- Proving/verifying keys generated at runtime (not pre-computed)
- No circuit parameter caching between runs

#### Proof Management
- Maximum 16 proofs per wallet (configurable but fixed at compile time)
- No automatic proof garbage collection (manual cleanup required)
- Proof storage is not encrypted (application responsibility)

#### Key Management
- No key derivation from master seed (each identity independent)
- No key rotation mechanisms
- Secret keys stored in memory (secure storage is application concern)

#### Performance
- Proof generation is CPU-intensive (~500ms per proof)
- No batch proof generation support
- Memory usage scales with number of proofs stored

### Security Considerations

#### Post-Quantum Security
- Falcon-512 provides NIST Level 1 post-quantum security
- 128-bit classical security equivalent
- Resistant to both classical and quantum attacks

#### Zero-Knowledge Properties
- Groth16 provides perfect zero-knowledge
- Non-interactive proofs with public verifiability
- Succinctness: constant-size proofs regardless of statement complexity

#### Implementation Security
- No custom cryptography - uses established libraries
- Pod-safe structures prevent memory corruption
- Fixed-size arrays eliminate buffer overflow risks
- Deterministic serialization prevents timing attacks

## [0.2.0-m2] - 2025-06-09

### Added
- Zero-Knowledge Proof Infrastructure:
- Arkworks integration with bn254 curve and Groth16 proving system
- Three core ZK circuits for privacy-preserving identity:
- RoleCircuit: Prove sufficient permissions without revealing exact role
- MembershipCircuit: Prove group membership without revealing identity
- PreimageCircuit: Prove knowledge of secrets (email, password) without disclosure
- ZK proof generation and verification with arkworks-groth16
- Circuit trait abstraction for extensible proof systems
- Enhanced Credential System:
- ZKCredentialIssuer with embedded zero-knowledge proofs
- Privacy-preserving credential verification without revealing claims
- Integration hooks for xaeroflux control plane events
- Development Dependencies:
- arkworks-bn254 = "0.5" for elliptic curve operations
- arkworks-groth16 = "0.5" for SNARK proof systems
- arkworks-std = "0.5" for constraint system utilities

### Changed
- Expanded CredentialIssuer to support ZK-SNARK proof embedding
- Updated circuit architecture to support xaeroflux P2P identity events
- Enhanced error handling for cryptographic operations

### Fixed
- Circuit constraint generation for privacy-preserving proofs
- Memory-safe ZK proof serialization for P2P gossip protocols

## [0.2.0-m1] - 2025-06-07

### Added
- IdentityManager trait with methods:
- new_id() -> XaeroID for Falcon-512 keypair generation
- sign_challenge(&XaeroID, &[u8]) -> Vec<u8> for detached signature creation
- verify_challenge(&XaeroID, &[u8], &[u8]) -> bool for signature verification
- XaeroID Pod-safe struct (897 B public key + 1281 B secret key)
- DID:peer support:
- encode_peer_did(&[u8; 897]) -> String (Base58BTC multibase)
- decode_peer_did(&str) -> Result<[u8; 897], Error> for offline resolution
- Multibase and thiserror dependencies added for DID encoding/decoding and error handling
- CredentialIssuer stub:
- CredentialClaims Pod-safe struct for claims (email, birth_year)
- FalconCredentialIssuer skeleton for signing claims with Falcon keys

### Changed
- Bumped crate version to 0.2.0-m2 and Rust edition to 2024
- Updated Cargo.toml:
- Added multibase = "0.10.1", thiserror = "1.0", and rand = "0.8"
- Disabled default features on large dependencies to minimize footprint
- Cleaned up tests in identity.rs to use rand::random::<[u8;32]>() instead of reserved .gen() method

### Removed
- Temporary use rand::Rng imports and .gen() calls

### Fixed
- Patch for test suite to compile under Rust 2024 and Clippy -D warnings
- GitHub Actions workflow updated to target xaeroID (removed xaeroflux refs)