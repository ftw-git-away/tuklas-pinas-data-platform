

Consolidated data sources · MD
# Tuklas Pinas — Data Pipeline Readiness Guide
 
A quick way to use this doc: it answers three questions, in order —
**what are we using, what can we ingest right now, and what decisions still need to happen before transformation and analysis.**
 
Finalized business and analytical questions live in `docs/business-and-analytical-questions.md`. That's the single source of truth for BQs/AQs — this document doesn't repeat them.
 
---
 
## Status at a Glance
 
- **Source selection:** substantially complete — all five candidate categories (PSA, DOT, PSGC, population, GDP/GRDP) have at least one reviewed, usable source
- **Bronze ingestion:** ready to start now
- **Silver and Gold decisions:** tracked below, but none of them block Bronze
---
 
## Source Matrix
 
One row per dataset. "Bronze Ready?" means it can be loaded today, unchanged. "Silver Blocker" names the one decision that has to happen before this source moves past raw ingestion.
 
| Source | Role | Years | Raw Grain | Status | Bronze Ready? | Silver Blocker |
|---|---|---|---|---|---|---|
| PSGC 2Q 2026 National and Provincial Summary | Validation / structural-integrity reference | current (Q2 2026) | Region→Province | Reviewed | Yes (reference only) | None |
| PSGC 2Q 2026 Publication Datafile | Master geography lookup; carries population counts | current (Q2 2026) | Region→Province→City/Mun→Brgy | Reviewed | Yes | Crosswalk rules need applying |
| Table B (Population + PGR) | Primary population anchor | 2010, 2015, 2020, 2024 | Region/Province/City | Reviewed | Yes | Doesn't cover 2019/2022/2023 — pending denominator method |
| Mid-year population (2015-based projections) | Candidate annual population source | 2020–2025 | Region/Province/City | Reviewed | Yes | Old 2015-census base + pre-NIR labels; doesn't cover 2019 |
| DOT Overnight Travelers | Primary tourism-activity source | 2000–2024 | Region/Province/City, increasingly municipality from 2008 | Opened and tested | Yes (ingest as-is) | Dedup rules, missing-data rules, geography normalization |
| Table 3.63 (Accommodation & Food Service GVA, region) | Region-level tourism economic reference | 2022–2024 | Region | Reviewed | Yes | None (QA use only) |
| Table 3.72/73 (GDP, province) | Provincial economic output | 2019–2023 | Province | Reviewed | Yes | Province/HUC handling |
| Table 3.74/75 (Per capita GDP) | Pre-computed per-capita economic figure | 2019–2023 | Province/HUC | Reviewed | Yes | Province/HUC handling |
| Table 3.94/95 (Accommodation & Food GVA) | Tourism economic proxy | 2019–2023 | Province/HUC | Reviewed | Yes | Province/HUC handling |
| PTSA tables | National tourism-economy context | varies | National | Reviewed | Yes | None (not used at province grain) |
| Tourism classification system | Reference / definitions | 2025 | N/A | Reviewed | N/A (reference doc) | None |
 
**Out of scope, not going into Bronze:** Table 1/2/3/5/6 (population — superseded by Table B or national-only), DOH-EB barangay population (wrong grain + provenance concern), Hotel Occupancy NCR (single region), Outbound tourism tables (wrong direction), single-month Inbound Update (too narrow).
 
---
 
## How to Load Bronze
 
The rule: **land every source exactly as published, with zero interpretation.** Every cleaning, joining, or aggregation decision happens later, explicitly and documented — never implicitly during load.
 
- **Preserve original values as published** — no unit conversion, no recalculation
- **Preserve original geography labels/codes exactly as given** — even where a fix is already known (e.g., a "Boracay" row stays "Boracay," not remapped to Malay, until Silver)
- **Preserve nulls as nulls** — an empty cell stays empty, never filled in
- **Load HUCs and provinces as separate rows**, exactly as each source presents them
- **No population interpolation** — load exactly the years Table B has (2010, 2015, 2020, 2024), nothing invented
- **No PSGC remapping** — known fixes are documented but not applied at this stage; Bronze reflects exactly what was downloaded
- **Never convert missing values to zero** — a blank stays a blank
Following these means every downstream decision can be made, or changed, later — without needing to re-pull or re-load anything.
 
---
 
## How to Label Overnight Travelers
 
These values are not unique-visitor counts and are not interchangeable with border "arrivals." 2024 foreign travelers in this source total 7.63M (DOT 2024 regional travelers file), compared to 5.44M in PSY Table 8.1 and 5.93M in the DOT 2024 citizenship table.
 
The specific reason for that gap is not confirmed. DOT does not publish how travelers are tallied, so "repeat check-ins" should not be asserted as a confirmed mechanism. The field should be treated as an activity/volume measure, not a headcount.
 
---
 
## Decisions Required Before Silver
 
None of these block Bronze — ingestion goes ahead regardless of where these land.
 
### Geography
 
Keep province and HUC as **separate canonical entities in Silver**. Only combine them into one total in Gold, and only if a specific BQ needs that combined figure.
 
**Geography dimension fields:**
- `geography_code`
- `geography_name`
- `geography_type`
- `parent_province_code`
- `region_code`
NCR has no provinces of its own, so it's handled explicitly under this structure rather than forced into a province shape.
 
**PSGC crosswalk fixes to apply:** Negros/NIR, Sulu, Cotabato City, Boracay, Zamboanga City, Makati/Taguig, Isabela City — exact codes are in the DQ Appendix.
 
### Population Denominators
 
Select one method to fill **2019, 2022, and 2023**:
- Interpolation from census counts
- Back-calculating from GDP ÷ per-capita GDP
- PSA's mid-year population releases
No candidate source currently covers 2019 — that year needs interpolation or the GDP ÷ per-capita GDP method, regardless of which approach gets chosen for 2022/2023.
 
### DOT Cleaning Rules
 
- Sub-row deduplication (e.g., Boracay nested inside Aklan)
- Missing-vs-zero distinction: a blank stays "unavailable," never becomes a true zero
---
 
## Decisions Required Before Gold
 
- Recovery/emerging-destination thresholds
- BQ calculation validation
- Final exclusions and comparability flags (BARMM, Boracay/Aklan pre/post-2021)
**Comparison periods to use once Silver is ready:**
- Common source window: 2019–2023
- BQ2: 2019 vs. 2022, and 2019 vs. 2023 — calculated separately
- BQ3: 2019 vs. 2023 only
- Table 3.63 (2022–2024) stays validation-only, never a direct BQ input
---
 
## Data Quality Appendix
 
Reference material — not needed to start Bronze.
 
| Issue | Affected Records | Impact | Action | Status |
|---|---|---|---|---|
| Overnight Travelers not unique-visitor counts | All DOT traveler figures | Totals exceed independent arrival counts in some years | Label as activity volume, not headcount | Open |
| Sub-row double-counting | Boracay (in Aklan), Legazpi (in Albay), others | Naive summation double-counts province totals | Build dedup rule for Silver | Open |
| Coverage-driven pseudo-growth | Multi-year national/regional totals | Growth partly reflects more places reporting over time | Flag caveat wherever multi-year trends are shown | Open |
| Boracay/Aklan reporting change | Aklan province, 2019–2024 | Boracay switched to Caticlan jetty counting starting 2021 (not 2022). In 2019–2020, Boracay is its own row and Aklan's row excludes it, so Aklan + Boracay is the correct total for those years. From 2021 onward, Boracay is already included in Aklan's row, so adding them double-counts. | Decide: combine as one unit using the correct year split (sum only for 2019–2020), or exclude Boracay pre-2021 | Open |
| Region II Foreign/OFW lumping | Region II travel-type splits | Only appears in the 2000–2007 and 2011–2013 files, outside the 2019–2023 core window | Not a current blocker; revisit if those years are ever used | Deferred |
| Region IV-A/IV-B split (2008) | Region IV provinces | Boundary change predates the 2019–2023 core window | Not a current blocker; revisit if pre-2019 years are ever used | Deferred |
| Sulu region reclassification | Sulu province | PSGC (Region IX) vs. historical BARMM tagging | Standardize to PSGC (`0906600000`) | Identified, not yet applied |
| Cotabato City region conflict | Cotabato City | PSGC (BARMM) vs. DOT (Region XII in 2019 / blank "-" in 2020–2022 / absent 2023–2024) | Remap to BARMM (`1908703000`); treat the 2020–2022 blanks per the missing-vs-zero rule, not as zero | Identified, not yet applied |
| Boracay PSGC mapping | Boracay rows | Reported as standalone, not its real municipality | Map to Malay, Aklan (`0600412000`) | Identified, not yet applied |
| Zamboanga City standalone row | Zamboanga City | Basilan absent from same files | Map directly to `0931700000` | Identified, not yet applied |
| Makati/Taguig EMBO boundary shift | Makati City, Taguig City (NCR) | Population shift (G.R. 235316) vs. historical GDP coverage | Document variance; roll up to NCR level | Identified, not yet applied |
| Isabela City naming collision | Isabela City / province / Negros Occidental municipality | Name-only matching risks misfiling | Match by PSGC code only | Identified, not yet applied |
| Isabela City value anomaly | Isabela City, 2019–2024 | Domestic count jumps 12,440 (2022) → 101,238 (2023) → 464,561 (2024), coinciding exactly with the label change to "Isabela City de Basilan"; also, the 2021 domestic count (15,155) is identical to 2019 — the same repeated-value pattern seen in Zamboanga Sibugay | Spot-check against source PDF before use in any calculation | Open |
| HUC vs. province double-counting | Per-capita GDP, GDP tables (3.72/73), tourism GVA tables, and Table B | Parent rows footnote "Excludes HUC" | Keep HUCs separate with `parent_province_code` | Identified, not yet applied |
| Suspicious province-year values (outside core window) | Quezon 2000, Cavite 2001/2010, Laguna 2002, Mindoro Oriental 2005–07, Mountain Province 2000 | Implausible, but outside 2019–2023 | Not a current blocker | Deferred |
| Zamboanga Sibugay anomaly (within core window) | Zamboanga Sibugay, 2023→2024 | Flat then ~22x jump | Spot-check against source PDF | Open |
 
---
 
## Ingestion Readiness
 
**Ready for Bronze — start now:**
- DOT Overnight Travelers
- PSGC
- Provincial GDP
- GDP per capita
- Accommodation & Food Service GVA
- Population source files
**Required before Silver:** implement province/HUC/NCR handling, PSGC crosswalk application, annual population denominator method, DOT deduplication/missing-data rules
 
**Required before Gold:** recovery/emerging thresholds, BQ calculation validation, final exclusions/comparability flags