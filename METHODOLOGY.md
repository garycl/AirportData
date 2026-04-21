# AirportData — Methodology & Caveats

This repo publishes aggregated BTS aviation data. The code that produces it lives at **[github.com/garycl/dot-download](https://github.com/garycl/dot-download)**.

Machine-readable ranges and per-dataset notes are in [`data_availability.json`](./data_availability.json).

## Datasets

| Dataset | Path | Grain | Period | Source |
|---|---|---|---|---|
| DB1B Market (aggregated, by origin) | `db1b_mkt/dbmkt_YYYY_aggregated.parquet` | Origin × Quarter | 1993Q1 – current | BTS DB1B + OD40 |
| DB1B Market (O-D pair) | `db1b_mkt/dbmkt_YYYY.parquet` | Origin-Dest × Quarter | 1993Q1 – current | BTS DB1B + OD40 |
| DB1B Ticket (aggregated) | `db1b_tix/dbtix_YYYY_aggregated.parquet` | Origin × Quarter | 1993Q1 – current | BTS DB1B + OD40 |
| T-100 Segment / Market | `t100_{seg,mkt}/t100_*_YYYY.parquet` | Carrier × Origin × Dest × Month | 1990M1 – current | BTS T-100 |
| OD40 / DB1C | `od40/` | Ticket, segment, market, carrier (monthly) | 2025-07+ | BTS OD40 (DB1C) |

---

## ⚠ Important: methodology break at 2025 Q2 → Q3

BTS retired the quarterly DB1B survey after Q2 2025 and replaced it with the **OD40 monthly (DB1C)** survey. The two surveys sample different fractions (10% vs 40%), changed reporting rules and reporting-carrier coverage, add new fields, and report at different time grains.

**The DB1B-shaped market and ticket files blend both surveys for 2025** so downstream dashboards see continuous time series:
- 2025 Q1/Q2: legacy DB1B (10% sample, BTS native proration)
- 2025 Q3/Q4: OD40 rescaled to 10%-sample equivalent

The structural break remains real and should stay visible through source-native fields and methodology notes. Comparable defaults are intended for trend charts, not for erasing every survey-design difference.

### Market fare/yield contract

For `db1b_mkt/dbmkt_YYYY_aggregated.parquet`, the default fare/yield fields are gross/tax-inclusive comparable fields:

- DB1B: `ItinFare * MktMilesFlown / MilesFlown`
- OD40: `TotalAmt * market_flown_miles / total_flown_miles`

Use additive components when combining airports or periods:

```
AvgMktFare      = sum(PWMktFare) / sum(MktFarePAX)
PWMktFareYield  = sum(PWMktFare) / sum(PWMktMilesFlown)
```

Do not average already-computed fares or yields across airports. `SourceNative` suffix columns retain the original DB1B/OD40-native basis for audit and break analysis. The explicit `*GrossComparable` columns are retained as lineage aliases for the default market fare/yield fields.

### Ticket fare/yield caveat

`db1b_tix/dbtix_YYYY_aggregated.parquet` extends through OD40 for Q3/Q4 2025, and passenger counts plus RT/OW shares are sample-rescaled. Ticket fare/yield fields are **not yet gross-comparable**: legacy DB1B `ItinFare`/`FarePerMile` are gross/tax-inclusive, while OD40-derived Q3/Q4 2025 ticket fare fields currently use `TotalAmt - TaxAmt`.

Treat ticket `ItinFare`, `FarePerMile`, `RTItinFare`, and `RTFarePerMile` as source-native across the 2025 Q2 -> Q3 boundary until separate gross-comparable ticket fields are published.

### Dashboard aggregation and NPIAS filtering

Fare allocation is done across the full itinerary and full set of markets before airport filtering. The dashboard then aggregates airport-period rows, filters displayed origins to the NPIAS-primary scope, and recomputes final totals, average fare, and yield from additive components. If an origin is NPIAS-primary and a destination is not, the destination remains part of that origin's network and fare/yield denominator.

### Pre-existing DB1B bug fixed 2026-04

`AvgMktFare` was previously computed as `MktFare / MktFarePAX` (a dimensionally-mixed hybrid that under-stated true pax-weighted average by the factor `mean(Passengers per row)`, ~2–3× for busy markets). It's now `PWMktFare / MktFarePAX`. **All historical aggregated files were regenerated in the 2026-04-17 push.** If you have local caches or downstream extracts from before that date, refresh them.

---

## Sample-fraction rules

| Dataset | Sample | Weight-to-100% multiplier |
|---|---|---|
| DB1B (pre-Q3 2025) | 10 % | × 10 |
| OD40 / DB1C (Q3 2025+) | 40 % | × 2.5 |
| T-100 | 100 % (census) | × 1 |

**Never sum DB1B and OD40 pax counts without reweighting.** For the dashboard-facing `dbmkt_YYYY_aggregated.parquet`, OD40 values have already been rescaled down to the 10%-equivalent scale so the `PAX` column is continuous.

---

## Carrier field guidance

Each dataset carries multiple carrier attributions with slightly different meanings. Pair them by semantics, not column name:

| Semantics | DB1B column | DB1C / OD40 column | Notes |
|---|---|---|---|
| **Marketing (brand)** — default | `TkCarrier` | `MktCarrier_N` (seg) | Cleanest cross-dataset match (<1 pp drift) |
| Operating (actual flyer) | `OpCarrier` | `OpCarrier_N` (seg) | Matches T-100 / Form 41 |
| Reporting (filer) | `RPCarrier` | `RpCarrier` (ticket) | **Do not cross-pair** — 3–5 pp phantom mainline inflation |

The per-origin aggregates strip carrier entirely (group by Origin × Quarter). For per-carrier analysis use `od40/carrier/` (new OD40 period) or join the raw DB1B quarterly files (legacy).

---

## Raw vs aggregated

Raw per-quarter (DB1B) and per-month (OD40) parquets are **too large for GitHub** (100 MB–1 GB each) and are not in this repo. They live in the local Dropbox data directory on the maintainer's machine; the pipeline ingests them and writes only the small aggregated outputs here.

If you need raw granularity, run the download scripts yourself from [dot-download](https://github.com/garycl/dot-download).

---

## Changelog

- **2026-04-21** — Published gross/tax-inclusive market comparable defaults, retained source-native market fields with `SourceNative` suffixes, clarified that ticket fare/yield comparability is still pending, and backfilled missing 2014Q4 DB1B Ticket cache used by market comparable fare allocation.
- **2026-04-17** — Added OD40 (DB1C) support; fixed DB1B `AvgMktFare` pax-weighting bug; regenerated 1993–2025 aggregates. Added this document.
- **2026-02-10** — Updated NPIAS hub classifications to FY25.
- See git log for earlier changes.
