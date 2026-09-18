# Runtime, CPI, compute, and transactions

Tips 11, 12, 45, 46, 47, 53, 56, 61, 62, 63, 68, 76, 79, 83, 86, 87, 89, 92, 98.

## Parallelism and write locks (12)

Transactions declare read/write sets. Two txs that **write the same account** cannot run in parallel.

Hot lock: a single fee treasury on every trade. Shard treasuries (Tensor pattern): use the last byte of a varying pubkey as index `0..=255` → 256 fee PDAs. Admin sweeps later. Audit any other globally mutated account (volume counters, global TVL).

## Events via CPI (11)

`msg!` / `sol_log` is expensive and truncates. Logging by CPI into a **noop** (or self-CPI) puts data in **instruction data**, which is cheaper and indexable. Anchor `emit_cpi!` is this pattern. Authenticate: outsiders must not be able to spoof the noop/self-CPI. Same idea as compression: data in calldata, not account space.

## Fee payer vs rent payer (45)

- **Tx fee payer:** first signer, must be a **System** account, charged even on revert (so it is writable on failure).
- **Rent/protocol payer:** whatever Signer the program names — not automatically the tx fee payer.

A smart-account/multisig user still needs a System account with SOL for fees.

## Compute units (46, 47, 92)

CU ≈ execution work. Default ~200k per ix; tx cap **1.4M** via ComputeBudget. Base fee 5000 lamports/signature; priority fee = micro-lamports per **requested** CU.

- Simulate, then `setComputeUnitLimit` to simulated + buffer. You **pay for the request**, not the used amount.
- Always include a small `setComputeUnitPrice` in clients.
- Same ix can vary CU (PDA bump search retries).
- `Pubkey.to_string()` + log is expensive; `pubkey.log()` is cheaper.
- Measure: `sol_log_compute_units()` or [compute_fn!](https://github.com/solana-developers/cu_optimizations).

## Heap (53)

Default: 32KB bump allocator, no free. `#[global_allocator]` for custom (e.g. smalloc) or free-last-alloc. `RequestHeapFrame` exists but has historical pitfalls — read current ComputeBudget docs before relying on it.

## Address lookup tables (56)

Tx size 1232 bytes (~IPv6 MTU). Practical unique accounts per ix ~30 without ALTs; v0 + LUT → 64 locks/tx (LUT can store 256 keys).

Create (`createLookupTable`, recent-slot seed) then `extendLookupTable` (~30 keys/batch). Append-only. Closable only after 256 slots so an index cannot be remapped under an in-flight tx.

## Sysvars (61)

Clock, Rent, Instructions, EpochSchedule, SlotHashes, etc. SIMD-0127: access via **syscalls** (`Clock::get()?`) without passing the account.

If passed as accounts, **check the address**. Fake sysvar accounts are a known exploit class. Prefer syscalls.

Useful: `Clock` (slot, epoch, unix timestamp — tip 58 in [ops-testing.md](ops-testing.md)), `Instructions` (flashloans, tip 76).

## Loaders (62)

Upgradeable BPF Loader: program account (metadata) + **program-data** account (bytecode). Upgrade authority can swap code, change auth, close. Older loaders: immutable BPF loader v2; deprecated v1. Default: upgradeable loader + protected/revocable authority.

## Precompiles and crypto syscalls (63, 86, 79)

Precompiles are **native validator code** with program IDs; invoke like any program (often as **top-level ix**, not CPI — verify current restrictions per program).

| Program | Use |
|---------|-----|
| `Ed25519SigVerify111111111111111111111111111` | Native signatures |
| `KeccakSecp256k11111111111111111111111111111` | ETH/BTC ECDSA |
| `Secp256r1SigVerify1111111111111111111111111` | Many hardware keys |

Syscalls: `sha256`, `keccak256`, `blake3`, `poseidon`, `secp256k1_recover`, ed25519 helpers, `alt_bn128` (ZK), `big_mod_exp`.

Ed25519: pubkey = `kP` on curve; **PDAs are off-curve** so no private key exists. Primer: https://curves.xargs.org

## Direct lamports then CPI (68)

Mutating `account.lamports` then `invoke` fails: `sum of account balances before and after instruction do not match`.

Runtime snapshots balances at ix entry. On CPI it refreshes **only accounts in the CPI**; others keep the snapshot. If A and B were both changed but CPI only includes A, totals disagree.

**Fix:** all direct lamport moves **after** CPIs, **or** include every lamport-changed account in every subsequent CPI (even if unused).

## Write/reload across CPI (83)

Anchor keeps a **working copy**. CPI callees see **on-chain** bytes.

- If callee must observe your writes: serialize/`exit` before `invoke`.
- If you must observe callee writes: `reload()` after CPI.
- You cannot write their accounts; they cannot write yours (except lamport increase).

## Flashloans (76)

Typical: top-level `borrow` then user ixs then `repay` (not nested CPI — stack depth 4 and some programs reject CPI).

`borrow`:

1. Instructions sysvar; assert **not** itself a CPI
2. Scan following ixs; next interaction with **this** program must be `repay` for the **same** loan account (discriminator + accounts)
3. Disallow borrow-borrow-repay, borrow-config-repay, CPI-wrapped borrow/repay

`repay` also forbids CPI. Goal: exactly borrow → use → repay.

## Transaction path (87, 89)

Wallet signs → RPC (can sandwich / delay) → leader TPU. SWQoS: stake-weighted send path. Design programs assuming hostile RPC.

Serialization: shortvecs; signatures (64B); message = header (3 bytes of offsets) + sorted keys + recent blockhash + instructions (program index, account indexes, data).

Header splits the key list into writable signers, readonly signers, writable nonsigners, readonly nonsigners. **Signer/writable flags are transaction-global.** Reusing a pubkey already in the tx costs ~1 byte. v0 adds LUT indexes.

## CPI limits (98)

| Limit | Value |
|-------|--------|
| Call depth | 4 (`A→B→C→D`) |
| Trace / max CPIs | 64 (≤63 CPIs; includes top-level ixs) |
| Account grow in CPI | 10240 bytes |
| PDA seeds | 16 × 32 bytes |
| Accounts in CPI | Must appear in the top-level tx (historically callee program too) |
| Account locks / tx | 64 |
| Shared CU | 1.4M max / 200k default |
| Reentrancy | `A→B→A` forbidden; `A→A` allowed |

Signer seeds on `invoke_signed` create virtual PDA signers. Direct lamport edits: see tip 68.
