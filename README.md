# OddsIntel — Public Track-Record Ledger

This folder holds **daily, byte-identical, GitHub-signed, blockchain-anchored
snapshots** of every settled production pre-match bet placed by the OddsIntel
football model.


> **Gap: 2026-07-13 → 2026-09-20 — no daily snapshots.** The publishing workflow lost
> its database connection in the Supabase→VPS cutover of 2026-07-13 and failed on all
> **71** nightly runs until it was repaired on 2026-09-21
> (`LEDGER-WORKFLOW-RED-EVERY-DAY-2026-09-21`). It went red twice over: `UndefinedTable`
> until 2026-07-30, then `connection refused` on `localhost:5433` once `DATABASE_URL`
> moved to the tunnel DSN.
>
> **No bet data was lost.** Each daily file is cumulative since 2026-05-04, so the first
> snapshot after the repair contains every bet from the gap. What is missing is the
> per-day proof-of-existence for those 70 days — the signed commit and the
> OpenTimestamps stamp. Those were **deliberately not backfilled**: a file written in
> September cannot honestly carry an August commit date or Bitcoin attestation, and
> fabricating them would forge the exact audit trail this ledger exists to provide.
>
> Freshness is now machine-checked by smoke `LEDGER-SNAPSHOT-FRESH`.

## Verification mechanic — three independent anchors

1. **Live source of truth.** `https://oddsintel.app/api/v1/track-record`
   serves the same data straight from the production database (production
   strategies: calibrated + beta + active maturity, no retired; pre-match
   markets only; settled only).
2. **GitHub-signed daily commit.** Once per day (22:45 UTC) a GitHub
   Action runs `scripts/export_track_record_snapshot.py`, writes a
   deterministic JSON file to `ledger/YYYY-MM-DD.json`, updates
   `latest.json` and `index.json`, then commits as `github-actions[bot]`.
   GitHub verifies the commit signature — the commit history itself is
   the audit trail.
3. **OpenTimestamps Bitcoin anchor.** Each daily snapshot is hashed and
   submitted to the OpenTimestamps calendar servers; within ~1-6 hours
   the resulting `.ots` file is updated to include a Bitcoin block-header
   proof. After that, anyone can `ots verify ledger/YYYY-MM-DD.json` and
   independently confirm "this exact JSON existed at this Bitcoin block
   height" — without trusting GitHub, without trusting us, without
   trusting any centralized party.
4. **SHA-256 in index.json.** Lists the hash of every daily file. Any
   future edit would be visible in git history AND would break the
   recorded hash. The OTS proof is locked to that exact hash.

## What's in each daily file

```jsonc
{
  "snapshot": { "date": "...", "generated_at_utc": "...", "scope": "..." },
  "summary": {
    "since": "2026-05-04",            // calibrated-tier launch
    "total_bets": 624,
    "stake_total": 3586.18,
    "pnl_total": 343.25,
    "roi_pct": 9.5715,
    "median_clv_pin_pct": 2.10,       // robust — see note below
    "clv_pin_coverage_pct": 97.27,    // % of bets with a Pinnacle close
    "clv_pin_beat_pct": 56.01,        // % of picks with CLV>0
    "scope": "calibrated bots, pre-match markets (1x2, OU 2.5, BTTS), settled only"
  },
  "bets": [
    {
      "id": "...",
      "match_id": "...",                   // UUID stable across re-settle
      "kickoff_utc": "2026-05-04T14:30:00+00:00",
      "league": "Liga I",
      "country": "Romania",
      "market": "1x2",
      "selection": "home",
      "placed_odds": 3.35,
      "bookmaker": null,                   // the book's DISPLAY name if set (see note below)
      "placed_at_utc": "2026-05-04T04:27:01Z",
      "closing_odds": 3.15,
      "clv_any_pct": 6.35,                 // vs any-book close
      "clv_pin_pct": 5.35,                 // vs Pinnacle close
      "stake": 5.0,
      "pnl": -5.0,
      "result": "lost",
      "score": "2-2",
      "bot": "bot_v10_all"
    },
    ...
  ]
}
```

## How to verify each bet yourself

Take any row and:

1. Cross-reference `match_id` + `kickoff_utc` against
   ESPN / Flashscore / Pinnacle archive — find the same fixture.
2. Confirm the final `score` against that public source.
3. Apply the bet definition (`market` + `selection`) to the score to
   derive the result yourself — won/lost/void.
4. Check `placed_at_utc < kickoff_utc` to confirm our database recorded the
   bet pre-kickoff. That field is our own timestamp, so it is a consistency
   check, not independent proof: the independent evidence that a pick existed
   before kickoff is its Telegram channel post. The snapshot + OpenTimestamps
   only prove the row has not changed since the night it was first published
   (corrected 2026-09-28, #230).

## Why median CLV not mean?

Some historical "closing" Pinnacle snapshots in our `odds_snapshots` table
are vintage — captured hours or even days before kickoff because Pinnacle
didn't publish odds late, or because the `is_closing` flag was set on a
sub-optimal snap. Those produce outlier CLV values (±50%+) that wildly
swing the mean.

Median is robust to this. As of the
CLOSING-LINE-COVERAGE fix (engine commit `f5c0c94`) landing 2026-06-24, every imminent match (T-15 → T+5) now gets a
fresh per-fixture Pinnacle snap every 5 minutes, so the noise tail will
decay over the coming weeks and the mean will become trustworthy again.
Until then: **median is the publishable number.**

## Schema stability

The bet-row schema is intentionally narrow — no model-internal scores,
no calibrated probabilities, no signal weights. That's by design: this
ledger is for external verification of outcomes, not for replicating
the model. If you want the inputs, train your own model on public data.
The engine's source is private since 2026-09-29; this ledger is published separately at
[github.com/msellin/oddsintel-ledger](https://github.com/msellin/oddsintel-ledger), mirrored
from the engine after every change, and every file in it can be checked without our code
(`ots verify <file>` for the nightly snapshots, the RFC 3161 tokens in `sealed/` with openssl
or oddsintel.app/verify).

If we have to evolve the schema, additive-only changes will land in a
new field; we will not silently rename or remove fields.


## Note — bookmaker names (2026-10-02)

From the 2026-10-03 snapshot on, `bookmaker` reads `Other bookmaker` where it used to say `1xBet` or `Paripesa`.
Those two brands are not named on our public surfaces while a legal question is open. Every other value is unchanged.
Each snapshot holds every bet since 2026-05-04, so from 10-03 on the past rows of those two books read `Other bookmaker`
too. The snapshots up to 2026-10-02 are unchanged: they are timestamped and anchored, and rewriting them would break
their proofs. A row's price and result are identical in both forms.


## Note — the ledger is the /performance headline; prices are checked (2026-10-03)

From the first snapshot after 2026-10-03 the file holds exactly the picks and prices of the all-time figure on
oddsintel.app/performance. Until then it selected only the strategies active on the day it was written, so when one
was retired on 2026-10-02 its 472 earlier picks left the file (n 936 → 464) — the page had stopped doing that on
2026-09-29. What changes per row (no field is removed or renamed):

- **cohort**: every pick logged before kick-off while its strategy was Active, retired strategies included, plus the
  sharp-line forward test (`source` = `sim` or `forward_test`, new field).
- **`placed_odds`**: the best price on any publishable bookmaker at pick time (the page's price). A price more than
  1.25× Pinnacle's at that moment that none of the other bookmakers then quoting came within 5 % of is re-graded at
  the best price another bookmaker actually offered; `price_verdict` (`clean` / `phantom` / `unverifiable` / null =
  not checked), `price_repriced` and `odds_public_as_recorded` (new fields) let you redo it.
- **`price_basis`** now uses the page's vocabulary: `available` (all publishable books), `published` (forward test),
  `recorded` (no quote stored at pick time), `our_books`.
- **`stake`** is a flat 10 for every row and **`pnl`** is at `placed_odds` — the page's method. `pnl_stored` now holds
  the same result in units (1 = one stake), no longer a Kelly-staked euro amount.
- **`closing_odds`** and **`clv_any_pct`** are null; **`clv_pin_pct`** is the page's closing-line figure (the sharp
  close: Pinnacle, else a 5+-bookmaker consensus).

Snapshots up to 2026-10-03 are unchanged (timestamped and anchored). The dated restatement with its numbers is on
oddsintel.app/record/corrections.
