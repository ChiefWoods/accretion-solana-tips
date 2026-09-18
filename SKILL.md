---
name: accretion-solana-tips
description: Applies r0bre's 100 daily Solana tips from Accretion when writing, reviewing, or debugging Solana programs. Use for Anchor, Pinocchio, or native programs; account/PDA validation; remaining accounts; CPI and self-reentrancy; Token-2022; rounding and overflow; CU optimization; vault design; flashloans; TOCTOU; and Solana security reviews.
---

# Accretion Solana Tips

Compile and apply [r0bre's 100 Daily Solana Tips](https://accretion.xyz/blog/100-solana-tips) (Accretion Labs, Jun 2025; [original X thread](https://x.com/r0bre/status/1878796569597882757)).

**Do not treat this skill as source of truth.** Verify against current runtime/SDK code (tip 100). Tooling names and feature gates drift.

## How to apply

1. Identify the task: **write**, **review**, or **debug**.
2. Apply every [Hard rule](#hard-rules) that matches the change. Do not skip "because Anchor handles it" unless the framework path is confirmed.
3. Open the matching reference and follow those tips:
   - Structure, naming, state machines, Pinocchio/C → [program-design.md](references/program-design.md)
   - Accounts, PDAs, ATA init, close, layout, zero-copy → [accounts.md](references/accounts.md)
   - Invariants, CPI, remaining accounts, TOCTOU, rugs → [security.md](references/security.md)
   - SPL/Token-2022, rounding, vaults, donations → [tokens-math.md](references/tokens-math.md)
   - CPI limits, CU, sysvars, txs, flashloans → [runtime.md](references/runtime.md)
   - Testing, monitoring, incident response, tooling → [ops-testing.md](references/ops-testing.md)
4. For a review, run the checklist in [checklist.md](references/checklist.md).
5. Cite tip numbers in explanations (`tip 15`, `tip 68`) so the user can trace the rule.

## Hard rules

These fail audits and production programs. Default to them unless the user explicitly overrides.

| # | Rule |
|---|------|
| 13 | Financial math: no raw `+` `-` `*`. Use `checked_*` / `saturating_*` unless overflow is proven in a comment. |
| 20 | Round to the **protocol's** advantage. User outflows round down; user inflows / protocol outflows do not. |
| 21 | Prefer `From`/`TryFrom` over `as`. Truncating casts are bugs. |
| 24 | No floats on-chain. Scale integers (cents, bps, 1e9). |
| 15 | Seeded ATAs: `init_if_needed`, never `init`. Anyone can create an ATA and DoS `init`. |
| 26 | Do not `create_account` for known PDAs. Lamport griefing. Allocate + transfer rent + assign. |
| 40 | Close = defund + assign System Program + realloc 0. Zeroing lamports alone is account revival. |
| 10 | Remaining accounts: expected count, owner, discriminator, seeds/data. Treat as unchecked. |
| 27 | Verify CPI callee program ID. `invoke_signed` to an arbitrary program drains signers. |
| 14, 88 | Block self-CPI (`program_id`) unless the instruction is designed for it. Least privilege on accounts passed to arbitrary invoke. |
| 83 | Serialize own account writes **before** CPI if the callee must see them. Reload foreign accounts **after** CPI. |
| 68 | Direct lamport mutation then CPI fails (`sum of account balances…`). Mutate after CPIs, or pass every mutated account into the CPI. |
| 16 | Separate `payer` Signer from `authority` Signer. |
| 9 | PDA seeds: `"prefix:"` + pubkey(s) + optional id. No prefix that is a substring of another. Variable-length last. |
| 7–8 | Fixed-size fields first, variable-size last. Keep reserved padding (≥1 byte via future `Option`, or ≥64 bytes for fixed size). |
| 4–5 | End mutating instructions with state invariants and delta assertions (deposits never decrease balances). |
| 6 | Global operating-mode account (normal / halt / withdraw-only), admin ≠ upgrade authority, mostly read-only. |
| 11, 41 | Do not parse `msg!` logs off-chain. Use `emit_cpi!` / authenticated noop CPI. Logs inject, truncate at 10KB, and survive failed txs. |
| 33 | Store token **authorities**, not user token accounts. Users can close or reassign token accounts. |
| 35 | Token-2022: explicit allow/deny per extension. Permanent delegate, transfer hooks, transfer fees, closeable mints are hostile by default. |
| 30 | Empty `Vec` skips the loop. Require `!is_empty()` when post-loop logic assumes validated items. |
| 59 | Take/accept instructions must pin expected params (price, mint, amount). Account address alone is TOCTOU. |
| 70 | Anyone can donate tokens/lamports. Use internal balances; do not trust `token.amount` / `lamports` for share math. |
| 71 | No on-chain randomness from slot, blockhash, timestamp, or user entropy. |
| 61 | Sysvars via `Clock::get()?` (syscalls). Never trust an unchecked sysvar account. |
| 94 | Owner check ≠ type check. Transient assign-to-program then GC can fake ownership inside one tx. |
| 75 | Init and any instruction that depends on external state is frontrunnable. Design for it. |
| 100 | Read and run the code. Do not trust threads, this skill, or stale docs as absolute truth. |

## Topic router

| If you are… | Read |
|---|---|
| Splitting `lib.rs`, naming ix/accounts, state enums, nested Accounts, traits, macros | [program-design.md](references/program-design.md) |
| PDA seeds, `has_one`, ATA, close, remaining accounts, stack/heap, LazyAccount, duplicate accounts | [accounts.md](references/accounts.md) |
| Audit, invariant, reentrancy, arbitrary CPI, remaining accounts, TOCTOU, pool squat, backdoors | [security.md](references/security.md) |
| Fees, overflow, rounding, wSOL, Token-2022, vault vs per-pool ATA | [tokens-math.md](references/tokens-math.md) |
| CU, priority fees, ALT, sysvars, loaders, precompiles, flashloan introspection, lamport+CPI | [runtime.md](references/runtime.md) |
| Tests, fuzz, monitoring, cranks, time, metadata, incident plan, closed-source RE | [ops-testing.md](references/ops-testing.md) |
| Shipping a review | [checklist.md](references/checklist.md) |

## Tip index

| Tip | One-liner | File |
|-----|-----------|------|
| 1 | Split program: `lib.rs` + `instructions/` + `state/`. `anchor init --template multiple` | program-design |
| 2 | Prefer `has_one`; complex checks → validation fn + custom errors | accounts |
| 3 | Study Squads v4, Sanctum S, Ellipsis Plasma/Gavel | program-design |
| 4 | Invariants on state, called at end of mutating ix | security |
| 5 | Dynamic assertions: snapshot before, assert after | security |
| 6 | Global halt / withdraw-only mode; admin ≠ upgrade authority | security |
| 7 | Reserved padding for future fields | accounts |
| 8 | Fixed-size fields first; variable-size last | accounts |
| 9 | PDA seed pattern: `"prefix:"` + pubkey(s) + id | accounts |
| 10 | Remaining accounts are unchecked — owner, disc, seeds | security |
| 11 | Cheap logs via noop / `emit_cpi!`; authenticate | runtime |
| 12 | Shard write-hot accounts (fee treasuries) for parallelism | runtime |
| 13 | Checked math for financial ops | tokens-math |
| 14 | Self-reentrancy `A→A` is allowed; forbid unexpected self-CPI | security |
| 15 | ATA: `init_if_needed` (or ATA `CreateIdempotent`) | accounts |
| 16 | Separate payer vs authority | accounts |
| 17 | Custom error per failure + a test per error | ops-testing |
| 18 | Multisig authorities: `floor(m/2)+1` of `m` (min 2/3). Not 1/m or m/m | ops-testing |
| 19 | Incident runbook before you need it | ops-testing |
| 20 | Integer division rounds down; round to protocol | tokens-math |
| 21 | `as` truncates; use `From`/`TryFrom` | tokens-math |
| 22 | Monitor accounts, TVL, events, ix volume | ops-testing |
| 23 | Whitelist: NFT (`amount==1`), PDA, or merkle — not a giant `Vec` | security |
| 24 | No floats on-chain | tokens-math |
| 25 | Multi-program architecture for privilege separation | program-design |
| 26 | `create_account` DoS via lamports; allocate/assign/rent instead | accounts |
| 27 | Verify CPI program ID; signers propagate | security |
| 28 | Counterparty risk: least privilege, upgrade-auth hygiene | security |
| 29 | Occasional pedantic Clippy pass | ops-testing |
| 30 | Empty-vec loop skips validation | security |
| 31 | IDLs: clients, `anchor account`, explorers | ops-testing |
| 32 | ATA model; Token-2022 ATAs have immutable owner | accounts |
| 33 | Token freeze, delegate, closeable TAs, transferable ownership | tokens-math |
| 34 | CLI: `account`, `confirm -v`, `logs`, `block` | ops-testing |
| 35 | Token-2022 extensions need an explicit support policy | tokens-math |
| 36 | Accept SOL as wSOL (`So111…112` + `SyncNative`) | tokens-math |
| 37 | Anchor `fallback` fn for unknown discriminators | program-design |
| 38 | 8-byte discriminators on ix and accounts; don't confuse by size alone | accounts |
| 39 | `///` docstrings land in the IDL | program-design |
| 40 | Account revival if you only zero lamports | accounts |
| 41 | Log injection / truncation; don't parse logs | security |
| 42 | Only owner writes (except lamport increase); check owner before deserialize | accounts |
| 43 | `has_one` + seed on immutable field is often redundant CU | accounts |
| 44 | Pinocchio for small/fast programs; you own the checks | program-design |
| 45 | Tx fee payer = first signer, System account; rent payer is separate | runtime |
| 46 | Measure CU with `sol_log_compute_units` / `compute_fn!` | runtime |
| 47 | Set CU limit from simulation + buffer; add priority fee | runtime |
| 48 | Slot/epoch boundaries are bundle-attack surfaces | security |
| 49 | Zero-copy: `AccountLoader`; discriminator checked only on `load` | accounts |
| 50 | 10MB max account; CPI grow ≤10240; PDAs need stepwise realloc | accounts |
| 51 | 4KB stack / 32KB heap; `Box`, split fns, zero-copy | accounts |
| 52 | Track SIMDs and `solana feature status` | ops-testing |
| 53 | Default bump allocator; custom `#[global_allocator]` / heap frame | runtime |
| 54 | Event sequence numbers for indexers | ops-testing |
| 55 | Permissionless cranks are an attack surface; consider inline crank | ops-testing |
| 56 | ALTs / v0 txs for >~30 accounts; append-only, slot-seeded | runtime |
| 57 | `LazyAccount` for reading fields from large accounts | accounts |
| 58 | `Clock` unix timestamp is slot-granular; consecutive slots may share it | ops-testing |
| 59 | Offer/accept TOCTOU — pin expected params | security |
| 60 | Launchpad pool-squatting / graduation frontrun | security |
| 61 | Sysvars via syscall; spoofable if passed as accounts unchecked | runtime |
| 62 | BPF loaders; program vs program-data account | runtime |
| 63 | Precompiles (ed25519, secp256k1) are native, invoked like programs | runtime |
| 64 | Token metadata URI can be malleable even if update auth is revoked | ops-testing |
| 65 | Traits on Accounts structs for shared auth logic | program-design |
| 66 | Nested `#[derive(Accounts)]` to DRY constraints | program-design |
| 67 | `Option<Account>` None = this program_id; cannot register self | accounts |
| 68 | Direct lamport edit then CPI breaks balance invariant | runtime |
| 69 | Block scopes to release borrows | program-design |
| 70 | Donation attacks; keep internal balances | tokens-math |
| 71 | On-chain RNG is attacker-controlled | security |
| 72 | Per-pool vaults over one global vault | tokens-math |
| 73 | `AccountInfo` vs deserialized account vs `AccountMeta` | accounts |
| 74 | `unsafe` only when required; auditors inspect every block | program-design |
| 75 | Frontrun init and any external-state-dependent ix | security |
| 76 | Flashloans: inspect Instructions sysvar; borrow then repay; no CPI-borrow | runtime |
| 77 | Rent is a redeemable deposit; non-exempt create fails | accounts |
| 78 | Anchor lifetimes `'a,'b,'c,'info` | program-design |
| 79 | Ed25519; PDAs are off-curve (no private key) | runtime |
| 80 | Names: `subject_verb_object` instructions; never vague `owner` | program-design |
| 81 | Upgrade auth, fee switches, hidden accounts are backdoors | security |
| 82 | Macros to DRY; `cargo expand` for auditors | program-design |
| 83 | Write-before-CPI / reload-after-CPI | runtime |
| 84 | Explicit state enums + transition methods + `match` | program-design |
| 85 | Entrypoint; depend on other programs with `no-entrypoint` | program-design |
| 86 | Sigverify precompiles + hash / alt_bn128 / bigmodexp syscalls | runtime |
| 87 | RPC is a mempool; SWQoS; design for sandwich/frontrun | runtime |
| 88 | Arbitrary invoke: readonly, omit sensitive accounts, block self | security |
| 89 | Tx serialization: header offsets, shared writable/signer flags | runtime |
| 90 | C / sBPF assembly for size and CU | program-design |
| 91 | Duplicate accounts: last write wins; double `init` fails | accounts |
| 92 | Shared CU budget; request what you use; PDA bump variance | runtime |
| 93 | Newtypes (`struct Lamports(u64)`) not `type` aliases | program-design |
| 94 | Transient owner attack bypasses owner-only checks | security |
| 95 | Unit / ProgramTest / Anchor / LiteSVM / Mollusk — write tests | ops-testing |
| 96 | Fuzz with Trident / coverage-guided sequences | ops-testing |
| 97 | Unified Solana CLI toolchain (historically "mucho") | ops-testing |
| 98 | CPI limits: depth 4, trace 64, 10KB grow, 16×32-byte seeds | runtime |
| 99 | Closed-source: dump, strings, IDL hunt, simulate, RE | ops-testing |
| 100 | Check the code yourself | ops-testing |
