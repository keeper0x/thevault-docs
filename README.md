# The Vault: technical brief for the Solana Foundation

*Written for Solana developers and the Solana Foundation's reviewers. First
published 28 September 2026; this version is current as of 5 October 2026.*

*The Vault is experimental and unaudited. Every transaction in this brief
links to a public explorer, directly or through the [Evidence](evidence.md)
page, which also lists the figures in its tables and what produced them. The
source code, the tests and the full review record become public at the open
release (milestone 5 of the funded plan, Section 7.1). Until then the
repository is private, and nothing in this brief needs it: the on-chain
evidence is linked now.*

## Summary

* **What it is.** A Solana program whose only spend authority is a
  hash-based signature (generalised XMSS over Keccak-256, 256 one-time
  leaves per key, rotated in place). No Ed25519 key can spend from it; the
  one Ed25519 power left is the program's upgrade key, held by us today and
  moved to a Squads 2-of-3 multisig at milestone 5. The vault can also **execute a Jupiter swap from inside
  itself**: the vault PDA signs the route, the output lands in a
  vault-owned account, and the program checks the outcome on chain before
  the transaction may succeed.
* **Why now.** Transaction v1
  ([SIMD-0385](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0385-transaction-v1.md):
  4,096 bytes, 64 accounts) activated on mainnet on 15 September 2026, at
  slot 447,120,000, the first slot of epoch 1035
  ([feature account](https://explorer.solana.com/address/txv1aq4pp281K9um3tnPgkfX8UqtFT6wcVW3hNezGLL)).
  A 1,037-byte signature plus a real Jupiter route cannot fit in 1,232
  bytes. It now fits with room to spare: the largest swap landed on mainnet
  is 2,608 bytes, and the largest route replayed through the vault is 2,877
  bytes with 45 addresses.
* **Where it is.** On mainnet since 24 September 2026, as an unannounced,
  pre-audit experiment (program
  [`BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo`](https://explorer.solana.com/address/BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo)).
  A canary vault
  [swapped 0.01 SOL for USDC](https://explorer.solana.com/tx/wggJur7CkEwp7J3L5xawJqP1u2nhQ4wYSA8FXqvitS3boaGFsvPApQqewtZr5Xsipne4Gzj4HJCiu3dT6KbjiYM)
  through Meteora DLMM from inside the vault, and the 30 bps protocol fee
  moved to the treasury in the same transaction. A swap signed with an
  unreachable `min_out`
  [was rejected](https://explorer.solana.com/tx/uVygn8g4t3wZPPGcSKm8i5f4kjDceLBb4qY8EnWDSF4R3meiBk1DvsADYVM3moH3FjyxuAmKDXE23S5fdEiV3tY)
  by the vault's own settlement check. Six swaps from inside a vault have
  landed (24, 27 and 28 September), each paying the fee on chain. A browser
  app has been live since 27 September for our own testing. On mainnet it
  has created a vault, taken a deposit and landed swaps from inside it;
  withdrawing, restoring from 24 words and the program-hash check are built
  and covered by end-to-end tests. There is no traction to report:
  mainnet holds three vaults, all ours. We are not promoting it or seeking
  deposits before the external audit; the public beta, with a deposit cap
  and a bug bounty, opens after it (milestone 8).
* **How it was built.** One owner, working to a fixed process: a written
  specification for each milestone, an implementation with its tests, then
  an internal adversarial review gate that must return CONTINUE before the
  milestone closes. Every number is sourced, and every status uses the
  weakest true word. A first internal assessment and 23 internal review
  rounds since then have recorded 69 findings; the 10 release blockers
  among them are all resolved, and one non-blocking finding is open. Over
  320 automated tests passed in the last recorded run of each suite. The
  internal review is not an audit: the external audit this application
  funds is the independent check.
* **What we are asking for.** USD 111,275 on the convertible track, in nine
  milestones each paid on completion (Section 7.1): reliable network fees, a
  venue-compatibility matrix and native staking, so that the code the audit
  covers is complete (1 to 3); signed review engagements, an audit-ready
  open release, the external program audit and cryptographic review, and
  the fixes (4 to 7); a capped public beta and an adoption target (8 and 9).

## 1. The problem, in Solana terms

Every Solana wallet's authority is an Ed25519 key, and its public key is
on chain from the first signature.
[Shor's algorithm](https://arxiv.org/abs/quant-ph/9508027) recovers the
private key from the public key. There is no "harvest now, decrypt later"
delay for a signature scheme: the day a large enough quantum computer
exists, every exposed key can be signed for and its funds simply move.

Solana's own plan is deliberately unhurried. The Foundation's
[quantum readiness note](https://solana.com/news/quantum-readiness)
(27 April 2026) sets out research first, a post-quantum scheme for new
wallets if the threat becomes credible, and migration later, and says no
change is required today. Anza and Firedancer converged on Falcon as the
leading candidate
([Anza](https://www.anza.xyz/blog/securing-solana-against-a-powerful-quantum-adversary),
[Jump](https://jumpcrypto.com/resources/quantum-migration-paths-for-solana)),
and the Falcon verification syscall proposal
([SIMD-0461](https://github.com/solana-foundation/solana-improvement-documents/pull/461))
was closed unmerged on 17 June 2026, to wait for demand. Until the network
moves, opt-in protection built at the application layer is the only
protection a Solana user can choose.

That protection already exists in several forms (Section 6.2), mostly as
developer tools or early betas. To our knowledge, none of it trades from
inside the vault: to swap, rebalance or pay, a holder has to move funds
back behind an Ed25519 key, which undoes the protection for as long as the
funds are there. We expect protection that cannot trade to see little use;
closing that gap is what this project is for.

## 2. What we built

### 2.1 Signature and account model

| | |
|---|---|
| Scheme | DKKW25: Drake, Khovratovich, Kudinov and Wagner, *Hash-Based Multi-Signatures for Post-Quantum Ethereum* ([IACR ePrint 2025/055](https://eprint.iacr.org/2025/055)). Its generalised XMSS tree (Construction 3) over a target-sum Winternitz one-time encoding (Construction 6), with Keccak-256, from Blueshift's MIT-licensed [`solana-winternitz`](https://github.com/blueshift-gg/solana-winternitz) crate, pinned at commit [`d6c381a46929d2f8973619f38b66d79058c39b5a`](https://github.com/blueshift-gg/solana-winternitz/commit/d6c381a46929d2f8973619f38b66d79058c39b5a) |
| Parameters | 36 chains, w = 4 bits (16 positions per chain), target sum 297, 23-byte chain elements, 18-byte public parameter P, tree height 8 (256 leaves) ([crate constants](https://github.com/blueshift-gg/solana-winternitz/blob/d6c381a46929d2f8973619f38b66d79058c39b5a/src/lib.rs)) |
| Verifying key / signature | 41 bytes (root 23 ‖ P 18) / 1,037 bytes |
| Verification | About 36,500 CU per the crate's [README](https://github.com/blueshift-gg/solana-winternitz/blob/d6c381a46929d2f8973619f38b66d79058c39b5a/README.md) at the pinned commit; 36,420 CU when we ran the crate's own example program (verify plus bitmap write; mollusk-svm 0.15.1, SBPF v3). In this program a complete Execute with no inner instructions measures 37,695 CU, and a replayed leaf is rejected after 907 CU |
| Security target | 128-bit classical, 64-bit quantum, the scheme's own target, asserted by the crate's [parameter test](https://github.com/blueshift-gg/solana-winternitz/blob/d6c381a46929d2f8973619f38b66d79058c39b5a/src/tests.rs#L84). It is not the NIST XMSS profile ([RFC 8391](https://www.rfc-editor.org/rfc/rfc8391), [NIST SP 800-208](https://csrc.nist.gov/pubs/sp/800/208/final)), and the crate's README says it is not compatible with RFC 8391. We claim no NIST level |
| Account | one PDA per vault, `["vault", vault_id]`, 128 bytes zero-copy: tag, version, bump, `scheme`, `tree_index`, op counter, `vault_id`, current verifying key, 256-bit used-leaf bitmap |
| Address | `vault_id = keccak256("THEVAULT:ID:1" ‖ genesis_key)`, so the address commits to its first key: Initialize needs no hash-based signature and cannot be front-run into a foreign key |

The address is stable for life. Rotation replaces the verifying key,
clears the bitmap and increments `tree_index` in the same instruction, so
a key can never be exhausted: the last free leaf may only be spent by an
operation that installs the next key. Close leaves a program-owned
tombstone rather than deleting the account, because re-initialising a
closed address would reinstall the genesis key with an empty bitmap and
make every old signature replayable. The `scheme` byte reserves room for a
different signature scheme later.

Assets sit in ordinary associated token accounts whose owner is the vault
PDA, under SPL Token or
[Token-2022](https://www.solana-program.com/docs/token-2022). Depositing
needs no vault signature: any wallet sends SOL to the vault's address or
tokens to its token account, and SOL that arrives unwrapped is wrapped by
a later Execute. There is no admin instruction, no config account and no
address lookup table.

### 2.2 One instruction authorises everything

`Execute` carries one signature (one leaf) and a header of flags:
`NEXT_KEY` (rotate), `LAMPORT_MOVE`, `SETTLE` (swap settlement),
`BOUNDED`, `EXECUTOR`, plus a mandatory expiry slot. Up to eight inner
instructions follow, each a CPI the vault PDA signs.

The signed digest binds the program id (so the deployment), the vault id
and the vault's account address, the key generation, the leaf, every
header byte, every aux account address (up to 16 accounts the header
refers to by index: a transfer destination, the executor, the two swap
token accounts, the treasury's fee account), and, per inner instruction,
either every account address and data byte (EXACT, the default) or only
the program, discriminator and the authority, source and destination
slots (BOUNDED). BOUNDED exists so that a swap whose route went stale can
be re-sent with a fresh route under the same signature. It is safe only
because the settlement check below bounds what any route can do.

The vault PDA is promoted to signer inside a CPI only where the account
address equals the PDA; outer transaction signers are never promoted.
Inner programs and discriminators are compile-time constants: Jupiter v6's
`route`, `shared_accounts_route` and their v2 forms (program
[`JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4`](https://explorer.solana.com/address/JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4);
[Jupiter docs](https://developers.jup.ag/docs/get-started)), and
`TransferChecked`, `SyncNative` and `CloseAccount` on both token programs.
For Jupiter, the authority slot must be the vault and the source and
destination slots must be the signed token accounts. The staking upgrade
of milestone 3 adds the Stake program's `InitializeChecked`,
`DelegateStake`, `Deactivate`, `Withdraw`, `Split` and `Merge`, System
`AllocateWithSeed` and SPL Token `UnwrapLamports` to this list, each with
exact data and account-slot rules (Section 7.3).

### 2.3 The in-vault swap and its settlement check

One user signature, one v1 transaction of four instructions: three
idempotent ATA creates (the output account, the wrapped SOL account, the
treasury's fee account), then `Execute`, which CPIs into Jupiter with the
vault PDA as the route's authority. Before the CPI the program snapshots
both vault token accounts and rejects any other vault-owned token account
in the route. After every inner CPI it requires:

1. output received ≥ `min_out`, and input spent ≤ `max_in` (with the fee,
   when the input pays it);
2. on both accounts, owner still the vault, no delegate, no close
   authority;
3. both mints still owned by the same token program as their accounts, and
   carrying none of the rejected
   [Token-2022 extensions](https://solana.com/docs/tokens/extensions)
   (confidential transfer, permanent delegate, transfer hook, confidential
   mint/burn), re-checked **after** the CPI because a hostile venue could
   close and recreate a mint mid-call;
4. wrapped SOL measured as lamports above the rent reserve, never as the
   token `amount`, with a `SyncNative` on a native output after every
   settled swap.

Any failure reverts the whole transaction, including the used-leaf bit.
This check, not Jupiter's slippage setting, is what enforces the user's
intent: a compromised router, relayer or BOUNDED executor can at worst
deliver exactly `min_out`. One qualification: on 5 October we found two
small gaps in how the check measures lamports. Both are zero for every
vault account that exists today, and the fix is part of the milestone 3
upgrade (Section 10).

### 2.4 The protocol fee, without an oracle

30 bps of every settled swap, and nothing on transfers, withdrawals,
rotation or close. The rate is a compile-time constant, so after the
upgrade authority is set to none it can never be raised by anyone. The
fee goes to a treasury that is itself a vault of this program; its
address is derived on chain, never supplied by the client.

Which token pays is decided by the two mints the user signed: USDC or
USDT when either side has one, otherwise SOL when either side has it,
otherwise the output. On the input side the fee is taken after the swap,
from the amount actually spent, and `max_in` bounds spent plus fee. No
price is ever needed. Mainnet has run this rule since the upgrade of
28 September; the first live input-side fee and the first fee paid in
SOL landed that day (Section 5).

A share of the 30 bps for an app or wallet that brings users is under
design (owner decision in principle, 5 October). It changes the fee path,
so like every program change it must land before the code freeze, and it
is in the audit's scope (milestone 4).

### 2.5 Transactions, relayer and client

Every operation is one transaction with one Ed25519 signature: a
disposable relayer key generated in the browser, which pays fees and rent
and can delay or censor but cannot change what executes. It never signs a
vault instruction and is never promoted inside a CPI. It starts empty; the
user funds it from any wallet (the app calls it the gas tank), and it
holds only enough SOL for fees and rent, never vault assets. Milestone 1
replaces this key with one network-fee wallet derived from the 24 recovery
words, so every device that restores the vault uses the same fee wallet
and one approval funds both the vault and its fee reserve; the derivation
gets a design review before any code. The client asserts the 4,096-byte
and 64-address limits, a single signature and the compute-unit and
loaded-data limits before signing. Every transaction carries a small fixed
priority fee set by the client; it does not yet follow the market.

Keys come from a 24-word
[BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
phrase. Each tree is derived from the phrase, the program id (so keys for
different deployments are unrelated; Section 10 covers the period when
devnet shared the mainnet id) and `tree_index`; salts are derived too, so
a retry reproduces identical bytes instead of a second signature over the
same leaf. A local leaf ledger reserves a leaf atomically before any
signature exists and never signs while an earlier attempt on the same
tree is unresolved. Recovery on a new device is one fixed message per
tree: leaf 255, rotate to the next tree, no expiry. Every restore of that
tree, on any device, signs the identical digest and emits identical
bytes, so a lost or duplicated restore cannot make a key sign two
different messages.

Rust and TypeScript clients produce byte-identical keys, digests,
signatures and swap transactions from shared fixtures. The app is a
static site with no backend: it talks to a user-chosen RPC and the
Jupiter API, and before every signature it recomputes the deployed
program's hash from the chain and refuses to sign if it differs from the
program hash published on the [Evidence](evidence.md) page (and, from
milestone 5, in the open release). Anyone can run the same check against
the chain with the commands on the [Evidence](evidence.md) page.

### 2.6 Measured budgets

| Operation | Bytes | Accounts | CU |
|---|---:|---:|---:|
| Initialize (mainnet, landed) | 300 | — | 1,811 |
| Transfer, rotation, close, recovery (devnet, landed 28 September on the build now on mainnet) | 1,270 – 1,323 | — | 38,556 – 39,074 |
| Wrap stray SOL (mainnet, landed) | 1,459 | — | 55,694 |
| Swap SOL→USDC, 1 hop, Raydium CLMM (replay) | 2,159 | 24 | 141,275 |
| Swap SOL→USDC→SPL token, 2 hops, Meteora DLMM (replay) | 2,612 | 37 | 176,141 |
| Swap SOL→Token-2022 token, 1 hop, FluxBeam (replay) | 2,099 | 22 | 171,572 |
| Swap SOL→USDC, Meteora DLMM (mainnet canary, landed 24 September) | 2,163 | — | 129,328 |
| Swap SOL→USDC, fee on the output (mainnet, landed 28 September) | 2,159 | — | 127,782 |
| Swap USDC→SOL→SPL token, 2 hops, fee on the input (mainnet, landed 28 September) | 2,608 | — | 170,490 |
| Swap SPL token→SOL, fee paid in SOL (mainnet, landed 28 September) | 2,130 | — | 128,509 |
| Liquid-staking-token sale, Sanctum Infinity then Whirlpool (replay, 5 October, the mainnet build) | 2,877 | 45 | 248,898 |

"Replay" means the real Jupiter and venue program binaries executed
in-process against pinned mainnet state (Section 4.4). The vault
program's own share of a swap is 53,000–59,400 CU. On the fee-side build
now on mainnet, the first three replayed routes measure 141,371, 178,980
and 164,483 CU. The largest figure in the table, 248,898 CU, is about
17.8% of the
[1.4M CU transaction ceiling](https://solana.com/docs/core/fees/compute-budget).
The client requests 200,000 CU for plain operations and up to 400,000 for
swaps, tightened to 1.2 times the simulated figure before sending. Program
size is 65,504 bytes, the fee-side build on mainnet since 28 September
(64,112 before). Every row links to its transaction, or names the run that
produced it, on the [Evidence](evidence.md) page.

### 2.7 What we reused and what is new

We build openly on MIT-licensed prior work, and the
[Evidence](evidence.md) page lists every reused component with its
upstream repository, the commit studied and its licence. In short:
Blueshift's DKKW25/XMSS verifier, the used-leaf bitmap and the TypeScript
signer ([`solana-winternitz`](https://github.com/blueshift-gg/solana-winternitz/commit/d6c381a46929d2f8973619f38b66d79058c39b5a));
Dean Little's Winterwallet (the digest that binds every passthrough
address and byte, the CPI payload format, state-before-CPI, BIP-39
derivation, the persist-before-send client;
[commit studied](https://github.com/blueshift-gg/winterwallet/commit/672fc6789b1532ee680f24842d235e0be8737b61));
his earlier vault's PDA check
([commit studied](https://github.com/blueshift-gg/solana-winternitz-vault/commit/beb312c7a62914079d68e7eee2c1a3e44bda7a3d));
Qubit's relayer UX and tombstone close
([commit studied](https://github.com/QuantiumResistantInfra/quantum-vault/commit/47e06d5af6fc2c011bdc193eda72be23c9ff6184)).
New here: the vault program itself, the address construction, every
header clause, the digest layout, the allow-list and slot table, the
settlement invariants, the 64-account CPI executor, in-band rotation,
phrase-to-tree derivation including P, the leaf ledger, the v1
transaction builder, the Jupiter plan validator, the fee design and the
app. One studied project has no licence file, and nothing from it is used.

## 3. How we built it

### 3.1 The method

The project has one owner. Every milestone passes the same four steps, in
order:

1. **Specification.** The design, its invariants and the named tests that
   will gate the milestone are written before the code. A change to key
   derivation or to a wire format needs a design review before it is
   coded.
2. **Implementation.** The code, its tests and its documentation.
3. **Internal adversarial review.** A separate pass, made after the
   implementation and kept apart from it, reads, verifies and tries to
   break the work. Each round gives a verdict first (CONTINUE or
   FIX-FIRST), then numbered findings, each with a failure scenario and a
   severity.
4. **Gate.** A milestone closes only on CONTINUE. Every finding is recorded
   with its status; a finding that is not fixed is accepted only with a
   written reason.

The owner runs every step that touches mainnet funds or the deployer key;
each is recorded in a receipt, and its transactions are on the
[Evidence](evidence.md) page. Status uses five words and never a stronger
one than the evidence supports: *proposed, implemented, tested locally,
tested on real programs, internally reviewed*. Evidence distinguishes a
transaction that **landed** from one that was **simulated** and from a
number that was **computed**. We learned to be strict about that
(Section 4.9).

At the open release the output can be checked line by line: 1,784 lines of
program source (the ten source files of the deployed program, counted at
the committed build), over 320 automated tests (in the last recorded run of
each suite: 94 Rust, 140 TypeScript, 104 browser end-to-end), and every RPC
response saved. Today, the on-chain part can be checked on the
[Evidence](evidence.md) page. The internal review is not an audit; the external audit this
application funds is the independent check.

### 3.2 Timeline

| Date (2026) | What happened |
|---|---|
| 15 Sep | Research on five prior projects (six repositories) at pinned commits; core program, allow-listed CPI executor, settlement and BOUNDED mode against a hostile mock router; first internal assessment |
| 16 Sep | Gate 1: the first recovery design was shown to reuse one-time keys, so it was withdrawn |
| 17 Sep | Real Jupiter routes executed through the vault on pinned mainnet program binaries; gate 2 accepts the new recovery design |
| 18 Sep | TypeScript SDK, signer, leaf ledger and recovery, byte-identical to Rust; gate 3 plus three closure rounds |
| 18–19 Sep | Devnet: the first live submission was rejected (wire order), fixed, and the full lifecycle landed; gate 4 plus two closure rounds |
| 24 Sep | Protocol fee; mainnet deploy; treasury initialised; canary swap landed; gate 5, first part (the fee), plus two closure rounds |
| 25–26 Sep | The app; gate 5 full pass plus four closure rounds |
| 27 Sep | Fee side (stablecoin, then SOL, then output); app live; gate 6 found one defect, fixed the same day |
| 28 Sep | Gate 6 closed (two non-blocking follow-ups fixed the same day); devnet moved to its own program id; the fee-side upgrade rehearsed on devnet, then run on mainnet; three swaps from the live app, including the first live input-side fee and the first fee paid in SOL |
| 29 Sep | Design proposal for a network-fee reserve per vault (now milestone 1); its review is pending |
| 2–4 Oct | Public site; the app's entrance rebuilt; four internal review rounds (3–4 Oct) on the entrance and on the lock and key-refresh flows; one non-blocking finding left open; deployed 4 Oct |
| 5 Oct | Native staking: design proposal, two prototypes tested locally against mainnet's Stake and token programs, 39 liquid-staking routes replayed through the vault; two small gaps found in today's settlement check |

A gate is an internal review that must return CONTINUE before a milestone
closes; a closure round re-checks the fixes for a gate's findings.
Starting with the first internal assessment on 15 September, then across
seven gate reviews and twelve closure rounds to 28 September and four
further rounds on 3 and 4 October, the internal review recorded 69
findings. Ten were release blockers, and all ten are resolved. One
non-blocking finding is open by the owner's decision: a recovery that
lands after the app is locked still shows as sent, so the next unlock asks
for one extra key refresh; it needs a design review before it is coded.
Each finding is numbered in the review log, which is published at the
open release; the [Evidence](evidence.md) page lists the ten release
blockers now.

### 3.3 Three times a review changed the design

* **Recovery (gate 1).** The first recovery design chose the recovery leaf
  from the slot. Internal review showed that two restores 16 windows apart
  pick the same leaf and sign different messages: a one-time key signing
  twice. We withdrew the design and replaced it with the single immutable
  recovery message per tree described in Section 2.5.
* **Restores that miss history (three rounds, gate 3 and its closures).**
  Even the new recovery let a device restored from an old backup believe a
  tree was fresh after another copy had exposed a signature on it. Three
  rounds produced the design that ships: a tree marker with a one-hour
  lifetime, atomic reservation, a burned leaf retiring its tree, and every
  reservation tied to a fresh on-chain slot reading.
* **Mint recreated mid-swap (gate 5, first part).** Internal review worked
  out how a venue with authority over a closable output mint could close
  it during the CPI and recreate it with a permanent delegate, so the
  settlement check passed and the delegate could then drain the vault.
  Mints are now re-checked after every CPI, and a regression test builds
  the attack.

## 4. What was hard, and what we learned

The lessons below are the ones we think other Solana builders will hit
too.

### 4.1 Transaction v1 puts the signatures after the message

Our first devnet submission was rejected with `invalid message version`.
A legacy or v0 transaction is `count ‖ signatures ‖ message`. A v1
transaction is `message ‖ signatures`, with no count prefix, because the
count is the message's own `numRequiredSignatures`. Code reused from
legacy serialisation gets this backwards. Agave's RPC decodes with the
[wincode](https://github.com/anza-xyz/wincode) path
([`rpc/src/rpc.rs`](https://github.com/anza-xyz/agave/blob/master/rpc/src/rpc.rs)),
so we pinned `wincode::serialize` as ground truth and confirmed both the
wrong and the corrected wire directly against devnet; the live rejection
is quoted verbatim on the [Evidence](evidence.md) page. A week later the
app's local RPC shim was built to reproduce the same rejection as a
regression test. **Lesson:** test v1 wire bytes against the decoder the
RPC actually uses, not whichever codec compiles first.

### 4.2 The account limit binds before the byte limit, and so does your stack

Hops and venue layouts drive a route's account count. A three-hop quote can
approach 60 unique accounts while still under 3,600 bytes. Inside the
program, a 64-account CPI needs a 3,584-byte `CpiAccount` array in its
own 4 KiB stack frame; it measures 3,640 bytes after compiling and
disassembling (the struct is 56 bytes with
[pinocchio](https://github.com/anza-xyz/pinocchio) 0.11.2 on this
toolchain, not the 48 we first assumed). The client re-quotes down a
`maxAccounts` ladder (64, 48, 40, 32, 24) until the compiled transaction
fits. **Lesson:** for a CPI-heavy aggregator integration, budget accounts,
bytes and your own frame separately, and measure the compiled artifact.

### 4.3 Loaded account data is a per-transaction budget

A wrap that CPIs into the ATA and token programs failed with
`MaxLoadedAccountsDataSizeExceeded` because it was compiled with the small
limit we used for system-only transactions. Every executable a
transaction touches counts, including ours on every call. When the
fee-side upgrade made our program larger, the devnet rehearsal showed
that the app's 64 KiB limit for plain transactions no longer fit; it is
now 128 KiB, set before the mainnet upgrade. **Lesson:** set the
loaded-data limit per transaction shape, not once per program.

### 4.4 No devnet Jupiter: replay mainnet in-process, and say what that proves

Jupiter has no devnet deployment. We snapshot live quotes, every
referenced account, and the executable bytes of every program (with
hashes) at one slot, then replay the exact transaction in
[LiteSVM](https://github.com/LiteSVM/litesvm) with the real venue
binaries, swapping in only our program. That gave program-level evidence
on real routes a week before mainnet. It says nothing about fees,
scheduling or inclusion, and we label it that way; the network-level
evidence came from the mainnet canary. **Lesson:** pinned mainnet replay
is a legitimate test tier for dependencies with no test cluster, as long
as the docs never let "real programs" drift into "on chain".

### 4.5 Not every venue can serve a program-owned user

Two of the first live routes (17 September) failed through the vault. A
venue Jupiter labels "Quantum" takes the Instructions sysvar among its
accounts; reached through the vault it failed with its own error 9 after
944 CU, and the identical route landed for a plain wallet with Jupiter at
the top level. That control changed two things at once (Jupiter as an
inner call, and a program-owned trader), so the mechanism is not
established. In a HumidiFi route, Jupiter itself moved 17,873,280
lamports from the "user" with a plain System transfer. A PDA that carries
data and is owned by another program cannot pay that way
(`Transfer: from must not carry data`), and that route failed for a plain
wallet too, with the venue's own error.

The client therefore quotes from a fixed list of nine venue labels, three
of which had replay evidence when the list was written. Replays on
5 October landed liquid-staking routes through the vault on 11 venue
labels. One of them, AlphaQ, the only venue whose routes name the
Instructions sysvar, landed but paid the vault 0.05 bps less than a plain
wallet, for a reason not yet determined. Milestone 2 publishes an open
matrix, each venue marked working, failing or untested with its reason,
and removes failing venues from routing. **Lesson:** "Jupiter supports
it" does not mean a PDA taker is supported. Build the venue list from
replay evidence, not from the aggregator's labels.

### 4.6 Re-validate everything after an untrusted CPI

Settlement first read wrapped SOL by its `amount`, and took the baseline
before the vault's own top-up. Together these let a BOUNDED executor spend
up to twice the signed budget in the SOL-in flow. Mints were not
inspected either, so a Token-2022 permanent delegate could move tokens
with no vault signature. Later came the recreated-mint attack (Section
3.3). Then, when the fee moved to the input side, a native output was no
longer synced, so a landed swap read "received 0" in the client. Program
state was right; the receipt was wrong. **Lessons:** re-check every piece
of state settlement relies on after the CPI returns, not only balances.
Wrapped SOL is lamports. When you add a code path, re-audit every reader
of the state it changes.

### 4.7 Token-2022 changes both the fee and the rent

A transfer-fee mint can withhold the entire protocol fee in the
treasury's account, where only the mint's withdraw authority can collect
it. We admitted such mints on purpose and rewrote the invariant to name
three quantities: debited, withheld and spendable. Separately, the app
priced every new token account at 165 bytes, but an extension-bearing
mint needed 182, so a signature was reserved before the shortfall
surfaced. **Lesson:** with fee-on-transfer tokens, "fee charged" and "fee
received" are different claims. Size Token-2022 rent from the mint's
extensions.

### 4.8 Stateful signatures fail in ways a stateless signer never does

A failed transaction still publishes its signature, so the one-time rule
is one message per leaf ever, not one landed transaction per leaf. Every
serious finding in the signer came from this rule. It covered a recovery
schedule that repeated leaves, two concurrent calls reserving the same leaf
(durable writes are not atomic decisions), an old backup that forgot an
exposure, and a closed signer that could still sign from a cached key.
**Lesson:** design recovery as a single immutable message, reserve before
you sign, and state honestly what is still residual: two devices using
the same vault at once remain unsafe, and the app says "one device at a
time".

### 4.9 Documentation drifts ahead of evidence

One review round found four places where the docs claimed more than the
evidence: a mistyped activation slot, a slot read after confirmation
reported as the landing slot, a deployed hash with no receipt, and
simulations described as landed. **Lesson:** keep a fixed vocabulary
(landed, simulated, computed) and re-derive every published number from
its archived artifact at review time.

Two operational notes, briefly. Deploying a 64 KB program to devnet with
`--use-rpc` failed five times and stranded a 0.33 SOL buffer each time,
while the CLI's default direct QUIC transport landed first try (extend
ProgramData before upgrading; reclaim buffers with
`solana program close --buffers`; see the
[Anza CLI guide](https://docs.anza.xyz/cli/examples/deploy-a-program)).
And with no price oracle, the only fee-side rule that gives a hostile
executor no new lever is one derived from mints the user already signed.

## 5. Where it stands, 5 October 2026 :id=5-where-it-stands-29-september-2026

| Piece | Status |
|---|---|
| Vault program: hash-signature authority, rotation, close and reopen | implemented, tested locally, on mainnet since 24 September; internally reviewed at gates 1 to 4 |
| In-vault swap, allow-list, settlement checks | tested on real programs (pinned replays); six swaps landed on mainnet (24, 27 and 28 September); two small measurement gaps found 5 October, zero today, fix in milestone 3 (Section 10) |
| Protocol fee, output side | tested on real programs; landed on mainnet 24 September; internally reviewed at gate 5 (first part) |
| Fee side (stablecoin, then SOL, then output) | tested on real programs; internally reviewed at gate 6 (closed 28 September); live on mainnet since 28 September, with a live input-side fee and a live fee paid in SOL |
| Rust and TypeScript SDKs, signer, leaf ledger, recovery | tested locally, byte-identical across languages; internally reviewed at gates 2 and 3 |
| App | live since 27 September for our own testing; tested end to end against the real program through a local RPC shim; internally reviewed at gate 5 (full pass and four closure rounds) and in the four rounds of 3–4 October; one non-blocking finding open |
| Liquid staking tokens (JitoSOL, JupSOL, mSOL, INF) through the swap | tested on real programs (pinned replay, one snapshot); none swapped on mainnet yet; milestone 3 |
| Native staking from the vault | proposed; two throwaway prototypes tested locally against mainnet's Stake and token programs; not yet through the internal review gate; the deployed program is unchanged; milestone 3 |
| Network-fee wallet derived from the 24 words | proposed; design review pending; milestone 1 |
| Venue-compatibility matrix | partial replay evidence (Section 4.5); milestone 2 |
| Reproducible build, multisig upgrade authority, external audit, cryptographic review | not started; milestones 4 to 7 fund them |
| Upgrade authority set to none | not funded; after the beta (Section 7.2) |

"Internally reviewed at gate n" means the work passed internal review
gate n (Section 3.2). It does not mean an external audit.

**Mainnet accounts.** Program
[`BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo`](https://explorer.solana.com/address/BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo)
(deployed 24 September; upgraded 28 September to the fee-side build that
passed gate 6: 65,504 bytes, sha256
`337fe41baebc826919ccde94d8d33b60002c3551af4815ac75e2380e0eb00f0a`, read
back from the chain; the Evidence page gives the command). Treasury vault
[`3VXKr5pcfq8D7nfphRHAt6z8oS7RZHCcNRLrqyxTzo1D`](https://explorer.solana.com/address/3VXKr5pcfq8D7nfphRHAt6z8oS7RZHCcNRLrqyxTzo1D);
canary vault
[`HGTkMLVxZjcGo3reUhBAVy2DYuyJA2Vd8Kr1NM5zg1SK`](https://explorer.solana.com/address/HGTkMLVxZjcGo3reUhBAVy2DYuyJA2Vd8Kr1NM5zg1SK);
live-app test vault `AJo49aZkuGP53fLrT1mYtQ8eRYPFqtBfJZEEJ2yrzp8C`. The
upgrade authority is still a single deployer key and moves to a
[Squads](https://squads.xyz) 2-of-3 at milestone 5, before any public
beta.

**On chain** (read 5 October 2026): three vaults, all ours, and 18
transactions that reference the program: 17 succeeded, and one landed as
a failure by design (the rejected swap below). The latest is from
28 September; none has landed since. They include:

* the deploy, the treasury's initialisation and the mainnet canary of
  24 September: 10,000,000 lamports in, 1,170,571 USDC base units gross,
  fee 3,511 = ⌊gross × 30 / 10,000⌋ to the treasury, 1,167,060 to the
  vault;
* the same run's deliberately failing swap, signed with `min_out` at twice
  the quote, which landed carrying the vault's own `SettleFailed`
  (`Custom(14)`) at the Execute instruction while Jupiter's own call
  succeeded;
* the recovery that followed, then the real swap;
* our own test of the live app on 27 September, from a third vault: a
  wrap, then 0.1 SOL swapped into a Token-2022 token and all of it back to
  USDC. Each swap paid exactly 30 bps of its gross output to the treasury
  (7,350,582 token units, then 36,167 USDC base units);
* on 28 September, the fee-side upgrade (a ProgramData extension, then the
  upgrade; the bytes read back hash to the build that passed gate 6) and three swaps
  from the redeployed app in the same test vault. SOL→USDC paid 3,554 USDC
  base units on the output; USDC into an SPL token (two hops) paid 3,000
  USDC base units on the input, the first live input-side fee; that token
  back to SOL paid 25,281 lamports, the first fee in SOL. Each is exactly
  ⌊amount × 30 / 10,000⌋ on the side the signed mints select.

All 18, with links, times and figures, are on the [Evidence](evidence.md)
page, together with the commands that count them.

**Next.** The funded plan in Section 7.1, in order. Before it, on
28 September: gate 6, the fee-side upgrade and devnet's own program id.
Three internal review requests are written and not yet held: the
program-id change with the mainnet upgrade, the network-fee wallet design
(milestone 1) and the native-staking design (milestone 3).

## 6. Who it is useful for

### 6.1 People and organisations

* **Long-term holders** who want savings off a key that is exposed on
  chain, without giving up the ability to swap and rebalance. They can hold
  SOL, USDC or any compatible SPL or Token-2022 token, swap between them
  through Jupiter without the funds ever sitting in an Ed25519-controlled
  account, withdraw to any wallet, and restore from 24 words.
* **Treasuries**: DAOs, foundations and protocols hold assets for longer
  than current estimates of the time to a cryptographically relevant
  quantum computer, and they need to rebalance and pay. A treasury vault
  does both; the project's own treasury is one. Others should wait for the
  audit and the beta.
* **Anyone who cannot audit a wallet.** The app shows the fee as its own
  line and the net minimum before signing, and it refuses to sign against
  a program whose hash differs from the published one.

### 6.2 Honest context on alternatives

Quantum-resistant custody on Solana exists as developer tools, a
launchpad treasury and early betas (checked 5 October 2026; on-chain reads
and links on the [Evidence](evidence.md) page):

* **Winterwallet** (Blueshift;
  [code](https://github.com/blueshift-gg/winterwallet/commit/672fc6789b1532ee680f24842d235e0be8737b61);
  program
  [`winter5vMwvf51xrSVPTxbAAD6qiSTmPeRTSizMCQCa`](https://explorer.solana.com/address/winter5vMwvf51xrSVPTxbAAD6qiSTmPeRTSizMCQCa))
  is a command-line tool and SDK, on mainnet since 27 April 2026, with its
  public code unchanged since release day. Its limit of 16 accounts per
  inner instruction
  ([`MAX_CPI_INSTRUCTION_ACCOUNTS`](https://github.com/blueshift-gg/winterwallet/blob/672fc6789b1532ee680f24842d235e0be8737b61/common/src/lib.rs))
  cannot carry a Jupiter route.
* **ORE's Falcon wallet**
  ([ore-cli](https://github.com/regolith-labs/ore-cli), on Blueshift's
  [Vector](https://github.com/blueshift-gg/vector) with
  [Falcon-512](https://github.com/blueshift-gg/solana-falcon512)) is a
  command-line wallet on mainnet that transfers and stakes ORE; it has no
  swap or settlement check.
* **[winternitz.io](https://winternitz.io)** is a token launchpad that
  keeps its own treasury in a Winternitz vault (program
  [`13EtnfYGUH8NaGAnUpDTVgSsXoewNnULp7ESwHzQUANT`](https://explorer.solana.com/address/13EtnfYGUH8NaGAnUpDTVgSsXoewNnULp7ESwHzQUANT)).
  Its users trade on Meteora with ordinary wallets, and a quantum-safe
  wallet with in-browser signing is the first item on its roadmap.
* **[quantumsol.tech](https://quantumsol.tech)** has a browser app on
  mainnet that deposits into and withdraws from a hash-signature vault.
* **[QShield](https://github.com/sol-9/QShield)** (a devnet research
  preview) and **[qsol.day](https://qsol.day)** (a devnet beta) are browser
  wallets that do not yet run on mainnet.
* **Qubit's quantum vault**
  ([code](https://github.com/QuantiumResistantInfra/quantum-vault/commit/47e06d5af6fc2c011bdc193eda72be23c9ff6184))
  was deployed on mainnet on 31 May 2026 and its program was closed on
  3 June 2026.
* **Squads and mainstream wallets** change who holds the keys, not the
  Ed25519 signature underneath.
* **Native Falcon accounts** would be the network-level answer. By the
  Foundation's own account they are not imminent. If they arrive, the
  vault's lasting value is the part they do not provide: an allow-listed
  executor that settles a trade on chain without the funds leaving
  quantum-safe control.

To our knowledge, as of 5 October 2026, no other quantum-resistant vault
on Solana trades without the funds leaving it. Here the vault signs the
route itself, and the program checks on chain that the signed minimum
arrived, so on every vault account that exists today a compromised router
or relayer can at worst deliver that minimum (two narrow gaps found by our
own testing on 5 October, worth nothing on today's accounts, are in the
audit scope; Section 10). Where others are ahead: Winterwallet came first,
simpler designs have a smaller audit surface than an allow-listed CPI
executor, ORE's wallet already stakes ORE, and none of these projects,
ours included, is audited yet. That is why the audit
comes first.

## 7. Roadmap

### 7.1 Funded milestones

Nine milestones, USD 111,275 in total on the convertible track, each paid
on completion and each deliverable checkable from outside. Milestones 1
to 7 are product development (USD 89,625); 8 and 9 are adoption
(USD 21,650). The amount is mostly two external reviews, a Solana program
audit and a scoped cryptographic review (USD 60,000, paid through to the
reviewers; estimates until we hold written quotes). The rest is builder
pay for defined deliverables (USD 42,375), a USD 5,000 bug bounty pool paid
only for valid findings, a lawyer's review of the terms of use
(USD 3,000) and a year of infrastructure (USD 900).

| # | USD | Deliverable |
|---:|---:|---|
| 1 | 3,750 | **Reliable network fees.** One network-fee wallet derived from the 24 recovery words; one approval funds the vault and its fee reserve; reserve view, low-balance warning, top-up and a sweep of older per-device fee wallets. Delivered with a design review of the derivation, Rust-to-TypeScript test vectors, end-to-end tests, a devnet run and published mainnet transactions. |
| 2 | 3,000 | **Venue compatibility for program-owned accounts.** A published open matrix of the venues Jupiter routes through, each marked working, failing or untested when the trader is a program-owned account, with replay evidence on the venues' real programs and the reason for each failure; failing venues removed from routing; the replay harness published. |
| 3 | 9,000 | **Staking from the vault.** A reviewed program upgrade (the vault stays staker and withdrawer, no lockup, stake returns only to the vault, staking never rides with a swap); stake, unstake (whole or part) and withdraw in the app; a default validator set chosen by published rules that exclude the superminority and cap commission at 5%; liquid staking tokens (JitoSOL, JupSOL, mSOL, INF) through the settlement-checked swap once one swap per token has landed on mainnet; tests against the real Stake program, a devnet rehearsal, and a mainnet delegation of at least 1 SOL with deactivation and withdrawal, signatures published. |
| 4 | 30,000 | **Review engagements signed.** Letters countersigned with a Solana security firm (program audit, including staking and any fee change made before the code freeze) and a cryptography reviewer (verifier, key and fee-wallet derivation, deterministic salts, one-time-key lifecycle), each with a written scope, a code-freeze date and one fix-review round; deposit invoices issued. This tranche pays the deposits. |
| 5 | 6,750 | **Audit-ready open release.** Program, SDKs, settlement-check library and test vectors published under MIT with every review record (the app stays closed; its build hashes are published); 0.x packages marked unaudited; the frozen commit on mainnet from a reproducible build checkable with [`solana-verify`](https://github.com/solana-foundation/solana-verifiable-build); upgrade authority on a Squads 2-of-3; audit package delivered. |
| 6 | 31,500 | **Review reports delivered.** Both reports delivered and shared in confidence with the Foundation with a fix plan; the SHA-256 of each report published on delivery. This tranche pays both reviewers' balances (USD 30,000) and USD 1,500 of builder pay; the full reports are published at milestone 7. |
| 7 | 5,625 | **Findings fixed and reports published.** Every finding fixed, or accepted with a published reason, and re-checked; the fixed program upgraded through the multisig, verifiable; reports published, matching the hashes; 1.0 packages. |
| 8 | 12,650 | **Capped public beta.** Own domain; a per-vault deposit cap enforced by the app; lawyer-reviewed terms; a USD 5,000 bug bounty; published monitoring; an incident runbook; guides for moving savings in, restoring and staking. |
| 9 | 9,000 | **100 vaults** within 90 days of the beta, each holding at least 0.1 SOL-equivalent and having made at least one swap or stake, with a public monthly report and its open script. |

Done before this application, on 28 September: gate 6, the fee-side
upgrade and devnet's own program id.

### 7.2 After the grant

Third-party integrations and setting the upgrade authority to none come
after these milestones. Staking is inside them (milestone 3); a direct
stake-pool deposit was examined and dropped (Section 7.3), and other
yield programs are not planned before the code freeze. Each remaining item
needs a program change, a follow-up audit or other teams' decisions, so
none is promised on a date. The burn comes last: after the bounty period,
and only once the allow-list is complete. From then on no one, including
us, can change the vault's rules; the dependencies that remain are listed
in Section 10.

### 7.3 Staking from inside the vault

The vault PDA is program-owned and carries data. It can credit any account
directly with `LAMPORT_MOVE`, and it can sign as an authority through the
CPI. What it cannot be is the funding account of a System transfer. That
one fact shapes every staking path.

| Path | What the program needs | Status on 5 October 2026 |
|---|---|---|
| **Liquid staking tokens** (JitoSOL, JupSOL, mSOL, INF) through the existing swap | Nothing in the program. In the app: a separate venue set used only for these four (venues replayed through the vault in both directions), shared-account routes, the token list and a value display | Tested on real programs (pinned replay, one snapshot): all 37 runnable routes of the first sweep, and two later ones, landed through the mainnet program in a local replay, across 11 venue labels. None has been swapped on mainnet; the app release (milestone 3) waits for one mainnet swap per token |
| **Native delegation** (the vault is staker and withdrawer) | New allow-list entries and rules; no new header flag and no digest change. Stake `InitializeChecked`, `DelegateStake`, `Deactivate`, `Withdraw`, and `Split` and `Merge` for partial unstake; System `AllocateWithSeed`; SPL Token `UnwrapLamports`, which moves SOL from the vault's wrapped-SOL account into the new stake account, so one Execute with one leaf funds, creates, initialises and delegates. Invariants: after any Execute every stake account the program touched has the vault as both staker and withdrawer with no lockup; stake returns only to the vault or its token account; a staking Execute carries no swap and no lamport move, and a swap references no stake account the vault controls | Proposed. Two throwaway prototypes, tested locally against mainnet's own Stake v5.1.0 and p-token v1.0.0 binaries. The final-rules prototype is 74,128 bytes (mainnet's ProgramData holds 74,352), ran the full lifecycle and refused all 37 rule-breaking payloads before signature verification. Not yet through the internal review gate. Shipping it is milestone 3's reviewed program upgrade, before the freeze |
| **Deposit into an SPL stake pool** | An allow-list entry for the pool's deposit instruction, a settlement rule, and a second signing address | Dropped: Jupiter's Sanctum route already mints at the pool rate, and for the large tokens a market route is cheaper; no referral income; the pool debits the depositor with a System transfer from a signer that carries no data, which neither the vault nor a wrapped-SOL account can be |

Staking has to land before the upgrade authority is set to none, because
after that no allow-list entry can be added. Native staking pays no
protocol fee; liquid-staking swaps pay the 30 bps fee each way. A stake
account needs 1.00166624 SOL: the 1 SOL minimum delegation in force since
epoch 989
([SIMD-0490](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0490-upgrade-stake-to-v5.md),
read back by simulation on 4 October), plus rent. Any deposit cap must
therefore be at least 1.01 SOL-equivalent.

**What the vault protects and what it does not.** A liquid staking token
held in the vault is protected from key theft like any other token in it,
but the stake pool behind it is not: its program, its operators' keys and
its program's upgrade key sit outside the vault. Native stake is the one
form in which every party that can move the SOL is the vault or Solana
itself: the stake account's only withdraw authority is the vault, and the
[Stake program](https://github.com/solana-program/stake) has no upgrade
key (it changes only through feature-gated upgrades the network adopts).
The validator decides how much is earned, not where the SOL goes.
Residual risk: a future Stake or token release that changes an
instruction form could strand stake under a frozen vault (stranded, not
stolen); the app pins both programs' hashes and switches staking off if
either changes.

**Validators.** An open list, and a default set of two or three chosen by
published on-chain rules: not delinquent, credits in each of the last four
full epochs, commission at most 5%, outside the superminority (the 18
largest validators, together a third of stake). At epoch 1049
(4–5 October), 532 validators pass. The pick among them is lowest commission, then a
rotation by a hash of the vote key and the epoch, so no payment buys a
place.

### 7.4 What the primitive opens beyond that

The general idea is **an account with no private key, whose authority is a
quantum-safe signature, that can carry out a restricted set of actions and
verify their outcome on chain.** A swap came first because it is the
hardest and the most useful. These are directions, not commitments; each
needs an allow-list entry, and therefore a program change, before the
upgrade authority is set to none.

| Direction | What exists | What it would take |
|---|---|---|
| Treasuries and DAO reserves | the product as it is | onboarding and a deposit-cap policy |
| Bounded payments with a hard budget | `EXECUTOR` names a helper, `BOUNDED` lets it pick the route, every signature expires | a keeper that lands pre-signed, bounded authorisations when due; the user's device signs each one, since only the user can sign |
| Lending deposits | nothing yet; not planned before the freeze | per-protocol allow-list entries with a receipt-token settlement rule |
| k-of-n approvals, or a Squads adapter through an EXACT CPI | the EXACT digest | design work |
| A cheaper or different scheme | the `scheme` byte | a Keccak-permutation syscall ([SIMD-0579](https://github.com/solana-foundation/solana-improvement-documents/pull/579), proposed July 2026 and still open on 5 October; it proposes 70 or 130 CU per permutation) or a Falcon verifier would cut cost; moving balances into native post-quantum accounts later would be new design, not an existing feature |
| The same vault on EVM rollups | Keccak-256 is a native EVM opcode | a separate contract reusing the design, not the code |
| Private settlement | the vault hides nothing today; confidential mints are rejected | a zero-knowledge proof of "at least Y received"; research |

## 8. What the ecosystem gets regardless

The program, the Rust and TypeScript SDKs, the settlement-check library
and all test vectors are MIT-licensed and are published at the open
release (milestone 5) with every review record; the audit reports follow
at milestone 7. The web app stays closed; the code that derives and uses
the signing keys lives in the open SDK, and the app's build hashes are
published. Four parts are useful to teams that never touch the product:

* **A settlement-check library** for any program that lets an external
  router move its funds: delta, owner, mint, delegate, close authority,
  "no other account of ours", mints re-checked after the CPI, and native
  accounting. It is independent of the signature scheme.
* **The authorisation digest, with cross-language test vectors**, so other
  hash-based signers on Solana can interoperate rather than each inventing
  its own.
* **A byte-exact Transaction v1 builder and estimator**, checked against
  the serialiser and wire order the RPC decodes.
* **The record.** A study of the quantum-resistant vaults on Solana we could find at
  pinned commits, with a copied-versus-new table; the 69 findings, with
  reproductions and fixes, as a public record of how stateful signers and
  CPI-settled custody fail; and replay evidence of which venues accept a
  program-owned trader (the open matrix of milestone 2). Measured compute
  and byte budgets for hash-based signatures under v1 are also data for
  any SIMD discussion of post-quantum verification.

## 9. Open source and transparency

Open source: partially. At the open release (milestone 5) the program,
the Rust and TypeScript SDKs, the settlement-check library and all test
vectors are published under MIT, with every review record, a reproducible
build checkable with `solana-verify`, and the upgrade authority on a
Squads 2-of-3. The web app stays closed; its build hashes are published,
and before every signature it recomputes the deployed program's hash
from the chain and refuses to sign on a mismatch. The repository is
private until then and is public before any public beta takes deposits.
The audit reports are shared in confidence with the Foundation at
milestone 6 (their SHA-256 published on delivery) and published at
milestone 7.

Until then the rules are the ones we will be judged by: every number has a
source or a test, every reused component is attributed with its licence,
every finding is recorded with its status, and a milestone is not called
done without the tests that gate it. What can be checked today, without
the repository, is on the [Evidence](evidence.md) page.

## 10. Risks and open problems

* **The signature library is weeks old and unaudited.** A defect in it is
  a total break. We pinned the version we reviewed line by line and
  reproduced its conformance vectors. That is not an audit; milestones 4
  and 6 buy one (the cryptographic review).
* **Keccak-256 for SHA3-256, and deterministic salts,** are argued, not
  proved: a heuristic transfer and a joint-PRF hybrid. Keccak-256 and
  SHA3-256 ([FIPS 202](https://csrc.nist.gov/pubs/fips/202/final)) share
  the permutation and differ in padding: SHA3 appends domain-separation
  bits. The cryptographic review is asked to confirm or reject both
  arguments.
* **Two devices at once can reuse a key.** Leaf choice is randomised and
  the marker design bounds the damage, but only one device at a time is
  safe, and the app says so.
* **Two small gaps in today's settlement check,** found by us on
  5 October while designing staking. Both concern how lamports are
  measured in a swap's token accounts, and both are zero for every vault
  account that exists today, because every one was created at the current
  rent rate; they would grow only if rent per byte falls. A hostile mock
  venue reproduces both locally; the final staking prototype refuses both.
  The fix is part of the milestone 3 upgrade and the audit's scope.
* **Until the authority is burned, the vault is as safe as whoever holds
  it:** today one deployer key, then (milestone 5) a Squads 2-of-3, both
  Ed25519. Validator keys and the programs the vault calls (Jupiter, the
  token programs, the Stake program) remain classical.
* **Jupiter is a dependency we do not control.** Before the burn, a new
  Jupiter instruction format would stop swaps (not holding or
  withdrawing) until we ship and land an upgrade. After the burn, nothing
  can be patched or added: a new Jupiter format or a new Token-2022
  extension would need a new program. The extension policy is a
  blacklist, and the audit should weigh an allow-list.
* **Staking dependencies.** Liquid staking tokens depend on pools whose
  programs and keys sit outside the vault (Section 7.3). The Stake program
  and the token program change only by feature gates, but in 2026 the
  Stake program has been replaced twice and the token program once; after
  the burn, an incompatible release could strand vault stake until a form
  the frozen rules accept exists.
* **Cluster separation rests on distinct program ids.** Key derivation
  and the digest separate deployments by program id. Until 28 September
  our devnet test deployment reused the mainnet id, so between those two
  clusters the separation did not hold. No mainnet vault ever existed on
  devnet (checked 28 September). Devnet now runs its own id and its copy
  of the mainnet id is closed (transactions on the Evidence page); binding
  the cluster itself, not only the program id, is left for a future
  version.
* **One open app finding.** A recovery that lands after the app is locked
  still shows as sent, so the next unlock asks for one extra key refresh.
  It costs a refresh, not funds, and it waits for a design review.
* **No privacy.** The address is stable for life, and every swap's route
  and bounds are public.
* **Not every venue serves a program-owned user,** which excludes some
  liquidity (Section 4.5).
* **The quantum timeline is uncertain in both directions,** and native
  post-quantum accounts could narrow the standalone vault's edge to its
  executor and its head start.

## Appendix: where to check each claim

Everything below is public today. The repository itself (source, tests,
saved RPC responses and the full review log) is published at the open
release, milestone 5.

| Claim | Where to check it |
|---|---|
| The program, its hash and its upgrade history | [Evidence](evidence.md): program and accounts, hash recipe, deploy, extension and upgrade transactions |
| Three vaults, all ours; 18 program transactions | [Evidence](evidence.md): the two RPC queries and their results |
| The canary swap, the fee, the rejected swap, the recovery | [Evidence](evidence.md): mainnet transactions and fee accounting |
| The live-app swaps of 27 and 28 September | [Evidence](evidence.md): mainnet transactions and fee accounting |
| Devnet lifecycle, the wire-order rejection, the devnet program id | [Evidence](evidence.md): devnet section |
| Bytes and compute units, test counts | [Evidence](evidence.md): measured budgets and tests |
| The app's build and its program-hash check | [Evidence](evidence.md): app build record |
| Staking facts (minimum delegation, Stake and token programs, validators, replays) | [Evidence](evidence.md): staking section |
| Competitors | [Evidence](evidence.md): competitor checks |
| Reused code and licences | [Evidence](evidence.md): prior work we build on |
| Review findings | [Evidence](evidence.md): review record summary; the full log at milestone 5 |
| Transaction v1, SIMDs, the scheme, dependencies | the links in Sections 1, 2 and 7 |
