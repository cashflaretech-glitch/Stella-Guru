---

Cross-Chain Announcement Schema Conformance Audit

This document audits announcement schema compatibility across Stellar Guru supported chains and defines the canonical SDK normalization model used by off-chain indexers and clients.

The platform supports:

EVM (Solidity / Hardhat)

Stellar (Soroban / Rust)

Solana (Anchor / Rust)

CKB (RISC-V / Rust)


Although every implementation publishes the same logical stealth announcement, each blockchain exposes different native primitives (logs, events, script arguments, witnesses, or account models). The SDK normalizes these differences into a single representation.


---

Canonical SDK Announcement Model

interface Announcement {
  chain: "evm" | "stellar" | "solana" | "ckb";

  schemeId: number | null;

  stealthAddress: Uint8Array;

  caller: Uint8Array | null;

  ephemeralPubKey: Uint8Array;

  metadata: Uint8Array;

  txHash: Uint8Array;

  logIndex: number | null;
}

Field semantics:

Field	Description

chain	Source blockchain
schemeId	Stealth scheme identifier (null on CKB)
stealthAddress	Chain-native recipient identity normalized into raw bytes
caller	Transaction initiator or emitting contract identity where available
ephemeralPubKey	Sender-generated ephemeral public key
metadata	Opaque metadata payload
txHash	Transaction hash
logIndex	Event/log ordering within a transaction when applicable



---

Chain Compatibility Matrix

Field	EVM	Stellar	Solana	CKB	Diverges?

schemeId	uint256 event field	u32 event topic	u32 event field	Not present	Yes
stealthAddress	address (20 bytes)	Soroban Address	Pubkey (32 bytes)	blake160(stealth_pubkey) stored in lock args	Yes
caller	msg.sender	Contract address / authorization context	Signer Pubkey	Not available	Yes
ephemeralPubKey	bytes (expected compressed secp256k1)	BytesN<32>	[u8;32]	33-byte compressed secp256k1 key in lock args	Yes
metadata	bytes	Bytes	Vec<u8>	Not independently stored	Yes



---

Announcement Carrier

Each chain transports announcements differently.

Chain	Announcement Mechanism

EVM	ERC5564Announcer event
Stellar	stealth-announcer Soroban event
Solana	guru-announcer Anchor emit!() event
CKB	guru-stealth-lock lock arguments and witness model


This divergence is intentional because each execution environment exposes different event capabilities.


---

Source of Truth

EVM

evm/contracts/ERC5564Announcer.sol
evm/contracts/interfaces/IERC5564Announcer.sol

Implements the ERC-5564 announcement interface.


---

Stellar

stellar/stealth-announcer/

Stateless Soroban contract emitting stealth announcement events.

The indexed topic strategy is documented in:

stellar/EVENT_TOPIC_DESIGN.md


---

Solana

solana/programs/guru-announcer/

Anchor program emitting stateless announcement events via emit!().


---

CKB

ckb/contracts/guru-stealth-lock/

Lock script embedding announcement information inside cell arguments while verifying ownership through ckb-auth.


---

Detailed Field Mapping

schemeId

EVM

uint256

Stellar

u32

(topic)

Solana

u32

CKB

Not encoded.

SDK

number | null

Finding

Medium

The width differs (uint256 vs u32), while CKB intentionally omits the field.

SDK implementations should normalize to a JavaScript-safe integer and represent CKB with null.


---

stealthAddress

EVM

20-byte account address.

Stellar

Soroban Address.

Solana

32-byte Pubkey.

CKB

blake160(stealth_pubkey)

stored inside the lock arguments.

SDK

Uint8Array

Finding

Intentional (Medium)

Each blockchain uses a different account model. SDKs should expose canonical raw bytes together with the chain identifier.


---

caller

EVM

msg.sender

Stellar

Contract execution context / authorization identity.

Solana

Transaction signer Pubkey.

CKB

Unavailable.

SDK

Uint8Array | null

Finding

Low

Caller semantics vary across execution environments and should be treated as auxiliary metadata.


---

ephemeralPubKey

EVM

Dynamic bytes (expected compressed secp256k1).

Stellar

BytesN<32>

Solana

[u8;32]

CKB

33-byte compressed secp256k1 public key stored in lock arguments.

SDK

Uint8Array

Finding

High

EVM and CKB naturally use compressed secp256k1 (33 bytes), while Stellar and Solana currently emit fixed 32-byte representations.

SDK implementations must normalize these formats during decoding.


---

metadata

EVM

bytes

Stellar

Bytes

Solana

Vec<u8>

CKB

No standalone metadata field.

SDK

Uint8Array

Finding

Low

Dynamic byte containers differ syntactically but normalize cleanly into a byte array.


---

SDK Normalization

Regardless of chain, SDK decoders should emit the canonical Announcement object.

Normalization responsibilities include:

converting all address types into raw byte arrays

normalizing public-key encodings

converting native byte containers into Uint8Array

mapping unavailable fields (schemeId, caller, logIndex) to null

preserving transaction ordering information where available


This abstraction allows downstream wallets, scanners, and indexers to process announcements without chain-specific logic.


---

Proposed Alignment Improvements

The current implementations are compatible, but several optional improvements would increase long-term consistency.

1. Standardize schemeId

Document an implementation guideline limiting values to a portable range (for example, u64) while maintaining ERC-5564 compatibility.

Breaking change: No.


---

2. Validate EVM Ephemeral Keys

Require the emitted ephemeralPubKey to be exactly one compressed secp256k1 public key (33 bytes).

Benefits:

eliminates malformed announcements

aligns with CKB

simplifies SDK validation


Breaking change: No.


---

3. Future CKB Versioning

A future version of guru-stealth-lock could optionally include a compact schemeId prefix inside lock arguments to improve symmetry with event-based chains.

This should only be introduced as a new script version.


---

Intentional Architectural Differences

These differences are fundamental and should remain unchanged.

1. CKB uses lock scripts rather than event logs, embedding announcement data inside cell arguments and witnesses.


2. Recipient identity is chain-native (address, Soroban Address, Solana Pubkey, or blake160).


3. Caller semantics differ because execution environments expose different authorization models.




---

Conformance Testing

Repository tests validate announcement correctness across all supported chains.

Chain	Coverage

EVM	evm/test/ERC5564Announcer.test.ts validates ERC-5564 event emission and ABI decoding.
Stellar	Property tests verify event emission, authorization, and announcement invariants for stealth-announcer.
Solana	solana/tests/ validates guru-announcer event payloads emitted through Anchor.
CKB	Unit tests validate parsing of guru-stealth-lock arguments, including extraction of the embedded ephemeral public key and blake160 stealth identity.



---

Overall Assessment

Category	Status

Logical announcement compatibility	✅ Compatible
SDK normalization feasibility	✅ Fully achievable
Address encoding	⚠ Chain-native by design
Event transport	⚠ Chain-specific by design
Ephemeral key encoding	⚠ Requires SDK normalization
Metadata compatibility	✅ Compatible
Cross-chain interoperability	✅ Fully supported through SDK normalization
