# Pick seals — proof that every pick existed before kick-off

Since 2026-09-28 (#235) every new pick is **sealed before kick-off by an independent Time-Stamp Authority**
(RFC 3161 — DigiCert's public TSA, FreeTSA as fallback). Every 5 minutes we hash the new picks and send
**only that hash** to the TSA. The TSA signs it with **its own clock**. We cannot backdate that time, and
we cannot change a pick after it is sealed, because its hash would stop matching.

Why this and not the rest of `ledger/`: the nightly snapshots and their Bitcoin stamps are taken **after
settlement**, so they prove a result existed, not that the pick existed before the match. Telegram posts
prove the picks that were posted, but not every pick is posted. The seal covers every pick.

## Files
- `<date>/<batch>.jsonl` — the exact bytes that were hashed: one pick per line (bot, match, market,
  selection, recorded odds, pick time, kick-off). Since `"format":2` (2026-09-28) each line also carries
  `graded_odds` / `graded_basis`: the price the public record grades the pick at (best price available on
  any publishable book at pick time), so that price is provably fixed before kick-off as well.
- `<date>/<batch>.tsr` — the TSA's signed token over `sha256(<batch>.jsonl)`.
- `index.json` — every batch with its sha256, TSA, TSA time and pick count.

A batch is published only after **all** its matches have kicked off, so a pending pick is never revealed early.

## Verify a pick yourself
```bash
# 1. find the pick's line
grep '"pick_id":"<id>"' ledger/sealed/*/*.jsonl
# 2. check the token covers exactly this file, and read the TSA's time
openssl ts -verify -data ledger/sealed/<date>/<batch>.jsonl -in ledger/sealed/<date>/<batch>.tsr \
  -CAfile /etc/ssl/certs/ca-certificates.crt        # macOS: -CAfile /etc/ssl/cert.pem
openssl ts -reply -in ledger/sealed/<date>/<batch>.tsr -text | grep "Time stamp"
```
If the TSA time is earlier than the pick's `kickoff_utc`, the pick provably existed before the match.
If your CA bundle lacks the TSA's intermediate certificate, extract the chain from the token:
`openssl ts -reply -in <batch>.tsr -token_out | openssl pkcs7 -inform DER -print_certs > chain.pem`
and pass `-untrusted chain.pem`.

Writer: `workers/jobs/pick_seal.py` (scheduler job `pick_seal`). Publisher: `scripts/export_pick_seals.py`
(nightly, `.github/workflows/track_record_ledger.yml`). Tables: `pick_seal_batches`, `pick_seals` (migration 501).
