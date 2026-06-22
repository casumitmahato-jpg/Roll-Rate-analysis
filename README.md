# Roll Rate Analysis — Freddie Mac Loan-Level Data

## Overview
12-month forward roll-rate (transition) analysis on the Freddie Mac Single-Family
Loan-Level dataset. The notebook tracks how loans move between delinquency
buckets over a 12-month horizon — i.e. of all loans that were Current (or
30/60/90 DPD) at month B, what share end up in default 12 months later — and
aggregates this into bucket-level transition rates, yearly trends, and a
macroeconomic correlation check against the resulting default rate (ODR), feeding
into a full Vasicek/ASRF PIT PD term structure.

## Why roll rates
Roll-rate analysis is a foundational building block for credit risk and
provisioning work — both **IFRS 9** (and its Indian equivalent, **Ind AS 109**)
and **CECL** ECL models, as well as Basel PD estimation. It gives a direct,
empirical read on how delinquency emerges and progresses through a portfolio
over time, independent of any model assumptions — useful both as a standalone
diagnostic and as an input into PD term-structure construction for expected
credit loss provisioning under either framework.

A static delinquency snapshot tells you the *current* state of stress in a
portfolio. A transition matrix tells you the *trajectory* — which is what
actually feeds into IFRS 9 / Ind AS 109 stage migration, PD term-structure
construction, and ECL provisioning.

## My Journey

I started with Freddie Mac's raw origination (observation) and monthly
performance data spanning 2015 to 2023. From the full population, I selected
loans that were **active as of the January 2018 snapshot date**, and applied a
**stratified random sample** on top of that, with the condition that each
selected loan had **complete performance history from Jan 2018 through Dec
2023**. I then mapped every loan ID to its monthly performance status by
reporting month — at this stage the underlying performance data ran to
**~565 million rows**. Pivoting this into a wide table (loan ID as rows,
reporting month as columns) left me with **~17 million unique loan IDs**.

While building this, I noticed a number of rows carried an `NR` (not
reported) status. My ideal approach for handling this would have been a
**rule-based interpolation**: where a single reporting month was missing
between two known values, infer the missing bucket from the surrounding
values (e.g. if the value before is 1 and the value after is 3, the missing
month is likely 2; if the next value is 0, the missing month is likely 0, and
so on). However, working on a personal laptop with limited RAM and storage,
this kind of conditional row-by-row imputation across millions of loans
wasn't feasible. Instead, I took the more conservative route and **dropped
every loan that had even a single missing reporting month**, which is the
main data-completeness limitation of this analysis (see Limitations below).
I also worked with **parquet instead of CSV** throughout, since parquet
allows reading only the columns actually needed for a given step, which
mattered a lot given the storage and RAM constraints of processing this at
scale on a laptop.

After dropping incomplete loans, I was left with **~1.3 million loans** with
fully complete monthly performance from Jan 2018 to Dec 2023. From this
clean panel, I computed month-by-month delinquency bucket counts and shares —
the resulting chart turned out to be genuinely interesting to look at,
because the COVID-19 period (Apr 2020 onward) shows a textbook roll-rate
cascade: 1-30 DPD spiked first, then 31-60 DPD, then 61-90 DPD, each peak
smaller than the last as borrowers cured or entered forbearance — followed by
early-stage buckets staying elevated even as they declined from their peaks,
which kept feeding inflow into 91+ DPD well after the initial shock had
passed.

From there, I built a **12-month forward transition table**, measuring for
each starting bucket how much of that bucket's population rolled into default
within 12 months — visualized in the corresponding graph — and then averaged
these transition rates **year by year from 2018 to 2022**.

Since my underlying loan data is U.S.-based, I sourced **U.S. macroeconomic
variables (MEVs)** — around 12 candidates, covering both index levels and
growth rates. I tested each against the yearly ODR series on two criteria:
correlation strength (**|r| > 0.50**) and a **sign test** (actual correlation
sign matching the economically expected sign). **6 MEVs passed both tests**:
`Real_GDP_Growth_Pct`, `Ind_Prod_Index`, `Ind_Prod_Growth_Pct`,
`Case_Shiller_Growth_Pct`, `FHFA_HPI_Growth_Pct`, and `Unemployment_Rate`.

I then built **three forward scenarios for 2023–2027** for each selected MEV:
**baseline** (the projected/mean MEV path), and **optimistic / pessimistic**
as **+1 / -1 standard deviation** around the baseline. Each MEV was given
**equal weight** in combining them into a single standardized **final
economic factor** per scenario per year.

This economic factor feeds into a **Vasicek/ASRF (Merton-style) model** to
convert each bucket's **through-the-cycle (TTC) PD** into a **point-in-time
(PIT) PD**, using an asset correlation (**rho = 0.15**) for every bucket.

Because the PIT PD from this conversion is a **conditional PD** — conditional
on having survived to the start of that forward year — I built a **survival
analysis** to convert these into **unconditional PDs**:
- Year 1: unconditional PD = conditional PD (no prior survival to condition on); survival after year 1 = 1 − conditional PD(year 1).
- Year 2 onward: unconditional PD(year *n*) = conditional PD(year *n*) × survival(year *n−1*); survival(year *n*) = survival(year *n−1*) − unconditional PD(year *n*).

After computing unconditional PDs for all three scenarios, I applied scenario
weights of **80% baseline / 10% optimistic / 10% pessimistic** to get a single
weighted unconditional PD per bucket per year. From this final table, I
derived:
- **Stage 1 PD** — the bucket-wise, year-wise 1-year unconditional forward PD.
- **Stage 2 PD** — the cumulative PD over the remaining life of the loan, calculated by summing all yearly unconditional PDs.

## Dataset

### Sample design
- Loans **originated between 2015 and Jan 2018**, all still active as of the
  January 2018 reporting month, tracked monthly through **Dec 2023**.
- Drawn as a **stratified random sample by origination quarter**, to preserve
  the relative vintage composition of the original population rather than
  skewing toward any single origination period.
- Final analysis population: **~1.3 million loans** with fully complete
  monthly performance data from Jan 2018 to Dec 2023, after removing any loan
  with even a single missing (`NR`) reporting month.

## Methodology
1. **Batch processing with checkpointing** — the full performance panel
   (~565M rows) is too large to load in memory, so it's processed in chunks
   via `pyarrow.parquet` batch iteration, with per-batch checkpoint/resume
   logic (parquet accumulator + progress file) to survive interruptions on
   long-running jobs.
2. **Before-transition snapshot** — month-by-month share of loans in each
   delinquency bucket, as a raw baseline diagnostic.
3. **12-month forward transition** — for each starting bucket, computes the
   12-month roll-to-default rate.
4. **Yearly aggregation** — transition rates averaged by reporting year (2018–2022).
5. **MEV correlation & selection** — candidate macroeconomic variables tested
   against the yearly ODR series on correlation strength (|r| > 0.50) and
   expected sign.
6. **Scenario construction & PIT conversion** — baseline/optimistic/pessimistic
   MEV scenarios (2023–2027) combined into a final economic factor, used to
   convert TTC PD to PIT PD via Vasicek/ASRF (rho = 0.15), then converted to
   unconditional PD via survival analysis and scenario-weighted (80/10/10)
   into final Stage 1 (1-year) and Stage 2 (cumulative lifetime) PDs.

## Findings

### 1. Before transition — raw delinquency snapshot
`dpd_rate_trend.png`

The COVID shock produced a textbook roll-rate cascade: 1-30 DPD spiked first
(Apr 2020), followed by 31-60 DPD, then 61-90 DPD — each peak smaller than the
last as borrowers cured or entered forbearance. Early-stage buckets declined
from their peaks but stayed above pre-pandemic levels for an extended period,
which kept feeding inflow into 91+ DPD even as the shock itself faded —
declining ≠ normalized. By 2022, delinquency levels broadly normalized, aided
by home-price appreciation, stimulus, and low rates acting as a credit-loss
absorption mechanism rather than a conversion into permanent losses.

### 2. After transition — 12-month roll-to-default check
`bucket_transition_trend.png`

The 12-month forward transition rate increases sharply and monotonically with
delinquency severity at the starting point. Loans starting Current show the
lowest 12-month roll-to-default rate by a wide margin; loans starting 61-90
DPD roll to default at dramatically higher rates, since that population has
already survived multiple cure opportunities without curing.

### 3. Selected MEV vs ODR
`selected_mev_vs_odr.png`

Each macroeconomic variable that passed the sign and correlation-strength
test is plotted against the yearly ODR series to visually confirm the
relationship the statistical test picked up, before being carried forward
into scenario construction.

## Output files
| File | Description |
|---|---|
| `dpd_rate_trend.png` | Delinquency bucket rates by reporting month (before transition) |
| `bucket_transition_trend.png` | 12-month roll-to-default rate by starting bucket (after transition) |
| `selected_mev_vs_odr.png` | Selected macro variables plotted against ODR |
| `roll_rate_12m_transition.parquet` | Full 12-month transition rate matrix |

## Limitations

1. **Missing-data handling.** Ideally, single missing (`NR`) reporting months
   between two known values would be imputed with a rule-based approach
   (inferring the missing bucket from the surrounding values). Due to RAM and
   storage constraints of processing this on a personal laptop, this wasn't
   feasible at scale, so any loan with even one missing reporting month was
   dropped entirely — reducing the usable population from ~17 million to
   ~1.3 million loans. This is a completeness trade-off, not a methodological
   choice, and a more capable compute environment would likely recover a
   meaningfully larger and possibly differently-composed sample.

2. **Short ODR/MEV correlation window.** Storage constraints limited the ODR
   series used for MEV correlation and selection to **5 years (2018–2022)**.
   A longer historical window could change both the correlation strength and
   sign for some variables, and potentially the final set of selected MEVs.

3. **Retained correlated MEV pairs.** Two pairs among the selected MEVs are
   conceptually overlapping — `Ind_Prod_Index` / `Ind_Prod_Growth_Pct`, and
   `Case_Shiller_Growth_Pct` / `FHFA_HPI_Growth_Pct`. One variable from each
   pair could reasonably have been dropped to reduce multicollinearity, but
   both were retained since each independently passed the sign and
   correlation-strength tests.

## Tech stack
Python, pandas, pyarrow (batch parquet processing), NumPy, SciPy, Matplotlib

## Notes
Built as part of a broader ECL provisioning pipeline aligned with IFRS 9 /
Ind AS 109 principles — roll rates feed into PD term-structure construction
via Vasicek/ASRF TTC-to-PIT conversion. Also relevant to RBI's upcoming
ECL-based provisioning framework (effective April 2027), which will replace
the current IRAC norms for Indian banks and NBFCs.
