# NOTES — gta6-forecast build log

Entry for make.fun `gta6-twitch-ccu-forecast-01` (forecast GTA VI launch-day
Twitch CCU, hours 2–24, five quantiles per hour). Built 18 Nov 2026 in a
fresh session started after the competition opened (17 Nov 06:00Z).
Program: single-file `main.py`, Python 3.13, stdlib only.

## Model

Model D (two-regime, smooth blend), calibrated from the case file's own
comparables at run time — nothing is hard-coded, no practice answer is
embedded anywhere:

- Usable comparable: release-day curve (hours 1–24 vs `day_start`) with a
  real hour-1 print and ≥ 15 of 24 hours. 37 of 38 in a practice case
  (the GTA VI pre-release history itself has no hour 1 and is excluded).
- Big branch: empirical quantiles of ln(viewers_h / viewers_1) per hour
  over the pool of big launches (hour-1 ≥ 100,000).
- Small branch: per-hour OLS of ln-ratio on log10(hour-1 print) over all
  usable comparables, plus empirical quantiles of the residuals.
- Blend in log space with w = sigmoid((log10(p1) − 5) / 0.40); p1 = 100k
  sits at the midpoint, p1 ≥ 1M is effectively all big-pool (saturating —
  the regression is never allowed to under-forecast a huge launch).
- Fallbacks keep every output valid: thin pool (< 5 curves) → all-pool
  quantiles; no pool at all → flat persistence with widening spread.
  Quantiles are clamped to (0, 1e9] and sorted monotone.

## Variants tested (leave-one-out, 37 pseudo-cases)

Mean pinball loss on the log scale. "Big" = pseudo-cases with hour-1
≥ 100k (n = 16) — the regime GTA VI will almost surely be in.

| Variant | Overall | Big ≥100k | Mid 25–100k | Small <25k | Worst case |
|---|---|---|---|---|---|
| Baseline (starter: flat median, σ 0.15) | 0.9297 | 0.2611 | 0.3869 | 2.3955 | 3.9411 Silksong |
| T50 (hard switch at 50k) | 0.3989 | 0.2293 | 0.3057 | 0.7303 | 1.6395 Wukong |
| T100 (hard switch at 100k) | 0.4091 | 0.2293 | 0.3435 | 0.7303 | 1.6395 Wukong |
| T150 (hard switch at 150k) | 0.4093 | 0.2296 | 0.3435 | 0.7303 | 1.6395 Wukong |
| **blend100, scale 0.40 (SHIPPED)** | **0.3828** | **0.2125** | 0.2796 | 0.7244 | 1.6929 Wukong |
| blend100, scale 0.25 | 0.3890 | 0.2133 | 0.2990 | 0.7265 | 1.6468 Wukong |
| blend100 + similarity weights | 0.4034 | 0.2160 | 0.3048 | 0.7656 | 1.6949 Wukong |
| blend100 + median cap | 0.3892 | 0.2139 | 0.2991 | 0.7260 | 1.6442 Wukong |
| blend100 + clean pool only | 0.4100 | 0.2310 | 0.3438 | 0.7306 | 1.5737 Wukong |

Scale sensitivity for the shipped blend (overall / big):
0.15 → 0.3973 / 0.2164; 0.25 → 0.3890 / 0.2133; **0.40 → 0.3828 /
0.2125**; 0.60 → 0.3858 / 0.2138; 0.80 → 0.3964 / 0.2163.
The curve is a smooth U with its bottom at 0.40, and the median at huge
first hours stays honest there (median h18 ratio at p1 = 1M: 1.11 at
0.25, 1.00 at 0.40, 0.86 at 0.60, 0.76 at 0.80) — wider scales let the
regression leak back in and under-forecast the evening, which is exactly
the failure this design exists to avoid. So 0.40 is the shipped scale:
best on both slices, at the bottom of a smooth U rather than a sharp
spike, with no under-forecast where it matters most (a first hour of
500k–2M).

Shipped model, verified by running the actual `main.py` in leave-one-out
mode (not a copy of the algorithm): overall 0.3828, big 0.2125, mid
0.2796, small 0.7244. 90% coverage 0.821, typical miss 79.5% (baseline:
0.624 coverage, 213% miss). Best pseudo-cases: ELDEN RING 0.114,
Nightreign 0.128, Black Ops 7 0.143.

## What was rejected, and why

- Hard thresholds (T50/T100/T150): all worse than the blend on overall
  and big, and the threshold choice itself is arbitrary (T50 ≈ T100 ≈
  T150 on big games). The blend removes the knife-edge.
- Similarity weights (console-only +1, rolling-midnight +0.5): worse on
  every slice. The platform/unlock strings are free text, often
  "unverified", and the weights add noise rather than signal.
- Median cap (regression median floored at big-pool median): no change
  (0.3892 vs 0.3890) — the blend already saturates, the cap never binds.
- Clean pool (drop early-access-contaminated / weak-tier comparables):
  worse (0.4100 / 0.2310). Dropping 8 curves thins the pools more than
  the contamination hurts them.
- Using `streams` / `categories_hourly` (final-only fields): null in
  every practice case, so no backtest could validate them. Left out.
- Using the GTA VI category's pre-release level as an anchor: dormant
  (median 27 viewers) vs a launch-day 100–1000× jump. Never an anchor.

## Validation of the shipped file

- 4 practice cases: output valid (23 rows, hours 2–24, 5 quantiles each,
  monotone, > 0, ≤ 1e9, ~1.5 KB per file) — checked programmatically.
- Determinism: two full runs byte-identical (`diff -r` clean).
- Runtime: 0.18 s for all 4 cases (~0.05 s/case), cap is 8 s/case.
- Practice loss: NOT measurable locally — the practice answers are held
  by the server. The numbers above are the leave-one-out backtest, not
  a practice score. Do not quote them as a practice loss.

## 5 weakest points (self-review)

1. **One final case, no second chance.** The model is validated on 37
   pseudo-cases, but the final is a single GTA VI day. If its first hour
   is far outside the observed range (e.g. > 1.5M, or < 50k because the
   recorder struggled), the curve shape is an extrapolation, not an
   interpolation. Mitigation: saturating blend + wide honest spread.
2. **Small-regime losses are large.** Worst pseudo-cases (Black Myth:
   Wukong 1.65, Marathon 1.25, ARC Raiders 0.90) are single-global-
   unlock launches that spike unpredictably. If GTA VI's first hour
   somehow lands small, the forecast will be poor. Nothing in the data
   fixes this; the spread is the only honest answer.
3. **Coverage 0.82, not 0.90.** The 5–95% band is slightly too narrow —
   the empirical pools are thin (16 big curves) and past launches
   disagree more than the quantiles admit. A wider spread would cost
   pinball at every hour; the current width is the backtest optimum,
   but it is not perfectly calibrated.
4. **The house model (superbot) is the real opponent, not the
   baseline.** It gets one shot at the final with 64k tokens of
   reasoning and no backtest — it could simply be better on this one
   day. Our edge is empirical calibration on 37 launches; there is no
   guarantee it holds for GTA VI specifically.
5. **No final-only signal is used.** `streams` (top-20 live streams,
   languages, start times) and `categories_hourly` exist only in the
   final case and could carry real signal about which regions are awake
   at the checkpoint. They were left out because they cannot be
   backtested — a deliberate, conservative choice that may leave
   accuracy on the table.
