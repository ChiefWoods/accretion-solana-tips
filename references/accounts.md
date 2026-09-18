# Accounts, PDAs, layout, and init/close

Tips 2, 7, 8, 9, 15, 16, 26, 32, 38, 40, 42, 43, 49, 50, 51, 57, 67, 73, 77, 91.

## Account model (42, 73)

Accounts are ledger files: pubkey, owner, lamports, data, executable. **Only the owner may write data or decrease lamports.** Anyone may *increase* lamports (donation / griefing).

| Type | Role |
|------|------|
| `AccountInfo` | Runtime view: pubkey, lamports, data, owner, signer/writable/executable |
| Deserialized `Account` | Parsed `data` |
| `AccountMeta` | CPI/tx description: pubkey + is_signer + is_writable |

Always check **owner** (and discriminator) before trusting data. Default owner is System Program until assigned. After assignment, the private key of that pubkey cannot write the account; only the program (or PDA `invoke_signed`) can.

System Program is what requires a signature to allocate/assign new accounts — not the SVM itself.

## Discriminators (38)

Anchor: 8-byte `sha256("global:<ix>")` / `sha256("account:<Name>")`. Native programs must put a unique prefix on every account type and check it on load.

Token program uses **size** (Mint 82, TokenAccount 165, Multisig `3+n*32`) instead of discriminators. Do not copy that pattern if you also have variable-length accounts that can collide on size.

## Constraints (2, 43)

- Prefer `has_one` for key equality.
- If the check is more than a key compare, use a dedicated validation function + custom error.
- Always attach custom error codes to constraints.
- `has_one` + `seeds` on the same immutable field is often redundant CU. Keep both if the field might become mutable later.

## PDA seeds (9)

```
"pool:"  + mint.key()  [+ id]
```

Rules:

1. Static string prefix first, with a terminator (`"pool:"`) so `"pool"` is not a prefix of `"pool_admin"`.
2. Then pubkeys.
3. Then numeric ids if needed.
4. Variable-length strings/bytes last, and only if unavoidable.
5. No extra seeds.

Max per PDA (tip 98): 16 seeds, 32 bytes each.

## Layout and padding (7, 8)

- Fixed-size fields at the front; `Option` / `Vec` at the end so RPC `memcmp` filters have a static offset.
- Reserve `_reserved` / padding. One zero byte is enough to grow via `Option::None` → `Some`. For a stable size, reserve ≥64 bytes.

## Payer vs authority (16)

Two signers: `authority` (protocol permission) and `payer` (rent / protocol fees). Same pubkey may be passed for both. PDAs acting as authorities often cannot pay rent. Tx fee payer is a third concept (tip 45).

See: https://developers.metaplex.com/guides/general/payer-authority-pattern

## ATA (15, 32)

Associated Token Program PDA: wallet + token program + mint. Indexers treat it as the canonical token account.

- Anyone can create an ATA for any authority. Anchor `init` then **fails forever** → mint-seeded pools can be DoS'd. Use **`init_if_needed`**. ATA program: `CreateIdempotent`.
- After init, SPL Token allows **owner reassignment**. Token-2022 ATAs use immutable owner, so the derived wallet stays owner.
- Derive: `solana find-program-derived-address ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL pubkey:$WALLET pubkey:$TOKENPROGRAM pubkey:$MINT`

## Account creation griefing (26)

`system_instruction::create_account` requires **0 lamports**. Anyone can transfer 1 lamport to a known PDA and make init fail.

Do what Anchor does: `allocate` + transfer remaining rent + `assign`. Never raw `create_account` for PDAs.

## Closing / revival (40)

Zeroing lamports is not close. Attacker re-funds before tx end → account survives (whitelist tickets reuse).

Correct (Anchor `close`):

1. Drain lamports to destination
2. Assign owner to System Program
3. Realloc to 0 bytes

Zeroing data + a "deleted" discriminator is strictly weaker; the account still exists.

## Rent (77)

Rent-exempt deposit ≈ 2 years. Feature `CJzY83` disabled rent collection; creating a non-exempt account fails. `solana rent 100` prints the exempt amount. Closing returns the deposit.

## Remaining accounts

See [security.md](security.md) tip 10. They are unchecked. Prefer typed accounts. If used, validate count, owner, discriminator, seeds.

## Option accounts (67)

`Option<Account>` **None** is implemented by passing **this program's `program_id`**. Cost is one extra account index byte, not a new pubkey.

Consequence: an `Option<AccountInfo>` used to "register a program" can never register the currently executing program — it always parses as `None`.

## Zero-copy (49)

`#[account(zero_copy)]` + `AccountLoader<>`. `load_init()` on init; `load()` / `load_mut()` after.

Overlays the byte buffer — no deserialize copy. **Discriminator is not checked until load.** Skipping `load` is account-confusion.

`#[account(zero)]` = preallocated zeroed keypair account (tip 50).

## Large accounts (50)

- Max account size: 10MB (~73 SOL rent historically; re-check `solana rent`).
- CPI may grow an account by **≤10240 bytes**.
- PDAs are created via CPI → grow in 10KB steps (or use a keypair account allocated outside the program, then `#[account(zero)]`).

## Stack / heap (51)

4KB stack, 32KB heap (bump allocator, tip 53). Anchor deserializes accounts onto the stack.

Mitigations: `Box<Account<…>>` (heap), split into non-inlined functions (new stack frames), zero-copy, or omit unused accounts (then you must validate remaining accounts yourself).

## LazyAccount (57)

Anchor ≥0.31, feature `lazy-account`. Replaces `Account<>` to load individual fields (`load_authority()?`) without full deserialize. Still checks owner + discriminator. `load()` / `load_mut()` for full account. CU win is mainly **immutable** field reads; mut currently deserializes the whole account.

## Duplicate accounts (91)

- Readonly twice: fine.
- Writable + readonly: the account is writable for the tx; Anchor skips serializing the readonly copy; **writable write wins**.
- Writable twice: **last write in Accounts struct order wins**.
- `init` twice: "already in use".
- `init_if_needed` twice: first init does not write discriminator until ix end → second hits discriminator `0` and fails.

## Types recap

When building CPIs, set `AccountMeta` flags explicitly. Do not pass writable/signer unless required (tips 28, 88).
