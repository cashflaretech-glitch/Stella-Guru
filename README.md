# Stellar Guru Contracts

Stealth address smart contracts for the **Stellar Guru** multichain privacy platform. EVM contracts in Solidity (Hardhat), Stellar contracts in Soroban/Rust, Solana programs in Anchor/Rust, and CKB scripts in Rust (RISC-V).

Every payment generates a fresh one-time stealth address so on-chain observers cannot link the sender, recipient, or transaction history.

---

# EVM Contracts (Solidity)

| Contract | Description |
|----------|-------------|
| **ERC5564Announcer** | Minimal singleton contract that emits Announcement events according to ERC-5564. Contains no storage and no access control. |
| **ERC6538Registry** | Stealth meta-address registry implementing ERC-6538. Supports direct registration and delegated registration using EIP-712 signatures with replay-protected nonces. |
| **StellarGuruSender** | Atomically transfers ETH or ERC-20 tokens to a stealth address while publishing a stealth announcement in the same transaction. Supports batch transfers and optional ETH gas tips. |
| **StellarGuruNames** | Privacy-preserving `.guru` name registry mapping human-readable names to stealth meta-addresses. Ownership is verified using secp256k1 spending key signatures. |
| **StellarGuruWithdrawer** | EIP-7702 delegation target enabling gas-sponsored withdrawals from stealth addresses where a sponsor pays gas on behalf of the owner. |

---

# Stellar Contracts (Soroban/Rust)

| Contract | Description |
|----------|-------------|
| **stealth-announcer** | Emits stealth address announcement events without storing state. |
| **stealth-registry** | Stores mappings from addresses to 64-byte stealth meta-addresses using authorization-gated registration. |
| **stealth-sender** | Performs atomic token transfers together with stealth announcements. Supports batch transfers. |
| **stellar-guru-names** | Privacy-preserving name registry using SHA-256 hashed storage keys with reverse lookup support and lowercase alphanumeric validation (3–32 characters). Supports one level of hierarchical subdomains. |

---

# Hierarchical Names (stellar-guru-names)

Names may exist as:

- `alice`
- `payments.alice`

Each label must satisfy:

- 3–32 characters
- lowercase
- alphanumeric only

Only one level of nesting is allowed.

Example:

```
payments.alice
```

Valid.

```
payments.finance.alice
```

Rejected with:

```
NameTooDeep
```

### Delegation

Subdomains can only be registered when:

- the parent name already exists
- the caller currently owns the parent name

Ownership of the parent is verified every time a subdomain is updated or released.

If ownership of `alice` changes, authority over every `*.alice` subdomain automatically transfers to the new owner.

### Resolution

```
resolve("payments.alice")
```

returns the subdomain's stealth meta-address.

If the parent (`alice`) has been released:

```
NameNotFound
```

is returned.

### Migration Safe

Existing flat names continue working exactly as before without modification.

### SDK Follow-up

The off-chain resolver should split:

```
payments.alice.guru
```

into

```
payments.alice
```

before calling the resolver.

---

# Stellar Design Notes

`stellar/EVENT_TOPIC_DESIGN.md`

contains the indexed-topic strategy for the stealth announcer contract.

---

# Solana Programs (Anchor/Rust)

| Program | Description |
|---------|-------------|
| **guru-announcer** | Stateless event emitter using Anchor `emit!()`. |
| **guru-sender** | Atomic SOL and SPL token transfers combined with stealth announcements. |
| **guru-names** | PDA-based `.guru` name registry. Names are stored as PDA seeds and validated to 3–32 lowercase alphanumeric or hyphen characters. |

---

# CKB Scripts (Rust / RISC-V)

| Script | Description |
|---------|-------------|
| **guru-stealth-lock** | Lock script verifying secp256k1 signatures against `blake160(stealth_pubkey)` while embedding ephemeral public keys inside the cell arguments. Delegates verification to on-chain `ckb-auth`. |
| **guru-names-type** | Type script for `.guru` name registration cells supporting create, update, and release operations. Stores spending and viewing public keys with ownership proven by the cell lock script. |

---

# Getting Started

## Network Preflight Checks

Before deployment verify that the target network is correctly configured.

```bash
./scripts/check-network.sh testnet
```

Checks include:

- Network passphrase
- RPC connectivity
- Friendbot availability (testnet/futurenet)
- Deployment identity

`deploy.sh` automatically performs these checks.

---

# Prerequisites

- Node.js 22+
- Rust toolchain
- Cargo
- Anchor CLI
- Solana CLI
- riscv64-elf-gcc

---

# EVM

```bash
cd evm

npm install

npx hardhat compile

npx hardhat test
```

---

# Stellar

```bash
cd stellar

cargo test --workspace
```

---

# Property Tests

Each Soroban crate contains property-based tests inside:

```
tests/properties.rs
```

These validate:

- Event emission
- Registration round trips
- Invalid input rejection
- Batch transfer invariants
- Complete name lifecycle

Run:

```bash
cd stellar

cargo test --workspace --test properties
```

Extended testing:

```bash
WRAITH_PROPTEST_CASES=16384 cargo test --workspace --test properties
```

Every property executes at least **1,024** generated cases by default.

Nightly CI raises this to **16,384** cases.

---

# Generated Bindings

TypeScript bindings are automatically generated into:

```
stellar/bindings/typescript/
```

These bindings provide fully typed Soroban clients.

## Install Dependencies

```bash
pnpm install
```

## Compile Contracts

```bash
cd stellar

cargo build --target wasm32-unknown-unknown --release

cd ..
```

## Generate Bindings

```bash
pnpm bindings:stellar
```

To generate bindings against deployed testnet contracts:

Configure:

```
stellar/contract-ids.json
```

or environment variables such as:

```
STEALTH_REGISTRY_CONTRACT_ID=...
```

Then run:

```bash
pnpm bindings:stellar
```

Regenerate bindings whenever:

- contract functions change
- events change
- custom Rust types change
- new deployments occur

---

# Solana

```bash
cd solana

anchor build

anchor test
```

---

# CKB

```bash
cd ckb

make build
```

---

# Project Structure

```
evm/
  contracts/
  test/
  scripts/deploy.ts

stellar/
  stealth-announcer/
  stealth-registry/
  stealth-sender/
  stellar-guru-names/

solana/
  programs/
    guru-announcer/
    guru-sender/
    guru-names/
  tests/

ckb/
  contracts/
    guru-stealth-lock/
    guru-names-type/
  testnet.toml
```

---

# Deployed Addresses

## Horizen Testnet

| Contract | Address |
|----------|---------|
| ERC5564Announcer | TBD |
| ERC6538Registry | TBD |
| StellarGuruSender | TBD |
| StellarGuruNames | TBD |
| StellarGuruWithdrawer | TBD |

---

## Stellar Testnet

| Contract | Contract ID |
|----------|-------------|
| stealth-announcer | TBD |
| stealth-registry | TBD |
| stealth-sender | TBD |
| stellar-guru-names | TBD |

---

# Pause / Circuit Breaker

| Contract | Pausable | Admin |
|----------|----------|-------|
| stealth-announcer | No | N/A |
| stealth-registry | Yes | Upgrade Admin |
| stealth-sender | Yes | Upgrade Admin |
| stellar-guru-names | Yes | Upgrade Admin |

See:

```
stellar/PAUSE.md
```

for implementation details.

---

# Solana Devnet

| Program | Program ID |
|----------|------------|
| guru-announcer | TBD |
| guru-sender | TBD |
| guru-names | TBD |

---

# CKB Testnet

| Script | Code Hash | Cell Dep |
|---------|-----------|----------|
| guru-stealth-lock | TBD | TBD |
| guru-names-type | TBD | TBD |
| ckb-auth | TBD | TBD |

---

# Features

- Cross-chain privacy
- One-time stealth addresses
- ERC-5564 support
- ERC-6538 support
- EIP-712 delegated registration
- EIP-7702 sponsored withdrawals
- Soroban support
- Solana Anchor support
- CKB RISC-V scripts
- Batch transfers
- Privacy-preserving name service
- Hierarchical subdomains
- Replay protection
- Type-safe SDK bindings
- Property-based testing

---

# License

MIT