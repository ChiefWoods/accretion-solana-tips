# Program design, structure, and Rust craft

Tips 1, 3, 25, 37, 39, 44, 65, 66, 69, 74, 78, 80, 82, 84, 85, 90, 93.

## Project layout (1)

Do not keep a production program in one `lib.rs`. Split:

```
src/
  lib.rs              # declare_id, program module, dispatch only
  instructions/
    mod.rs
    init.rs           # Accounts + validation + handler
    transfer.rs
  state/
    mod.rs
    global.rs         # account def + impls for that account
```

`lib.rs` only wires handlers. Each instruction file owns its `#[derive(Accounts)]`, extra validation, and business logic. Each state file owns one account type.

Generate with: `anchor init --template multiple`.

Study real programs (3): [Squads v4](https://github.com/Squads-Protocol/v4) for Anchor basics; Sanctum S and Ellipsis Labs Plasma/Gavel for non-Anchor.

## Multi-program architecture (25)

For large systems, split by privilege (MetaDAO pattern: autocrat / conditional vault / AMM).

**Why:** a program can only write accounts it owns. Splitting means vault code cannot mutate proposal accounts.

**Costs:** call depth 4, max ~63 CPIs (tip 98), coordinated upgrades. Use a global "updating" operating mode (tip 6) during rollouts. Do not over-CPI.

## Naming (80)

| Thing | Pattern | Avoid |
|-------|---------|--------|
| State accounts | Capitalized noun: `User`, `Global`, `Pool` | `Config` (say `FeeConfig` / `GlobalConfig`) |
| Instructions | `subject_verb_object`: `user_withdraw_lp`, `admin_collect_fees`, `public_crank_market` | `withdraw`, `new_user`, `crank` |
| Account fields | `authority` or `payer` (payer = rent/fees only) | `owner` |
| Mints | `mint` | `token` |
| Numeric fields | `locked_fee_lamports`, `fee_bps` | `locked`, `amount` |
| Enums | `ProposalState::Voting` not `Live` / `Proposed` |

Ambiguous names force every reader to re-open implementations.

## State machines (84)

If the protocol has phases, encode them as an enum with transition methods — do not infer phase from `total_raised` + `clock`.

```rust
pub enum LaunchState {
    Initialized,
    Collecting,
    Launched { committed: u64 },
    Failed,
}

impl LaunchState {
    pub fn start_collecting(&mut self) -> Result<()> { /* only from Initialized */ }
}
```

Put data that only exists in one phase on that variant. Drive instruction logic with `match`. Unclear transitions are a top bug class.

## Nested Accounts and traits (65, 66)

Reuse constraint sets:

```rust
#[derive(Accounts)]
pub struct AdminAction<'info> {
    pub global: Account<'info, Global>,
    pub admin: Signer<'info>, // has_one / seeds once here
}

#[derive(Accounts)]
pub struct Pause<'info> {
    pub admin_action: AdminAction<'info>,
    // ...
}
```

Define traits on account bundles for shared authorization (Light Protocol, GMX). Copy-paste constraints drift.

## Docs and errors (39)

`///` docstrings on structs, fields, and helpers. Anchor copies them into the IDL. `//` for local justification (especially math). Document the formula next to financial math.

## Anchor fallback (37)

One fallback per program, native entrypoint signature, runs when no ix discriminator matches:

```rust
pub fn fallback(program_id: &Pubkey, accounts: &[AccountInfo], data: &[u8]) -> ProgramResult {
    Err(ProgramError::InvalidInstructionData)
}
```

Default: reject. Only implement if you intentionally handle unknown discriminators.

## Entrypoint (85)

Native:

```rust
#[cfg(not(feature = "no-entrypoint"))]
entrypoint!(process_instruction);
```

Depend on other Solana programs with `no-entrypoint` so you do not register two entrypoints. Anchor's `#[program]` defines the entrypoint. Runtime finds murmur3("entrypoint") = `0x71E3CF81`, not the ELF entry.

## Pinocchio, C, assembly (44, 90)

Pinocchio (`anza-xyz/pinocchio`) replaces `solana-program` with almost no deps — smaller binary, less CU. **You** must implement owner/signer/discriminator checks.

C (`solana_sdk.h` + `sol_deserialize`) and sBPF assembly (e.g. deanmlittle/sbpf) are valid for size/CU-critical paths. Prefer Rust + small `unsafe`/inline asm over a full-C program unless the user asks.

## Lifetimes (78)

`Context<'a, 'b, 'c, 'info, T>` holds references: `program_id`, `accounts`, `remaining_accounts`. `'c: 'info` means remaining-account refs must outlive `AccountInfo` data. `'info` is the same lifetime on `#[derive(Accounts)]`. Do not invent lifetime gymnastics; follow Anchor's signatures.

## Borrow checker (69)

Use block scopes to drop immutable borrows before `load_mut` / lamport borrows. Prefer scope over `clone` or `unsafe`.

## unsafe (74)

Allowed operations: raw pointers, unsafe fns, mutable statics, some traits, unions. Typical Solana use: overlaying account bytes.

Rules: smallest possible block, comment the invariant, extra audit attention. Never to silence the compiler. Check alignment, length, and that no overflow produces OOB pointers.

## Macros (82)

DRY fee math, transfers, checked arithmetic, assertions. Prefer one sound implementation over 10 copies.

Kinds: declarative (`macro_rules!`), procedural, derive (`#[derive(Accounts)]`), attribute (`#[account(mut)]`).

Auditors: `cargo install cargo-expand && cargo expand` to read generated code.

## Type safety (93)

`type Lamports = u64;` does **not** prevent mixing token amounts and slots.

```rust
pub struct Lamports(pub u64);
pub struct TokenAmount(pub u64);
```

Implement `apply_fee` etc. on the newtype. Access via `.0` or a method.

## References

- Squads v4: https://github.com/Squads-Protocol/v4
- Pinocchio: https://github.com/anza-xyz/pinocchio
- sBPF: https://github.com/deanmlittle/sbpf
