# Security patterns

Tips 4, 5, 6, 10, 14, 23, 27, 28, 30, 41, 48, 59, 60, 71, 75, 81, 88, 94. Related: 15, 26, 40, 68, 70, 83 in other files.

## Invariants and assertions (4, 5)

**Invariants** are methods on state that must hold after every mutation (Squads v4 spending-limit: amount ≠ 0, members unique and non-empty). Call at the **end** of the instruction so a bad transition reverts.

**Dynamic assertions:** snapshot values at entry, assert the intended delta at exit (deposit: vault token amount must not decrease). Cheap, catches entire classes of transfer bugs.

## Kill switch (6)

Global state account with an operating-mode enum: `Normal`, `Halted`, `WithdrawOnly`, `Limited`.

- Changeable by an **admin key distinct from upgrade authority**.
- Pass as **read-only** on most instructions.
- Enforce in every mutating ix (or via a nested `AdminAction` / mode-check accounts struct).

A mode flip lands when an emergency upgrade may not.

## Remaining accounts (10)

`remaining_accounts` are unchecked. On every use:

1. Expected **count** (and grouping, e.g. mint+ata pairs)
2. **Owner**
3. **Discriminator**
4. PDA **seeds** if PDA
5. Semantic **data** checks

Missing owner checks after deserialize are a recurring critical finding.

## Self-reentrancy (14, 88)

Runtime forbids `A → B → A`. It **allows** `A → A`.

Programs that `invoke` a **user-chosen** program (multisig, DAO, some flashloans, bridges) can be told to call themselves mid-execution (e.g. execute proposal → CPI into same program → `relinquish_vote` → proposal no longer executable, then marked executed).

Mitigations:

- Reject callee `== crate::ID`
- Or allow only a whitelist of ix discriminators
- Do not pass writable program-owned accounts into arbitrary CPI
- Mark everything possible read-only
- Omit accounts the callee must not see/change (signer bits **propagate** to the callee)

`invoke` and `invoke_signed` both forward parent signatures. `invoke_signed` additionally signs PDAs.

## Arbitrary / wrong-program CPI (27, 28)

If `invoke_signed` targets an attacker program, that program receives **your PDA signature** and can drain every account that PDA can sign for.

- Hardcode or constrain callee program IDs
- Least privilege: readonly unless a write is required
- Upgradeable callees: require their upgrade authority to be a multisig; assume they can become malicious
- Immutable + audited callees are the lowest counterparty risk

## Empty collections (30)

```rust
for item in &vec { validate(item)?; }
// runs the "success" path when vec is empty
require!(!vec.is_empty(), ErrorCode::Empty);
```

## Access gates (23)

| Pattern | Notes |
|---------|--------|
| NFT | Check collection **and** `amount == 1`. Empty token account bypasses ownership-only checks. |
| Whitelist PDA | `["whitelist:", user.key()]` created by admin; user signs. |
| Merkle | One root on-chain; inclusion proof. Prefer over a `Vec` of pubkeys. |

## Logs are not events (41)

Never parse `msg!` / program logs off-chain.

- Log injection via user-controlled strings
- Failed txs still emit logs
- CPI caller can emit lookalike logs
- Truncation at ~10KB

Use authenticated `emit_cpi!` / noop CPI (tip 11).

## Slot/epoch boundaries (48)

Last ix before a boundary + first ix after (Jito bundles, leaders) can sandwich state transitions keyed off slot/epoch. Do not give that position an unfair claim, price, or reward.

## TOCTOU on offers (59)

`accept_offer(offer_account)` is bait-and-switch if the maker can update, close+reopen, or replace the account at the same address.

The take ix must pin: `offer_account` **and** `min_price`, `mint`, `amount`, expiry, etc. Same class as frontrun-init (75).

## Pool squatting / graduation (60)

Launchpad that CPIs `initializePool` at a deterministic PDA (`"pool" + mint_a + mint_b`) can be frontrun. Graduation fails; funds stuck.

Prefer pools that init at a **non-deterministic** address, or graduate by adding liquidity to an existing pool (price-match first — usually worse). Design graduation to survive a pre-existing pool.

## Frontrunning (75, 87)

Assume every ix is seen by a hostile RPC/leader before it lands.

High-value: account **init** at a known address (attacker inits with their settings; victim keeps using it), and any ix whose result depends on mutable external state (oracles, pool price, whitelist).

Mitigate with pinned params, `init_if_needed` + ownership checks, slippage bounds, and not assuming "we win the race to init".

## Randomness (71)

Chain is deterministic. Leaders order txs. Users add a revert-if-loss ix after a coinflip.

Do **not** seed from blockhash, slot, timestamp, account data, or user entropy. Use a verifiable oracle or a cryptographer-reviewed commit-reveal. Default: **no on-chain RNG**.

## Transient owner (94)

```rust
require_keys_eq!(account.owner, other_program::ID);
// later save account.key() as "an other_program account"
```

Attacker assigns a **zero-lamport** system account to `other_program` for this tx only. Runtime GC returns it to System after the tx. Later they own that key and write attacker data.

Owner check ≠ "this account was initialized by that program." Check discriminator + data layout + (if PDA) seeds. Do not persist addresses that only passed an owner check inside one tx.

## Backdoors and trust (81)

Default backdoor: **upgrade authority**. Also: unbounded fee updates, code in tests/deps, accounts initialized by an old program version then wiped from source.

Raise assurance: freeze or strict multisig upgrade auth; fresh deploy key; pinned deps; fee caps even admins cannot exceed; audits that hunt backdoors; (optional) formal verification. Still: a determined author can hide a rug — say so when reviewing unaudited upgradeable programs.

## Related hard checks

- ATA `init` DoS — [accounts.md](accounts.md) tip 15
- `create_account` lamport grief — accounts tip 26
- Account revival — accounts tip 40
- Lamports then CPI — [runtime.md](runtime.md) tip 68
- Donation / share inflation — [tokens-math.md](tokens-math.md) tip 70
- Stale copies across CPI — runtime tip 83
