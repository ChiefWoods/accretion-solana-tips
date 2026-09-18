# Operations, time, testing, and tooling

Tips 17, 18, 19, 22, 29, 31, 34, 52, 54, 55, 58, 64, 95, 96, 97, 99, 100.

## Errors and tests (17, 95, 96)

Custom error **per failure** (constraints, validators, `?` sites). Write **at least one test per error**. Coverage follows from exhaustive errors.

Testing stack — pick what makes you write more tests; auditors should be fluent in all:

| Tool | Role |
|------|------|
| Rust `#[test]` | Pure helpers, math, packing |
| `solana-program-test` / cargo-test-sbf | Full ix in SBPF, in-process |
| `anchor test` (TS + mocha) | Local validator; `--detach` to inspect in an explorer |
| LiteSVM | Fast VM; warp clock, edit accounts, skip sigverify |
| Mollusk | Minimal harness; `process_instruction` + `Check`s; CU benches |

Fuzz (96): random ix data **and** account graphs; then **sequences**. Trident for Solana-aware fuzz; libFuzzer for rust functions. Start with one critical ix overnight.

## Lints (29)

Periodic, not every compile:

```bash
cargo clippy --all -- -W clippy::all -W clippy::pedantic -W clippy::restriction -W clippy::nursery -D warnings
```

Most hits are noise. Keep the rare real ones.

## Authorities / multisigs (18)

Upgrade authority and admin authorities: Squads (or equivalent), not a hot key.

Config: `n = floor(m/2)+1` of `m` → 2/3, 3/4, 3/5, 4/7. **Not** 1/m (any key is god) or m/m (one loss/hostage bricks it). Smallest sensible: 2/3. 2/4 allows two disjoint majorities. Extra keys can model extra board seats.

## Incident plan (19)

Write before the exploit:

- Contacts: counsel, law enforcement, auditors, CEX/bridge/stablecoin freeze desks, stakeholders, secure channels
- Roles: fix / comms / forensics
- Pause path: mode ix (tip 6) **and/or** prebuilt freeze upgrade
- Lawyer-reviewed user messages

Audited programs still get drained.

## Monitoring (22, 54)

Poll IDL-decoded global accounts; watch ix volume, TVL, fees, new accounts, events.

**Sequence numbers** on events (increment a counter when emitting) let indexers detect gaps. Cost: extra write lock. Timestamp-only sorts but cannot prove completeness. Prefer `emit_cpi!` (tip 11), never raw log scrape (tip 41).

## Cranks (55)

Programs cannot self-wake. Permissionless crank ixs advance time-based state.

Audit:

- Can two crankers produce different end states from the same start?
- Sandwich / first-after-crank edge (prices, rewards)
- Missed cranks vs over-incentivized cranks
- Multiple cranks drifting out of sync

Often better: call the crank helper from hot user ixs (pass the extra accounts).

## Clock (58)

`Clock::get()?` — slot, epoch, epoch-start timestamp, unix timestamp.

Unix time has **second** resolution and is **per slot**. Consecutive 400ms slots may share the same timestamp. Validators require non-decreasing timestamps. Expiry `== now` may still be valid next slot.

## Metadata (64)

Metaplex: on-chain name/symbol + URI to off-chain JSON (name/symbol/description/image again).

Revoking metadata update authority does **not** freeze the JSON if the URI is a mutable HTTPS server. Require content-addressed URIs (ipfs/`ipfs://<cid>` or arweave tx id). Wallets may display JSON over on-chain fields.

## IDLs and CLI (31, 34)

IDL → Rust `declare_program!()` / solores; TS Anchor client; [bettercallsol.dev](https://bettercallsol.dev).

```bash
anchor account --idl some.json targetprogram.TargetAccount <PUBKEY>
solana account $PUBKEY
solana transaction-history $PUBKEY
solana confirm -v $SIGNATURE
solana block $SLOT
solana logs
solana feature status
```

## Protocol changes (52)

Read SIMDs on GitHub. `solana feature status` shows feature-gate pubkeys. Testnet activates first. Re-verify this skill against current runtime when behavior matters (tip 100).

## Unified toolchain (97)

The post described an alpha CLI named **mucho** (install, validator, inspect, `Solana.toml`, tokens, fuzz/coverage). Treat the brand as historical — use the **current** official Solana/Agave + Anchor CLIs. Inspect pubkey/signature/slot with whatever `inspect` the installed toolchain provides.

## Closed-source programs (99)

```bash
solana program dump <PROGRAM_ID> program.so
llvm-objdump --print-imm-hex --source --disassemble program.so
strings program.so
anchor idl fetch <PROGRAM_ID>
```

Then: IDL in docs/SDK/frontend; CPI graph from landed txs; Dune on discriminators; devnet (more logs, maybe not identical); `simulateTransaction` (can skip valid sigs); Ghidra/IDA/Binary Ninja. Not a substitute for source + audit.

## Check the code (100)

Do not treat Twitter, this skill, or docs as absolute truth. Read runtime/SDK source, run a repro, then decide.
