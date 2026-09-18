# Tokens, math, and vaults

Tips 13, 20, 21, 24, 33, 35, 36, 70, 72.

## Checked math (13)

Unsigned wrap (`0_u8 - 1 = 255`) is a loss-of-funds class in finance.

- Default: `checked_add`, `checked_sub`, `checked_mul`, `checked_div` (propagate `None` → error).
- `saturating_*` only when saturation is the specified spec.
- Raw `+` `-` `*` only with a comment proving overflow/underflow is impossible.

## Rounding (20)

Rust integer division **truncates toward zero** (for uints: down).

Protocol-advantage default:

| Computing | Round |
|-----------|--------|
| Tokens **out** to a user | Down |
| Tokens **in** from a user | Up (user pays more, not less) |
| Fees owed to protocol | Down on user-out, never round fees away from protocol in a way users can loop |

Salami-slice: repeat tiny ops that always round to the user. Hunt every `/` and `mul_div`.

## Casts (21)

`1000_u16 as u8` → `232`. `as` float-to-int saturates (`300.0_f32 as u8` → 255); `to_int_unchecked` does not.

Use `u64::try_from`, `from`, `into`. If `as` remains, prove the value fits.

## No floats (24)

Binary floats are not decimal money (`0.1 + 0.2 ≠ 0.3`). Inputs can be `NaN` / `±inf` / `-0`.

Store scaled integers (`42069` cents, or 1e6/1e9 atomic units). Never `f32`/`f64` in account data or ix args.

## SPL Token facts (33)

- **Freeze authority** on mint: can freeze accounts. Production mints usually revoke freeze + mint auth.
- **Delegate:** owner can approve a spend cap; rarely used in production programs.
- **Token-program multisig:** exists; production uses Squads/Realms instead.
- **Closeable token accounts:** user may close anytime. Do not persist user TAs in a list for later batch airdrop — one close fails the ix. Store **authority**; accept any of their TAs.
- **Ownership transfer:** TA owner can be reassigned (hides transfer / can skip some fee designs on classic Token). Token-2022 ATAs with immutable owner block this.

## Associated token accounts

See [accounts.md](accounts.md) tips 15 and 32. `init_if_needed` / `CreateIdempotent`.

## Wrapped SOL (36)

Native mint: `So11111111111111111111111111111111111111112`.

Wrap: create TA for native mint → transfer SOL in → `SyncNative` (amount = lamports − rent). Unwrap: **close** the TA (lamports return as SOL).

If the protocol already handles SPL tokens, prefer wSOL over a second native-SOL code path.

## Token-2022 (35)

Same base ops as Token, plus extensions. **Decide a support policy per extension** and reject mints that enable hostile ones.

Treat as dangerous until proven otherwise:

| Extension | Why |
|-----------|-----|
| Permanent delegate | Can drain protocol vaults |
| Transfer hook | Arbitrary CPI / reentrancy / fail withdrawals |
| Transfer fees | Deposit ≠ credited amount; withdraw ≠ sent amount |
| Closable mint | Mint disappears under you |
| Default account state / freeze | Deposits can be frozen |
| CPI guard / memo required | Unexpected ix failures |
| Non-transferable | Cannot withdraw |

Harmless-ish (still check): metadata pointer, interest-bearing (non-functional display), confidential transfers (integration complexity).

On every new mint: parse mint extensions and **hard-fail** disallowed ones.

## Donation attacks (70)

Anyone can transfer tokens into a vault ATA or add lamports to any account. If share price / TVL uses `token.amount` or `account.lamports()`, an attacker inflates the denominator or triggers rounding (often with tip 20).

- Keep an **internal** `total_deposits` / per-user tracked balance.
- Ignore sudden external increases, or explicitly handle them as unattributed (protocol-owned).
- When internal count hits 0 on last withdraw, withdraw **all** remaining tokens so the TA can close (dust from donations otherwise sticks).

## Vault topology (72)

| Design | How | Tradeoff |
|--------|-----|----------|
| Unified vault | One PDA `"vault"` (and its ATAs) holds all pools/users | Easy TVL; one bug drains everything; write-locks the same TA |
| Per-pool / per-user vault | Each pool PDA owns its ATA; pool signs withdraws | Isolates loss; easier proof-of-reserves; harder TVL aggregation; better parallelism (tip 12) |

**Default: per-pool vaults.** Do not use a global signer PDA that can spend every pool.

## Combining with other tips

- Shard fee treasuries if a single fee ATA is write-hot (tip 12, [runtime.md](runtime.md)).
- Never `init` the protocol's ATA (tip 15).
- Separate payer from authority when creating TAs (tip 16).
