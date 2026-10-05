# Evidence

*Current as of 5 October 2026. This page carries the facts the
[technical brief](README.md) relies on, in a form anyone can check
without the project's repository: on-chain accounts and transactions with
public explorer links, the commands that reproduce each count, and, for
every measured figure, what produced it. Mainnet links open Solana
Explorer; devnet links add `?cluster=devnet`. The full evidence files
(every saved RPC response), the tests and the complete review record are
published at the open release, milestone 5 of the funded plan.*

## 1. Reading the explorer pages

* Transactions from the vault are Transaction v1. Solana Explorer shows
  them as version **V1**, with their size against the 4,096-byte maximum.
* The vault program has no published IDL, so Explorer shows it as
  "Unknown Program (BsKbo3sh…): Unknown Instruction". Use the **Tokens**
  tab (the treasury's fee and the vault's credit appear as balance
  changes) and the **Logs** tab.
* The protocol fee is not a separate transaction. It is a token transfer
  inside each swap, visible as the treasury token account's balance
  change in that swap.
* Explorer numbers instructions from 1; the RPC's `InstructionError`
  counts from 0. The rejected swap (M6) shows "Instruction #4 Failed"
  with `custom program error: 0xe`, which is the RPC's
  `InstructionError [3, Custom 14]`. Error 14 is `SettleFailed`
  (Section 11). In that transaction Jupiter's own call succeeds; the vault
  program is what fails.

## 2. Program and accounts

| Role | Address | Notes |
|---|---|---|
| Vault program (mainnet) | [`BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo`](https://explorer.solana.com/address/BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo) | upgradeable; upgrade authority a single deployer key until milestone 5 |
| Its ProgramData | [`DxHi9JBoD3gU5pZej3yYB2goJLh7nuaspZ5JJcDhorgz`](https://explorer.solana.com/address/DxHi9JBoD3gU5pZej3yYB2goJLh7nuaspZ5JJcDhorgz) | 74,397 bytes: a 45-byte header and a 74,352-byte program region; last deployed at slot 451,338,716 |
| Treasury vault (receives the fee) | [`3VXKr5pcfq8D7nfphRHAt6z8oS7RZHCcNRLrqyxTzo1D`](https://explorer.solana.com/address/3VXKr5pcfq8D7nfphRHAt6z8oS7RZHCcNRLrqyxTzo1D) | a vault of this program; 128 bytes, owner = program |
| Treasury USDC account | [`8nW1SfsL58X6NDV2djNaGjR5zGChUEx9AinEatYE8752`](https://explorer.solana.com/address/8nW1SfsL58X6NDV2djNaGjR5zGChUEx9AinEatYE8752) | fees in USDC (M8, M12, M17, M18) |
| Treasury wrapped-SOL account | [`8Hu9SR7m3hMahYfnvu9CjBNjbGgbofaB112C9fBx5YJ4`](https://explorer.solana.com/address/8Hu9SR7m3hMahYfnvu9CjBNjbGgbofaB112C9fBx5YJ4) | created in M19; fees in SOL |
| Treasury Token-2022 account | [`BzV8uqLfzBX5giMs2HAsieErA58UWAa4ppDZV2a8iqhf`](https://explorer.solana.com/address/BzV8uqLfzBX5giMs2HAsieErA58UWAa4ppDZV2a8iqhf) | fee of M11 |
| Canary vault | [`HGTkMLVxZjcGo3reUhBAVy2DYuyJA2Vd8Kr1NM5zg1SK`](https://explorer.solana.com/address/HGTkMLVxZjcGo3reUhBAVy2DYuyJA2Vd8Kr1NM5zg1SK) | the 24 September canary (M3 to M8) |
| Live-app test vault | `AJo49aZkuGP53fLrT1mYtQ8eRYPFqtBfJZEEJ2yrzp8C` | our third vault; its transactions are M9 to M13 and M16 to M19 below |
| Transaction v1 feature gate | [`txv1aq4pp281K9um3tnPgkfX8UqtFT6wcVW3hNezGLL`](https://explorer.solana.com/address/txv1aq4pp281K9um3tnPgkfX8UqtFT6wcVW3hNezGLL) | activated on mainnet at slot 447,120,000, the first slot of epoch 1035 (15 September 2026, 01:04 UTC); on devnet at slot 492,480,000 |
| Jupiter v6 program | [`JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4`](https://explorer.solana.com/address/JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4) | the only router the vault may call |
| Vault program (devnet) | [`48DpFqZJP6tp9hszZzVyW2hwjyN7JR9y3WyptaPtVfDn`](https://explorer.solana.com/address/48DpFqZJP6tp9hszZzVyW2hwjyN7JR9y3WyptaPtVfDn?cluster=devnet) | devnet's own id since 28 September; ProgramData `4cgpcsH9QUmHnTAdAUTRREu1qRoEwnNEUc7kQPFjHhd7`; same program hash as mainnet; last deployed at slot 505,140,142 |

## 3. Checking the deployed program

The program now on mainnet is the fee-side build of 28 September: 65,504
bytes, sha256

`337fe41baebc826919ccde94d8d33b60002c3551af4815ac75e2380e0eb00f0a`

The ProgramData program region is 74,352 bytes, because a 10,240-byte
extension (M14) made room before the upgrade. The program is the first
65,504 bytes and the remaining 8,848 bytes are zero. Tools that report
the ProgramData length, including the Explorer program page, show the
larger figure.

With the Solana CLI:

```
solana program dump BsKbo3shF4X5nG3sMcBApDGcDM7ocC7Gun4ycBqsfrRo prog.so --url mainnet-beta
head -c 65504 prog.so | sha256sum                  # 337fe41b…0f0a
tail -c +65505 prog.so | tr -d '\000' | wc -c      # 0: the tail is all zero
```

Without the CLI, from the public RPC (the account data starts with the
45-byte ProgramData header):

```
curl -s https://api.mainnet-beta.solana.com -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["DxHi9JBoD3gU5pZej3yYB2goJLh7nuaspZ5JJcDhorgz",{"encoding":"base64"}]}' \
  | jq -r '.result.value.data[0]' | base64 -d | tail -c +46 | head -c 65504 | sha256sum
```

We ran the second recipe on 5 October 2026: the account was 74,397 bytes,
the hash matched, and the tail held no non-zero byte. The devnet copy
(`4cgpcsH9…`) gives the same hash. The 64,112-byte build first deployed on
24 September (sha256 `82b4330d74c1158015352b35d40d6eaa8a82a2377f33f39742d0fb61348e2ea3`)
is no longer on mainnet. The app runs the same check in the browser
before every signature. A reproducible build checkable with `solana-verify` is part of milestone 5.

## 4. Counting vaults and program transactions

Both reads are public RPC calls, made on 5 October 2026 (slot 453,515,171):

* `getProgramAccounts` on the program with the filter `dataSize: 128`
  (every vault account is 128 bytes) returns exactly three accounts: the
  treasury, the canary vault and the live-app test vault (Section 2). All
  three are ours.
* `getSignaturesForAddress` on the program returns 18 signatures: M1 to
  M3 and M5 to M19 below (M4 is a plain System transfer into the canary
  vault, so it does not reference the program). 17 succeeded; M6 landed
  as a failure by design. The latest is M19, 28 September 15:52 UTC; none
  has landed since.

## 5. Mainnet transactions

All 19 were confirmed `finalized` on the public mainnet RPC on 5 October
2026. Bytes and compute units (CU) are the landed transaction's wire size
and `computeUnitsConsumed`; a dash means the figure was not recorded.

| # | Time (UTC) | What it shows | Bytes | CU | Link |
|---|---|---|---:|---:|---|
| M1 | 24 Sep 21:23:26 | Program deploy (the 64,112-byte build) | | | [4iSCbijs…](https://explorer.solana.com/tx/4iSCbijsfmGX8gvuUkPwTKSR8mDZY3zwZvtZ7qYh2nap9d3cKYW5DD5qs6AC4uUrXqN5gaga5kg8AyTCwne5QN99) |
| M2 | 24 Sep 21:39:45 | Treasury vault initialised | — | — | [5kJoNoyX…](https://explorer.solana.com/tx/5kJoNoyXUWfJ5K1Haw949qTc7ae1NQEHhGj71hMkR8yF7ABxu4FFZvuKBmwKZuCZ2YfLZcLd7v22kVU1pLfqGTcL) |
| M3 | 24 Sep 21:41:10 | Canary vault initialised | 300 | 1,811 | [5NdfyRMi…](https://explorer.solana.com/tx/5NdfyRMingeRpqvdENwy3SasqJYFAbKNxf5ewRywtwAzQmnZVmbs4gkxgTJoPWg5KuwFn4c5FdTg4Cs8RZxv85av) |
| M4 | 24 Sep 21:41:13 | Deposit of 0.01 SOL (10,000,000 lamports) into the canary vault; a System transfer, no vault signature needed | 236 | 150 | [5MKbB4eb…](https://explorer.solana.com/tx/5MKbB4ebQEngnT92LrdTRY2WePQ7V8y3EfjAudj7vKVb7KSLdvzw1JSpEY8A8geLdyoxXZxSUePMPGBA6CVxRta) |
| M5 | 24 Sep 21:41:16 | Wrap of the deposited SOL by Execute | 1,459 | 55,694 | [2HT5DsPE…](https://explorer.solana.com/tx/2HT5DsPEv91CmR2WqctQapBwpgYzK6WtgfSyDGsUWGSEvAc3KrFkACVbACtq3jFTFs11sXU242Ytw5px2jmf3AMB) |
| M6 | 24 Sep 21:41:18 | Swap signed with `min_out` at twice the quote (quote 1,170,570; `min_out` 2,341,140): **rejected** by the vault's settlement check, `InstructionError [3, Custom 14]` | 2,163 | 127,343 | [uVygn8g4…](https://explorer.solana.com/tx/uVygn8g4t3wZPPGcSKm8i5f4kjDceLBb4qY8EnWDSF4R3meiBk1DvsADYVM3moH3FjyxuAmKDXE23S5fdEiV3tY) |
| M7 | 24 Sep 21:41:20 | Recovery after the failed attempt (the one-time-key recovery message) | 1,313 | 38,847 | [4AR1kzmV…](https://explorer.solana.com/tx/4AR1kzmVvFVociCTSDJYgxuwuWKjgJNV5T6QMzBmbatyK7KsM2nDhdQK8coZsmQfpGZohYcPFrkENLhWgKHq6cwS) |
| M8 | 24 Sep 21:41:24 | Canary swap, 0.01 SOL to USDC through Meteora DLMM, fee 3,511 USDC units to the treasury (Section 6) | 2,163 | 129,328 | [wggJur7C…](https://explorer.solana.com/tx/wggJur7CkEwp7J3L5xawJqP1u2nhQ4wYSA8FXqvitS3boaGFsvPApQqewtZr5Xsipne4Gzj4HJCiu3dT6KbjiYM) |
| M9 | 27 Sep 10:18:30 | Live-app test vault initialised (our third vault) | — | 2,935 | [5E88Dk6D…](https://explorer.solana.com/tx/5E88Dk6DsMKoKMbEUDdMJLXNN2cQMxhHdGRsBPcrXvMMu5kYvxwHMabXWUr3FsWxSVUM4KSoWR5Ft1SoWbqT2p8A) |
| M10 | 27 Sep 10:19:05 | Wrap of 198,699,520 lamports in the app | — | 58,691 | [34sh3BRP…](https://explorer.solana.com/tx/34sh3BRPcXo7jiD7vixbLVuKeAwmVBdsV722sHojQTbTZGb1szjyiHsFmnxrBVT2tJi8SvEzF7ZnD7WwKNeENHQP) |
| M11 | 27 Sep 10:21:47 | Swap of 0.1 SOL into a Token-2022 token, fee on the output (Section 6) | — | 148,686 | [3q5oAKmB…](https://explorer.solana.com/tx/3q5oAKmBEkseE3fzWiCRrcTu2qCBDU38y1zCLwDXmwcjYXVdxwVCFYwUTHjZUC4qtt1AY1CaqtjkhjfJBN5XkTEJ) |
| M12 | 27 Sep 10:22:47 | Swap of all of that token into USDC, fee on the output | — | 131,733 | [5Y4tw5A5…](https://explorer.solana.com/tx/5Y4tw5A553xCWuVKbUeEhGAUzv5THEUjrjpALzGAyF37N31WdsM1cKg49uW6dqrheBP6V3kRFM7zH7LpjZfFcwV8) |
| M13 | 27 Sep 12:33:40 | A key operation in the app (one vault instruction, no balance change) | — | 38,845 | [3k1tf14D…](https://explorer.solana.com/tx/3k1tf14D8jzaJMroehy7Uk9E6YeFaSpzZEXRU294ASzHEcwUZh1MoonK8z3ioVETgJC2fETHoB1v3uiK2duhQt4H) |
| M14 | 28 Sep 13:53:03 | ProgramData extended by 10,240 bytes (to 74,352) | | | [9RjzDBjL…](https://explorer.solana.com/tx/9RjzDBjLdAjDvmhNjH3NUUVaR6S7hsGx6R9ovdk47qZn72n9dz7XjTTtDZsTp82oxiq1pZZchWuazzFseoyRpEs) |
| M15 | 28 Sep 13:56:07 | **Upgrade** to the 65,504-byte fee-side build; the bytes read back hash to `337fe41b…0f0a` (Section 3) | | | [5dxbnYXk…](https://explorer.solana.com/tx/5dxbnYXkX9mhzJLnZtqcNq869WtoSvpTMS2WN7Y758dZ8eWesHP4UqH26Q6hr6iE5Ju6onKjT55XK3y15et3UmFr) |
| M16 | 28 Sep 15:50:43 | A vault instruction from the test vault with no balance change; by its compute units a key operation (not decoded) | — | 38,844 | [2DVrZ6Bx…](https://explorer.solana.com/tx/2DVrZ6BxUTJQp5zQgwccunymekQGkag1LUA2cJFoAmV33fA5x93pxDAucNMuPwRfW3KcrdHZiRN3Zj6P4q9Jv7s3) |
| M17 | 28 Sep 15:51:26 | Swap 0.01 SOL to USDC on the upgraded program, fee on the output | 2,159 | 127,782 | [4hQr1Yds…](https://explorer.solana.com/tx/4hQr1YsgpaPhgEJFVJfGSpd4h2TaLcFWoGLiHYyeg1UxjKoYWXaWyk9LMsHqjjbhUjsDk45KrDAiDj8xnQn3RAj4) |
| M18 | 28 Sep 15:51:45 | Swap 1 USDC to an SPL token (two hops), fee on the **input**: the first live input-side fee | 2,608 | 170,490 | [kZWWV9nu…](https://explorer.solana.com/tx/kZWWV9nu13pouV6pc7MyQW519KQTkP6zxyVkuzaoM16GTLqXVkPdwXk4Nq1e143xv5yWHytcKrsM8Rcn26cABAT) |
| M19 | 28 Sep 15:52:45 | Swap of that token back to SOL, fee paid **in SOL**: the first live fee in SOL | 2,130 | 128,509 | [2QVqNaLb…](https://explorer.solana.com/tx/2QVqNaLbZSdwCE4orhGDSrDwxkUrTCNWEuu8BMemJqRMLAN4Sa5b139erDSHfdsYZGY5dhT4a1AWNrR1qtrC6zKR) |

The slots are 450,149,241 (M1) to 451,364,763 (M19); each is on the
transaction's Explorer page.

## 6. Fee accounting per swap

The fee is ⌊amount × 30 / 10,000⌋ of the gross output, or of the input
actually spent when the input side pays. Which side pays follows the two
mints the user signed: USDC or USDT when either side has one, otherwise
SOL when either side has it, otherwise the output. The 24 and 27
September swaps ran on the first build, which always took the fee from
the output. Amounts are in each token's base units.

| Swap | Fee side and token | Basis | Fee | Vault received | Treasury account |
|---|---|---:|---:|---:|---|
| M8: 0.01 SOL → USDC | output, USDC | gross 1,170,571 | 3,511 | 1,167,060 (signed `min_out` 1,164,719) | `8nW1Sf…` |
| M11: 0.1 SOL → Token-2022 token ([mint](https://explorer.solana.com/address/CbcyNo7m1amFWqEQm2m4PLv1UNvpcL3C1Ujm6AkzpKoU)) | output, that token | gross 2,450,194,050 | 7,350,582 | 2,442,843,468 | `BzV8uq…` |
| M12: that token → USDC | output, USDC | gross 12,055,998 | 36,167 | 12,019,831 | `8nW1Sf…` (3,511 → 39,678) |
| M17: 0.01 SOL → USDC | output, USDC | gross 1,184,947 | 3,554 | 1,181,393 | `8nW1Sf…` |
| M18: 1 USDC → SPL token ([mint](https://explorer.solana.com/address/DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263)), 2 hops | input, USDC | spent 1,000,000 | 3,000 | 29,142,129,190 (the whole output; the input fell by 1,003,000) | `8nW1Sf…` |
| M19: that token → SOL | output, SOL (wrapped) | gross 8,427,195 lamports | 25,281 | 8,401,914 | `8Hu9SR…` (created in this transaction) |

Each fee equals the rule exactly: for example 1,170,571 × 30 / 10,000 =
3,511.7, floored to 3,511.

## 7. Devnet

Jupiter has no devnet deployment, so no swap, fee or settlement path runs
on devnet; devnet carries the vault lifecycle and the program's
deployment history. All signatures below were confirmed `finalized` on
the public devnet RPC on 5 October 2026.

### 7.1 Deployments, the program-id move and the upgrade rehearsal

Until 28 September devnet ran a copy of the mainnet program id. On that
day devnet moved to its own id and the copy was closed (Section 10 of the
brief explains why it matters).

| Date | Event | Slot | Link |
|---|---|---:|---|
| 19 Sep | First devnet deploy (the shared id, now closed) | 500,534,373 | [3NLfjr48…](https://explorer.solana.com/tx/3NLfjr4824K2awpmwvnSuMcPU4ssL69SX8C7TQ1nVHtv6tdxANdNJaZ8Whg2Ps7RjtRiPjtNZ5gtp55kbPzhA75y?cluster=devnet) |
| 24 Sep | ProgramData extension (shared id) | 503,597,313 | [4Sev1A9y…](https://explorer.solana.com/tx/4Sev1A9yq3xey6VgG6DQcbVzft3BRMtwUGav5iJojJjB2aUA8HCfyx6MpZVq3MY42JxhSZ1Tf1QyU6LH1XSuahuP?cluster=devnet) |
| 24 Sep | Upgrade through the CLI's default QUIC transport, after five `--use-rpc` attempts failed (shared id) | 503,612,093 | [2oCdKnJd…](https://explorer.solana.com/tx/2oCdKnJdmuYjFkEJgzs38J8YozrXgDhrbz4HDJYzPAEm62thoUfCmzM2f5wAo3CBEyQAJPjEqKmUhSpZ8794iv5L?cluster=devnet) |
| 28 Sep | Deploy of the 64,112-byte build under devnet's own id, first attempt | 505,136,344 | [do4j7tdU…](https://explorer.solana.com/tx/do4j7tdUEvZdjquZgU5NXBMdynpr3GiNij1zZ9VfepgjQHAQmbjnrENeirVSmn1rqhhVVT9tf71TewfeFj2Y38A?cluster=devnet) |
| 28 Sep | Close of devnet's copy of the mainnet id (0.3265678 SOL returned) | 505,137,697 | [P6SZmM8T…](https://explorer.solana.com/tx/P6SZmM8T8LMBqca5zDKEP9aARMnhCCtLqzkmFd5fccvQ5bYvCTWSm1r15tunDFALBzumdR2yvBNRc85UcHtjKeP?cluster=devnet) |
| 28 Sep | Rehearsal: ProgramData extension by 10,240 bytes | 505,139,868 | [43ZCsEiQ…](https://explorer.solana.com/tx/43ZCsEiQK1vuH4MPgtPoUJkypRTsYSM2f5nWnmrsRcECW6UhQnmzhbRZHuh14DjYHMm3AeuSTKSdS6BUWH63i1NZ?cluster=devnet) |
| 28 Sep | Rehearsal: upgrade to the 65,504-byte fee-side build (hash `337fe41b…0f0a`) | 505,140,142 | [3VU2c4ZZ…](https://explorer.solana.com/tx/3VU2c4ZZ8UaPKFLT6S2LZDFuyHrnR41Jj8W9WYxfeqZzAgwb8AJGMxaYVnX5kACzfSbovMUm3tQze2rgY2k8gSdm?cluster=devnet) |

The five failed `--use-rpc` attempts never produced a transaction (the
client gave up after its retries), so they have no signature.

### 7.2 Lifecycle on the build now on mainnet (28 September, devnet's own id)

| Step | Bytes | CU | Link |
|---|---:|---:|---|
| Initialize | 300 | 1,811 | [HvLfcoek…](https://explorer.solana.com/tx/HvLfcoek7sBvj2dt72H4uSAvxc8twKRkA85nQSrUYqHCuvGb9JBtedshPwS5Fw1gq9w5ohf7jJrMoeFS7jrcXTW?cluster=devnet) |
| Fund (System transfer) | 236 | 150 | [4xKNPiA5…](https://explorer.solana.com/tx/4xKNPiA5WFa1bhpphJt88qg4HnS83yGeSpcDkbDcYtB8uQwdXzswr1oYwt18LK9QekQPKWXXh9J9GZ5fgPwA3Amf?cluster=devnet) |
| Transfer out of the vault | 1,282 | 38,975 | [3KxSweya…](https://explorer.solana.com/tx/3KxSweyamJ3moAmgAVXr4tgyjzTM4K1hE6uJqJtsgNSUFm21facrTBwofX2QbHScbB1AnAXTvzXZEVb6W7BYNA6m?cluster=devnet) |
| A transfer that would take the vault below its rent floor: refused in simulation (`RentFloor`, code 5), never sent | 1,282 | 38,942 | simulated only |
| Top-up of the vault (System transfer) | 236 | 150 | [2XZgevjZ…](https://explorer.solana.com/tx/2XZgevjZu2cFAVMVxqUfY54YpwpsJSVgWprwkrYzad2eYFLAqTMAnc5fXPeEBHaviHFagnkdjqgyG196weJEW37s?cluster=devnet) |
| The refused transfer's signature, re-sent after its expiry slot: refused in simulation (`Expired`, code 3), never sent | 1,282 | 2,114 | simulated only |
| Recovery (the fixed recovery message: leaf 255, rotate to the next tree) | 1,313 | 38,840 | [5vb4Zmwa…](https://explorer.solana.com/tx/5vb4ZmwagVeG2FCCakg7xgXVAUDR6kxAW9ZxFiPjBfbkxepe9JVA4oDYXEU65fQvN1dqJ2UoSut5o6UJkoq58g1t?cluster=devnet) |
| Retry of the original transfer on the new tree | 1,282 | 38,976 | [4FmAZVxu…](https://explorer.solana.com/tx/4FmAZVxuSPAGpyb2DsVfbU48GA4Y3zmZupmJwAjSWgTTYTauZbiuAQdVTBWGoUG3YjmDU8L35c979nMnPUsSYW3P?cluster=devnet) |
| Rotation | 1,323 | 39,074 | [568mhVnb…](https://explorer.solana.com/tx/568mhVnb9uLzRuECCx6QkuPm6W8cjtmcJP3bJqFrcB21aa9t7xq6qSvigqUEBQJBpQvxNUF3AUAjs95R1jhPVJiU?cluster=devnet) |
| Close (leaves a tombstone) | 1,270 | 38,556 | [Ji3N5gAT…](https://explorer.solana.com/tx/Ji3N5gATYzSeNGzHKVt3vpRUnmmHgKZ1zUK67N1zjRH5yAvZoDzjcxKcDYkTrcscQoQirkyMviaNALem2iEBW8c?cluster=devnet) |
| Reopen | 1,313 | 38,857 | [5sm29XSc…](https://explorer.solana.com/tx/5sm29XScGrezT5FFArG2L8DxDENJcaYG5E8swN5oJ5RV9MtwFrqao2LJo59NBoQMVAxa3ApgPjt1k7WyCEUogjoa?cluster=devnet) |

### 7.3 The first full lifecycle (19 September, the shared id)

The first live submission on 19 September was rejected by the devnet RPC
before execution. Its response, saved verbatim:

```
code -32602
failed to deserialize solana_transaction::versioned::VersionedTransaction: Custom error: invalid message version
```

Cause: the transaction was serialised as `count ‖ signatures ‖ message`,
the legacy and v0 layout; Transaction v1 is `message ‖ signatures`. After
the fix the full lifecycle landed:

| Step | Bytes | CU | Link |
|---|---:|---:|---|
| Initialize | 300 | 1,811 | [3iMaFJTF…](https://explorer.solana.com/tx/3iMaFJTF1QxYXhx6qnr2T1LF8LEeDisFatNCyzEx5G3RmNwPaU1NskTnun9qAq4G9aXFbRgUhZswUZvJmSSJhAoe?cluster=devnet) |
| Fund | 236 | 150 | [2h7yzZWx…](https://explorer.solana.com/tx/2h7yzZWxHDucsHXEBJzBfQ5AqJHaaSYwnkmXYybsoEErzyBLoTUN8gPZjkdYTm8uiiLirV8tGBQJhJVQiKQfvR3e?cluster=devnet) |
| Transfer | 1,282 | 38,960 | [2dWJhqmZ…](https://explorer.solana.com/tx/2dWJhqmZ7PBYAvLsF8kkBNeTPDhwir2vhF5fsPUGrbZMz9KzP64PWdbSRdhhDSN3vssUjhBeppUkA4WzbCngfUcA?cluster=devnet) |
| Top-up | 236 | 150 | [3XuWGtbb…](https://explorer.solana.com/tx/3XuWGtbbL5YR8uZXpPN5wASgZZNv4FBpyky21HKEWyKqkS9wB3aGoqvRcbkTxv4yv91Rauaes7oAvaK3NUKHyYBg?cluster=devnet) |
| Recovery | 1,313 | 38,820 | [2PKXu5bT…](https://explorer.solana.com/tx/2PKXu5bTZszsUteD6p9KXVEpaRm2NwpauV8dNBQHD1D3P551qM4SThxonzPRtup8PApE1wqhieV3yEaYebTgCpTa?cluster=devnet) |
| Retry of the original transfer | 1,282 | 38,964 | [4xXBqsYL…](https://explorer.solana.com/tx/4xXBqsYLNLffmEsSZeEywh4dE3vHD5Y49SdSn1jBoxensMLGTW2hPV2BRFngMhmCQwYz1fGrHmreS6casm7iQBMj?cluster=devnet) |
| Rotation | 1,323 | 39,055 | [2m9Eqicz…](https://explorer.solana.com/tx/2m9EqiczHmPgot5RsNsfHx9etyfBR6JVcyGG7wPRiKJcMrAQH4Mq65Rtfn23Wf1kyD2J51JL7Buzjjs1Bvcw2YJj?cluster=devnet) |
| Close | 1,270 | 38,560 | [5vJApYTM…](https://explorer.solana.com/tx/5vJApYTMmRKsdS3CRiHXMkNXxd93vYpFTecgYVmEJSwfWXGq4S1pgGfS5DGUK7RpwATCFfDuomsCc84FigWiSpev?cluster=devnet) |
| Reopen | 1,313 | 38,837 | [YomKr44r…](https://explorer.solana.com/tx/YomKr44rGRRNmzVuWP35KTjZYXgWcm8SSc3YoVJbsy2uMHoo7Jb5Fh721LnTusC9Mtvb4ez5axUfLQ18TcJBggo?cluster=devnet) |

These ran under the shared id whose devnet copy was closed on 28
September; the transactions remain readable, but devnet history can be
pruned.

## 8. Measured budgets and what produced them

| Figure | What | Produced by |
|---|---|---|
| 300 B, 1,811 CU | Initialize | M3 (mainnet); devnet runs, Section 7 |
| 1,270–1,323 B, 38,556–39,074 CU | transfer, recovery, retry, rotation, close, reopen | devnet run of 28 September on the build now on mainnet (7.2) |
| 1,459 B, 55,694 CU | wrap stray SOL | M5 |
| 2,163 B, 129,328 CU; 2,163 B, 127,343 CU | canary swap; the rejected swap | M8; M6 |
| 2,159 B, 127,782 CU; 2,608 B, 170,490 CU; 2,130 B, 128,509 CU | the three swaps after the upgrade (the largest landed swap is M18) | M17, M18, M19 |
| 2,159 B / 24 accounts / 141,275 CU | SOL→USDC, 1 hop, Raydium CLMM | replay (below), first build; 141,371 CU on the fee-side build |
| 2,612 B / 37 accounts / 176,141 CU | SOL→USDC→SPL token, 2 hops, Meteora DLMM | replay, first build; 178,980 CU on the fee-side build |
| 2,099 B / 22 accounts / 171,572 CU | SOL→Token-2022 token, FluxBeam | replay, first build; 164,483 CU on the fee-side build (the fee now comes from the SOL input) |
| 53,156–58,022 CU (first build); 53,252–59,365 CU (fee-side build) | the vault program's own share of those three swaps | the same replays, CU attributed per program |
| 95,997–248,898 CU; at most 2,877 B and 45 addresses | 39 liquid-staking routes through the vault | replay of 4–5 October against the mainnet build `337fe41b…`, one pinned snapshot per route (the first 38 at slots 453,391,090–453,410,586; Section 12) |
| about 36,500 CU | XMSS verify, per the upstream crate | the crate's [README](https://github.com/blueshift-gg/solana-winternitz/blob/d6c381a46929d2f8973619f38b66d79058c39b5a/README.md) at the pinned commit |
| 36,420 CU | verify plus bitmap write in the crate's own example program | our run under mollusk-svm 0.15.1, SBPF v3 |
| 37,695 CU; 907 CU | a complete Execute with no inner instructions; a replayed leaf rejected | program tests against the compiled program, SBPF v3 |
| 3,640 of 4,096 B | the stack frame holding a 64-account CPI (`CpiAccount` is 56 B) | compiled and disassembled; pinocchio 0.11.2, platform-tools v1.56 |
| 65,504 B (64,112 B before 28 September) | program size | on chain, Section 3 |

**Replay** means: a live Jupiter quote, every account the transaction
references and the executable bytes of every program it touches (each
with its sha256) are pinned at one mainnet slot, then the exact
transaction is executed in-process in [LiteSVM](https://github.com/LiteSVM/litesvm)
(0.16) with the real Jupiter and venue binaries, swapping in only our
program. It is program-level evidence on real programs; it says nothing
about fees, scheduling or inclusion. The snapshots, quotes, program
hashes and the replay harness are published at the open release; the
harness also as part of milestone 2.

## 9. Tests

Last recorded run of each suite:

| Suite | Date | Result |
|---|---|---|
| Rust workspace: 49 program tests against the compiled program, 38 client, 7 local RPC shim | 27–28 Sep | 94 passed |
| TypeScript SDK | 28 Sep | 140 passed; 3 skipped (they need devnet) |
| Browser end-to-end (Playwright, one worker, against the real program through the local RPC shim) | 4 Oct | 104 passed; 9 opt-in skipped; 113 in all |

That is over 320 tests. A further 11 Rust probe tests are opt-in and not
counted. The program and the SDKs have not changed since the Rust and
TypeScript runs; the browser suite grew with the app work of 2–4 October.
The commands are `cargo test` (workspace), `npm test` (TypeScript SDK) and
`npx playwright test` (app); the tests are published at the open release.

## 10. App build record

The app is a static site. Each build publishes `/BUILD-HASH.txt`: the
sha256 of a manifest listing every served file with its own sha256.

| Deployed | Build hash | Program check on the live verify page |
|---|---|---|
| 27 Sep | first deploy; only the prefix `1cbf9ecd` was saved | |
| 28 Sep | redeploy after the program upgrade; hash not saved | program MATCHES `337fe41b…0f0a` |
| 2 Oct | `f10a5e59354eb7f04bc67c75e78242dbf3cecda5541cca5a03511d499dbba3aa` | MATCHES |
| 4 Oct | `96a2a034a293b15553c32ef1a32cbcfe3c0220e5ad3c53b5336c449de7c10b89` | MATCHES |
| 4 Oct (live on 5 Oct) | `d60ba25a92fb1db2713871db798f631d45dd6ed5713f5ee8a8742c7fbe36bb7c` | MATCHES |

## 11. Program error codes

Codes returned by the deployed program as `custom program error`:

| Code | Name | Code | Name |
|---:|---|---:|---|
| 1 | InvalidSignature | 12 | SettleRequired |
| 2 | LeafUsed | 13 | SlotMismatch |
| 3 | Expired | **14** | **SettleFailed** (`0xe`) |
| 4 | ExecutorNotSigner | 15 | ForeignVaultAccount |
| 5 | RentFloor | 16 | TooManyAccounts |
| 6 | Unsupported | 17 | TooManyInstructions |
| 7 | BadVaultState | 18 | BadInnerProgram |
| 8 | BadAuxIndex | 19 | LamportMoveConflict |
| 9 | BadVaultAddress | 20 | SameKey |
| 10 | BadLeaf | 21 | LastLeafNeedsRotation |
| 11 | NotAllowed | 22 | FeeAccount |

## 12. Staking: the public facts behind the brief's Section 7.3

All reads are from the public mainnet RPC on 4 and 5 October 2026.
Nothing was signed or sent.

**Minimum delegation is 1 SOL.** A `simulateTransaction` (signature
verification off) of one instruction to the Stake program
(`Stake11111111111111111111111111111111111111`, data `[13,0,0,0]`,
`GetMinimumDelegation`, no accounts) at slot 453,391,543 returned
1,000,000,000 lamports (return data `AMqaOwAAAAA=`) in 630 CU. Anyone can
repeat it with any funded fee payer. The 1 SOL minimum came with Stake
program v5.0.0
([SIMD-0490](https://github.com/solana-foundation/solana-improvement-documents/pull/490)),
switched on by feature `STk5Xj8hdAx3sTzmtJ3QysKkq6X2A3yj73JtxttiRyk` at
slot 427,248,000 (epoch 989).

**A stake account needs 1,001,666,240 lamports:** rent for its 200 bytes
(`getMinimumBalanceForRentExemption(200)` = 1,666,240) plus the 1 SOL
minimum. One lamport less fails with the Stake program's `Custom(12)` (in
simulation).

**The Stake program has no upgrade key.** Its ProgramData
[`6WU8Nxarf9fudRK5atWwjLY4vFaw5UrrWhL88qz7iCMJ`](https://explorer.solana.com/address/6WU8Nxarf9fudRK5atWwjLY4vFaw5UrrWhL88qz7iCMJ)
has no upgrade authority; it was last deployed at slot 443,232,000, when
feature `s51VGwCAgebo2745DSUris72RavoLkXGUmVJosESCXr` (SIMD-0391) installed
[v5.1.0](https://github.com/solana-program/stake/releases/tag/program%40v5.1.0);
program bytes sha256
`3d2d39c596ce8be2d47816b4ee5db9fc759d80fde54b08c930ad0b6daed64c2c`
(212,056 bytes). In 2026 the Stake program has changed twice, both times
by feature gate.

**The token program is p-token.** The SPL Token program's ProgramData
[`3gvYRKWyXRR9xKWe1ZjPhLY5ZJRN7KDB4rFZFGoJfFk2`](https://explorer.solana.com/address/3gvYRKWyXRR9xKWe1ZjPhLY5ZJRN7KDB4rFZFGoJfFk2)
has no upgrade authority and was deployed at slot 419,472,000 by feature
`ptokFjwyJtrwCa9Kgo9xoDS59V4QccBGEaRFnRPnSdP`
([SIMD-0266](https://github.com/solana-foundation/solana-improvement-documents/pull/266));
see the [p-token v1.0.0 release](https://github.com/solana-program/token/releases/tag/p-token%40v1.0.0).
Its instruction 45, `UnwrapLamports`, moves lamports out of a
wrapped-SOL account without closing it. The staking design uses it to fund
a new stake account from the vault's wrapped SOL inside one Execute.

**Validators.** From `getVoteAccounts` at epoch 1049: 671 current vote
accounts and 15 delinquent. The default-set rules filter them as follows:
credits in each of epochs 1045 to 1048 leaves 670; commission at most 5%
leaves 540; excluding the 18 largest by stake (together 33.65% of all
stake, the superminority) leaves 532. The 532 hold 38.35% of current
stake, and 222 of them charge 0%.

**Liquid staking tokens through the vault (replay).** 42 route snapshots
captured on 4 and 5 October from Jupiter's public API, replayed against
the mainnet program build `337fe41b…`:

* 39 landed through the vault with the settlement check and the protocol
  fee, across 11 venue labels (Meteora DLMM, Raydium CLMM, Manifest,
  Whirlpool, Sanctum Infinity, Sanctum Prop S, AlphaQ, the Sanctum router
  with three stake-pool programs and Marinade, Meteora DAMM v1, JupLend
  AMM, VaultLiquidUnstake).
* 3 were refused by the client before signing, as designed: a non-shared
  two-step split, and two routes with 65 and 70 account entries, over the
  64 the client allows in one instruction.
* For the first 38 snapshots the same routes were also run with a plain
  wallet as the trader, as controls; all 76 control runs landed. The vault
  never funded a System transfer.
* AlphaQ, the only venue whose routes name the Instructions sysvar,
  landed but paid the vault 0.05 bps less than the plain wallet in every
  case, for a reason not yet determined.
* Twelve routes run at call depth 5, the current mainnet limit: they work
  today with no headroom.
* Budgets: 95,997 to 248,898 CU, at most 2,877 bytes and 45 addresses.

Status: tested on real programs (pinned replay, one snapshot each). None
of these tokens has been swapped through a vault on mainnet yet.

**Native staking prototype (not deployed).** Two throwaway prototypes of
the program change were built (cargo-build-sbf 4.2.0, platform-tools
v1.56) and run in LiteSVM 0.16 against the Stake v5.1.0 and p-token
v1.0.0 program bytes pinned from mainnet, mainnet rent and epoch
schedule, and two real vote accounts. An unmodified copy of the program
builds byte-identical to the mainnet program, so the figures compare with
mainnet itself. The final-rules prototype:

| Measure | Value |
|---|---|
| Program size | 74,128 bytes; mainnet's ProgramData holds 74,352 |
| Stake (fund, create, initialise, delegate in one Execute) | 72,586 CU, 1,639 bytes, 9 addresses |
| Deactivate; withdraw; partial unstake | 52,117 CU; 52,551 CU; 77,222 CU |
| Rule-breaking payloads | all 37 refused before signature verification; no leaf used |
| Existing program tests | all 49 pass on the prototype |

Status: proposed design; prototypes tested locally; not yet through the
internal review gate. The deployed program is unchanged.

**Two small gaps in today's settlement check.** Found during this work.
Both concern how lamports are measured in a swap's token accounts; both
are zero for every vault account that exists, because every one was
created at the current rent rate, and would grow only if rent per byte
falls. A hostile mock venue reproduces both locally, and the final-rules
prototype refuses both. The fix is in the milestone 3 upgrade and the
audit's scope.

## 13. Competitor checks (5 October 2026)

| Project | What we checked | Links |
|---|---|---|
| Winterwallet (Blueshift) | Program: first transaction 27 April 2026, last 28 July 2026 (`getSignaturesForAddress`). Repository: three commits, all on 27 April 2026. Its README describes a command-line tool, a client and a TypeScript SDK, not formally audited. `MAX_CPI_INSTRUCTION_ACCOUNTS = 16` | [program](https://explorer.solana.com/address/winter5vMwvf51xrSVPTxbAAD6qiSTmPeRTSizMCQCa), [code](https://github.com/blueshift-gg/winterwallet/blob/672fc6789b1532ee680f24842d235e0be8737b61/common/src/lib.rs) |
| ORE's Falcon wallet | A Falcon-512 command-line wallet on mainnet, built on Blueshift's Vector, that transfers and stakes ORE; no swap or settlement check | [ore-cli](https://github.com/regolith-labs/ore-cli), [Vector](https://github.com/blueshift-gg/vector), [Falcon-512](https://github.com/blueshift-gg/solana-falcon512) |
| winternitz.io | Program active 9 to 23 August 2026 (`getSignaturesForAddress`). The site describes a launchpad whose treasury sits in a Winternitz vault; trading happens on Meteora; the first item of its roadmap is a quantum-safe wallet with in-browser signing | [site](https://winternitz.io), [program](https://explorer.solana.com/address/13EtnfYGUH8NaGAnUpDTVgSsXoewNnULp7ESwHzQUANT), [code](https://github.com/rabb757/winternitz-vault/commit/e85eae772d2f37f11bfc11b527b761f7cea7d341) (no licence file) |
| quantumsol.tech | A browser app on mainnet: deposit into a hash-signature vault, withdraw in two transactions; says it is not audited | [site](https://quantumsol.tech), [program](https://explorer.solana.com/address/BPzp6SKxzDq6au2hg4L6hVJk5dYTRtb3hsFNwLUaqB9P) |
| qsol.day | "Quantum-safe custody on Solana (devnet beta)"; a browser app built on Winterwallet's crates | [site](https://qsol.day), [code](https://github.com/qsolday-cyber/qsol) |
| QShield | ML-DSA-44 vaults with a browser wallet, devnet only, "research preview v0.1" (repository created 4 October 2026) | [code](https://github.com/sol-9/QShield), [app](https://qshield.hield-main.workers.dev) |
| Qubit | Program `34CJhzSBAptiSadvHZK4A1PhpcfdsbguyRXqnUQPpCiD` deployed on 31 May 2026 and closed on 3 June 2026 (the loader logs "Closed Program"); its ProgramData account no longer exists | [deploy](https://explorer.solana.com/tx/2aNWVkugBfFhwjCUb41VfoUFG3eKzfCygWc1NSYADZ53K9n7CuYk3jiCs5QWqphDst9R8SjqCniEgaAkfQtMJAFG), [close](https://explorer.solana.com/tx/DyoGU8j4ZCwK8Ci2DfLceMHpk8ScFDdXiqbWiwybGivNThbmgSKfDqa4BM5NKiFijjegdUBkd2yvCmwj45K85mE), [code](https://github.com/QuantiumResistantInfra/quantum-vault/commit/47e06d5af6fc2c011bdc193eda72be23c9ff6184) |

## 14. Prior work we build on

Every reused component, with the upstream repository at the commit we
studied and its licence. Files were read at these commits; the vault's
verifier is a pinned dependency, not a copy.

| Component | Upstream, at the commit studied | Licence | Used in |
|---|---|---|---|
| XMSS / target-sum verifier, keccak syscall binding, used-leaf bitmap check-then-set, TypeScript verifier and signer, chain-start and salt derivation layout, DKKW25 parameter checker, conformance vectors | [blueshift-gg/solana-winternitz @ d6c381a](https://github.com/blueshift-gg/solana-winternitz/commit/d6c381a46929d2f8973619f38b66d79058c39b5a) | MIT | program (pinned dependency), SDKs, tests |
| Digest that binds every passthrough address and byte; CPI payload format; positional passthrough parser; state written before the CPI; signer promotion by address equality; pinocchio skeleton; BIP-39 validation and hardened derivation with test vectors; persist-before-send client, atomic state file, simulate-then-pad, size estimators | [blueshift-gg/winterwallet @ 672fc67](https://github.com/blueshift-gg/winterwallet/commit/672fc6789b1532ee680f24842d235e0be8737b61) | MIT (Dean Little) | program, SDKs, tests |
| PDA equivalence check (about 150 CU); exact-length parsing; close and sweep | [blueshift-gg/solana-winternitz-vault @ beb312c](https://github.com/blueshift-gg/solana-winternitz-vault/commit/beb312c7a62914079d68e7eee2c1a3e44bda7a3d) | MIT (Dean Little) | program |
| Two-phase authorisation idea (ancestor of BOUNDED mode) | [deanmlittle/solana-winternitz @ 6a4057f](https://github.com/deanmlittle/solana-winternitz/commit/6a4057fc31cca29125f98cc28fa512eaf44a11b6) | MIT (idea only) | design |
| Relayer and wallet top-up UX; phrase display UX; LiteSVM harness shape; tombstone close | [QuantiumResistantInfra/quantum-vault @ 47e06d5](https://github.com/QuantiumResistantInfra/quantum-vault/commit/47e06d5af6fc2c011bdc193eda72be23c9ff6184) | MIT | app, tests, program (Close) |
| Calling Jupiter from a program: opaque passthrough of route data and accounts, signer only for the PDA | [jup-ag/jupiter-cpi-swap-example](https://github.com/jup-ag/jupiter-cpi-swap-example), [jup-ag/sol-swap-cpi](https://github.com/jup-ag/sol-swap-cpi) | pattern only, no code | program, SDKs |
| Jupiter instruction discriminators and account slots | [jup-ag/jupiter-cpi](https://github.com/jup-ag/jupiter-cpi) IDL | interface data | program constants |
| Transaction v1 wire layout | [SIMD-0385](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0385-transaction-v1.md); Agave's RPC decode path ([wincode](https://github.com/anza-xyz/wincode)) | public docs | SDKs |
| winternitz.io vault | [rabb757/winternitz-vault @ e85eae7](https://github.com/rabb757/winternitz-vault/commit/e85eae772d2f37f11bfc11b527b761f7cea7d341) | none found | read only; nothing used |

**New in this project:** the vault program (`Initialize`, `Execute`,
`Close`), the `vault_id` construction, the header clauses (`LAMPORT_MOVE`,
`SETTLE`, mandatory expiry, `NEXT_KEY`, `EXECUTOR`, `BOUNDED`), the digest
layout, the allow-list and slot table, the settlement invariants, the
64-account executor, in-band rotation, phrase-to-tree derivation including
P, the leaf ledger and its policies, the v1 builder and estimator, the
Jupiter plan validator, the test suites and the documentation. MIT at the
open release.

## 15. Review record (summary)

Every milestone closes only after an internal adversarial review gate
returns CONTINUE (method: brief, Section 3.1). From the first internal
assessment on 15 September to 4 October: seven gate reviews and twelve
closure rounds to 28 September, then four rounds on 3 and 4 October. They
recorded 69 numbered findings. Ten were release blockers, all resolved:

| What the review found | What changed |
|---|---|
| Key derivation did not separate deployments | the program id is an input to every key, so keys for different deployments are unrelated |
| An interrupted recovery could reuse a recovery leaf | its first fix was withdrawn at gate 1 (next row); the final design is one immutable recovery message per tree |
| That first fix chose the recovery leaf from the slot, so two restores 16 windows apart signed different messages with one key (gate 1) | replaced by the single recovery message: leaf 255, no expiry, identical bytes on every restore |
| A tree marker did not prove complete signing history after a restore | a burned leaf retires its tree |
| Leaf reservation was not atomic across overlapping signing calls | atomic reservation in both stores; one signing call at a time |
| Retiring a tree did not cover a restore onto an unchanged chain | bounded restore: markers expire after about an hour (9,000 slots) unless renewed |
| An open signer checked marker expiry against a slot that did not advance | every reservation takes a fresh on-chain slot reading, atomically |
| The confirm step could sign a worse minimum than the review panel showed (gate 5) | the signature covers exactly the reviewed plan; stale quotes are refused |
| A confirmed or unknown transaction was reported as definitely not landed (gate 5) | the app now tells apart rejected in simulation, submitted but unknown, confirmed success and confirmed failure |
| Create or Restore could overwrite a device's only gas-tank key (gate 5) | onboarding refuses an occupied device until the forget flow completes |

Of the other 59, 58 are closed (fixed, or accepted with a written
reason). One, about the app's display after a lock, is open by the
owner's decision and waits for a design review (brief, Section 10). The
full log, with every finding's failure scenario, response and status, is
published at the open release.

## 16. What is published at the open release

At milestone 5: the program, the Rust and TypeScript SDKs, the
settlement-check library and all test vectors under MIT; the tests; every
saved RPC response behind this page; the route snapshots and pinned
program hashes behind the replays; and the full review record. The web
app stays closed; its build hashes are published (Section 10).
