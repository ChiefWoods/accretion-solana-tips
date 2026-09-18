# Review checklist

Run on every new instruction, CPI, token integration, or audit pass. Cite failed tip numbers in findings.

## Accounts and init

- [ ] Owner + discriminator checked before use (38, 42)
- [ ] PDA seeds: `"prefix:"` + pubkeys + id; no prefix collisions (9)
- [ ] Seeded ATA uses `init_if_needed` / `CreateIdempotent`, not `init` (15)
- [ ] No `create_account` on known PDAs (26)
- [ ] Close uses defund + System assign + realloc 0 (40)
- [ ] Remaining accounts: count, owner, disc, seeds (10)
- [ ] `Option<Account>` None is `program_id` — no self-register bug (67)
- [ ] Zero-copy accounts are actually `load`ed (49)
- [ ] Payer and authority are separate signers (16)
- [ ] User token accounts are not stored; authorities are (33)
- [ ] Duplicate writable accounts: last write wins, and that is intended (91)

## Math and tokens

- [ ] No raw `+` `-` `*` on financial values (13)
- [ ] Every division rounds to protocol advantage (20)
- [ ] No `as` truncations; no floats (21, 24)
- [ ] Token-2022 mint extensions allow/deny list enforced (35)
- [ ] Share/TVL math uses internal balances, not `token.amount` (70)
- [ ] Vaults are per-pool (or justified global vault) (72)

## CPI and runtime

- [ ] Callee program ID constrained (27)
- [ ] No unexpected self-CPI; arbitrary invoke is least-privilege (14, 88)
- [ ] Writes serialized before CPI; foreign accounts reloaded after (83)
- [ ] No direct lamport mutation before CPI (or all mutated accounts passed in) (68)
- [ ] Sysvars via syscall, not unchecked accounts (61)
- [ ] CPI depth / 10KB grow / trace length considered (98)
- [ ] Hot write-locked accounts sharded if high volume (12)

## State and product logic

- [ ] Invariant / delta assert at end of mutating ix (4, 5)
- [ ] Halt / withdraw-only mode exists and is enforced (6)
- [ ] Explicit state enum + transitions, not ad-hoc time/amount checks (84)
- [ ] Take/accept pins price, mint, amount (59)
- [ ] Init/graduation cannot be squatted or frontrun into stuck funds (60, 75)
- [ ] Empty vecs cannot skip validation (30)
- [ ] No slot/hash/timestamp randomness (71)
- [ ] Cranks: deterministic, no toxic sandwich, or crank inlined (55)
- [ ] Slot/epoch edges do not grant first/last privilege (48)

## Ops

- [ ] Custom errors on every failure; tests for each (17)
- [ ] Upgrade/admin via `floor(m/2)+1` multisig (18)
- [ ] Events via `emit_cpi!`, not parsed logs (11, 41)
- [ ] Fee/admin caps so an admin cannot set 100% fee (81)
- [ ] Instruction names `subject_verb_object` (80)
