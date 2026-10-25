# Project Log: Customs Risk Predictive Project

## 2026-10-11: Data cleaning (raw 5,250 rows -> clean 4,746 rows)

Raw file: customs_clearance_data.csv (5,250 rows x 26 columns), never modified.
Cleaned file: customs_clean.csv (4,746 rows x 28 columns after step 10). Load with:
`pd.read_csv("customs_clean.csv", parse_dates=["entry_date", "accepted_date", "release_date"], dtype={"hs_code": str})`

### Cleaning steps

| # | Action | Result |
|---|---|---|
| 1 | Load raw, work on `df_clean = df.copy()` | 5,250 x 26 |
| 2 | Strip spaces on text columns | `port_of_entry` had leading spaces; now 12 distinct ports |
| 3 | Standardize case | `broker` had upper and lower variants: 12 -> 6 brokers. Did not title-case other columns (would damage USA/UAE/DRC) |
| 4 | Duplicates (after steps 2-3) | 0 found; `shipment_id` unique. The earlier project notes described 5% duplicates; this file has none |
| 5 | HS code: strip trailing "X" | 95 codes had 7 characters; repaired. All codes now 6 digits |
| 6 | Drop null `hs_code` / `country_origin` | 5,250 -> 4,840 (410 dropped; 8 rows were missing both) |
| 7 | Convert date columns to datetime | Done; range Sep 2025 - Sep 2026 |
| 8 | Weight artifact | 94 rows (1.9%) held one identical value (~130,837 kg); nothing between ~4,900 and it. Dropped with rule `weight_kg > 10,000`: 4,840 -> 4,746. Max weight now 4,999.99 |
| 9 | Release before acceptance | 179 rows. `release_date` set to NaT, rows kept. (Status: confirm applied and re-saved) |
| 10 | Derive `hs_chapter`, `hs_section` | 10 chapters, 8 of 21 sections present (VI, VII, XI, XIII, XV, XVI, XVII, XX). File now 4,746 x 28 |

### Evidence and reasoning
- country_origin missing: 24.8% slow vs 25.8% present, so missingness is unrelated to the target.
- Weight rows: dropped because the value is not a real measurement; loss is small.
- Release-before-acceptance rows: gaps are 1-5 whole hours; slow rate 15.6% vs ~26% overall, so
  not random. Dropping them would bias the data, so only `release_date` was blanked.
- Date order: 0 rows with accepted < entry.
- Chapters match products (30 pharmaceuticals, 39/40 plastics and rubber, 62 textiles, 70 glass,
  73 metals, 84/85 machinery and electrical, 87 vehicles, 94 furniture), so HS section is valid.

### Decisions kept as-is
- `MOMBASA` and `MOMBASA ICD` kept separate (seaport vs inland container depot).
- `entry_no` (333 repeats) and `container_number` (249 repeats) treated as legitimate.
- `NONE` in `fta_claimed` is a real category, not missing.
- Section is a coarser grouping of `product_desc` (49 products -> 8 sections); slow rate by
  section ranges ~20-28%, not yet tested for significance.

### Key findings
- Class balance: about 26% slow (Y), 74% not slow (N). Accuracy is a misleading metric here.
- LEAKAGE: `slow_clearance` is essentially "released more than 48 hours after entry"
  (N: 0-47 h, mean 23.5; Y: mean 82 h, max 120). `release_date` and anything derived from it
  are excluded from all feature sets.
- No `high_discrepancy` column exists; derive it from `declared_value_usd` vs `invoice_value_usd`.
- Licenses: 50 shipments required a license but did not obtain one; 0 obtained one they did not need.
- `restricted_party_pass`: 46 N (~1%), a rare class.
- 12 suppliers, no spelling variants; ChemCorp Industries and PlastiPack Ltd intact.

### Limitations
- Rows with missing HS code, origin or weight are removed, so a model never sees those cases.
- Cleaning steps are specific to this file; new files need inspection first.

## 2026-10-12: Feature and leakage decisions

Scoring moment: the declaration is lodged, before processing. Assumption: the data represents a
large e-commerce importer risk-profiling its own shipments.

- Leakage: `release_date` (the label is a >48h threshold on it); `accepted_date` (follows
  lodgement; AEO entries can be accepted and released together; entries can be rejected).
- Leakage by derivation: `duty_amount_usd` and `tax_amount_usd` are computed from value, probably
  invoice value (invoice-based rate had smaller spread: strong by hs_code, weak by product_desc).
  Excluded from both models.
- `hs_code`: 4,418 distinct values in 4,746 rows; first two digits map 1:1 to `product_desc`.
  Raw code not used. Use `product_desc` and `hs_section`.
- Each target has its own feature list: `invoice_value_usd` is excluded from the valuation model.
- `license_required` and `license_obtained` combine into one flag; assumes the licence is declared at entry.
- `broker_fee_usd` agreed before lodgement (weak feature). `restricted_party_pass` screened beforehand.
- Identifiers excluded; `entry_no` kept to group the train/test split.

### Open items
- Derive `high_discrepancy` and time features (RP1)
- CONVEYOR BELTS (ch 40) vs CONVEYOR BELT (ch 84): confirm distinct products (RP1)
- Slow rate of the dropped hs_code and weight rows (optional)
- Significance test of section differences (RP4)

## 2026-10-23: Plan for the course and project

Saved `Customs_Return_Point_Plan.md` in the project (return points RP1 to RP15 mapped to the
Portilla course order). Next return: RP1, after the Pandas section and its exercises.

## Concepts learned

- **Class imbalance:** Only about 26% of shipments are slow, and about 18% have a valuation
  discrepancy. A model that always says "not slow" is right 74% of the time and catches nothing,
  so accuracy is misleading. I judge models by precision, recall and AUPRC, and compare them with
  the base rate (a random model scores about 0.26 on AUPRC for delay), not with zero.

- **Data leakage:** Leakage is giving a model information it would not have when it makes the
  prediction. I found three kinds in this project. `slow_clearance` is defined by release time, so
  `release_date` is the label in disguise. `accepted_date` happens after lodgement and is tied to
  the outcome. Duty and tax are calculated from the invoice value, so they would leak the
  valuation target. A model with leaked columns scores very well in testing and fails in real
  use. I check every column by asking whether I would know it when the declaration is lodged.

- **Missing data is not always random:** Before dropping rows I compared the slow rate of
  missing and non-missing groups (24.8% vs 25.8% for country of origin). Similar rates mean that
  dropping the rows is safe. Different rates would mean the missingness carries information.

- **Artifacts vs outliers:** 94 weights shared one identical value. Real heavy cargo varies, so
  that was a data-entry artifact and not a genuine extreme. I removed it. A real outlier would
  need a different decision.

- **Derived features and their risks:** I built `license_missing` and `hs_section` from existing
  columns. A derived feature adds no new information, but it can make the signal clearer. It
  must be built only from columns known at the scoring moment.

- **Train/test split on time-ordered data (to confirm at RP6):** Shipments have dates, so the
  model should train on earlier shipments and be tested on later ones, as in real use. Shipments
  sharing an entry number must stay on the same side of the split.

- Time features from entry_date: entry_day_of_week (0=Mon), entry_month, entry_is_weekend (Sat/Sun).
- Used entry_date, not accepted_date: accepted_date follows lodgement, so it is not known at the
  scoring moment. The entry-to-acceptance gap is mostly importer/broker delay; kept for descriptive
  analysis (RP2), not as a feature. Definition of slow clearance unchanged.
- Entries spread evenly across days (13.4% to 15.3% each) and months (364 to 424 rows).
- Weekend: 1,290 rows (27.2%). Slow rate 30.3% vs 24.5% on weekdays (overall 26.1%).
  Saturday 30.9% and Sunday 29.7%, both above average. Weekdays 22.5% to 26.9%, probably noise.
- Month: slow rate 22.8% to 29.1%, no seasonal pattern at ~400 rows per month.
- Conclusion: entry_is_weekend is the useful time feature; day of week and month likely weak.
  Formal tests at RP4.

- license_missing: Y when license_required = Y and license_obtained = N. 50 rows (1.05%).
  Built two ways (apply function and boolean expression); identical results.
- Slow rate: 48.0% (24/50) when missing vs 25.8% otherwise. Strong but rare; small group means a
  wide margin of error (about ±12 points). Formal test at RP4.
- First attempt flagged 4,624 rows because the else branch caught shipments needing no licence.
  Fixed by flagging the specific risky case in the if. Lesson: put the case you care about in the
  if, not the safe cases.


  - high_discrepancy: derived from gap_pct = (invoice - declared) / invoice. Distribution has two
  clear groups: 3,890 rows between -5% and +5%, and 856 rows between 20% and 50%, with nothing
  between 5% and 20%. Threshold 10% (middle of the empty zone); counts identical for thresholds
  of 5%, 10%, 15% and 20%. Result: 856 Y (18.0%), 3,890 N.
- Negative gaps (over-declaration, 87 rows below -5%) treated as N: no revenue risk.
- gap_pct is part of the target definition and is excluded from all feature lists.

## 2026-10-25: RP1, feature engineering (customs_clean.csv -> customs_features.csv)

| Step | Action | Result |
|---|---|---|
| 1 | Load customs_clean.csv | (4746, 28); release_date 279 nulls (100 original + 179 blanked) |
| 2 | Time features from entry_date | entry_day_of_week, entry_month, entry_is_weekend. Weekend 1,290 rows (27.2%); 30.3% slow vs 24.5% weekday. Sat 30.9%, Sun 29.7%. Month 22.8% to 29.1%, no seasonal pattern |
| 3 | license_missing | 50 rows (1.05%); 48.0% slow vs 25.8%. First attempt flagged 4,624 rows (else branch caught shipments needing no licence); fixed by flagging the risky case in the if |
| 4 | high_discrepancy | gap_pct = (invoice - declared) / invoice. Two groups: 3,890 rows within -5% to +5%, 856 rows at 20% to 50%, nothing between. Threshold 10%; counts identical at 5/10/15/20%. 856 Y (18.0%), 3,890 N |
| 5 | CONVEYOR BELT(S) | Ch 84 vs ch 40, no overlap, classified separately in the real HS. Kept separate |
| 6 | Merged section names | 8 sections, 0 nulls |
| 7 | Group tables, Excel export | customs_summary.xlsx |
| 8 | Saved customs_features.csv | (4746, 35); reload verified |

Rules: gap_pct and the definition of high_discrepancy are excluded from every feature list.
accepted_date not used for time features: it follows lodgement. Definition of slow clearance unchanged.

Observations from group tables:
- Port is a strong driver of slow clearance: Alexandria, Mombasa, Lagos, Casablanca at 30-34% (about
  31% combined, 2,864 rows) vs 13-21% for the other eight (about 18%). Worth modeling.
- Port shows no clear pattern for discrepancy (14% to 23%); highest values on ~230 rows. Probably noise.
- Mombasa 30.7% vs Mombasa ICD 20.8%: supports keeping them separate.
- ___ (broker, section, origin: add yours)

Open: formal significance tests (RP4); weekend/month/licence effects to confirm.