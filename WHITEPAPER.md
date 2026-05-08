# The Baseline Biological Age Algorithm

**A transparent, biology-aware approach to consumer biological age estimation**

*Version 1.0 · May 2026*

---

## Abstract

Consumer biological age scores have a credibility problem. They move every day in response to noise that has nothing to do with biology, they hide their math behind proprietary weights, and they present single point estimates with no acknowledgment of uncertainty.

Baseline takes a different approach. Our biological age is computed from a deliberately layered architecture: raw sensor data is never used directly in composite scores, every metric is smoothed against a half-life matched to its underlying biology, hysteresis prevents noise-driven jitter, and every score ships with an explicit confidence interval that widens when inputs are stale or missing. The algorithm is fully open — published here, in code, with weights and thresholds visible — so users, clinicians, and researchers can audit the math instead of trusting a black box.

This paper documents the algorithm in full. It is not a research validation paper. It is a transparency document.

---

## 1. Why we wrote this

Three observations motivated this work.

**First, daily-recomputing biological age is mathematically dishonest.** A bio age that moves 0.4 years because someone slept badly Tuesday is reporting a change in the *estimate*, not in the underlying biology. Sleep does not age a body overnight. Conflating the two erodes user trust.

**Second, opacity is the norm.** A 2025 review of composite wearable scores found that recovery scores from major brands can differ by twenty points or more for the same night, with weights and thresholds undocumented. Users cannot reason about what they are seeing.

**Third, point estimates without uncertainty are wrong.** A bio age of 38.2 reported with no confidence band implies a precision the underlying data does not support — particularly when key inputs (a recent blood panel, last night's sleep) are missing.

Baseline addresses all three.

---

## 2. The four-layer architecture

The system separates raw observations from displayed values through four explicit layers. This separation is not cosmetic — it is the core design decision.

| Layer | What it contains | What it never does |
|---|---|---|
| **Layer 1 — Raw store** | Every observation from every source, with timestamp and provenance | Touched by display logic |
| **Layer 2 — Smoothed state** | EWMA-smoothed values per metric, with biology-matched half-lives | Recomputed without history |
| **Layer 3 — Composite scores** | Pillar scores, overall vitals, biological age, all with confidence intervals | Computed from Layer 1 directly |
| **Layer 4 — Display** | What the user sees on their dashboard | Updated on every sync |

The user-facing biological age number is a Layer 3 value. It is mathematically impossible for a single bad night to move it more than a fraction of a year, because the bad night first has to move a Layer 2 EWMA, which is anchored against weeks of prior data.

---

## 3. Layer 2: Smoothing with biology-matched half-lives

Every metric is smoothed using an exponentially weighted moving average. The weight of an observation taken `d` days ago is:

```
w(d) = exp(-ln(2) · d / h)
```

where `h` is the half-life in days. An observation `h` days old contributes half the weight of today's observation; one `2h` days old contributes a quarter, and so on.

The half-life for each metric is chosen to match how fast the underlying biology actually changes. This is the single most important parameter set in the system, and it is the reason the algorithm does not drift on noise.

### Half-life configuration

| Metric group | Half-life | Rationale |
|---|---|---|
| HRV, resting heart rate, sleep score, sleep duration, body temperature deviation | 7 days | Autonomic nervous system metrics shift on a weekly rhythm. A single night is noise; a week is signal. |
| Indoor air quality | 3 days | Environment changes faster than physiology. We want to catch a poorly-ventilated week, not last winter's CO₂. |
| Body fat percentage, blood pressure, weight stability | 14 days | Real tissue change is measurable in fortnights, not days. |
| Acute:chronic workload ratio | 14 days | Load balance shifts on the order of a training block. |
| Training consistency, weekly volume, intensity distribution | 30 days | Aerobic adaptation requires weeks. |
| Lab values (ApoB, LDL, HDL, triglycerides, HbA1c, hsCRP, ferritin, Vitamin D) | No smoothing | Discrete measurements with their own date stamp. Smoothing would invent data between draws. |

These values are configured in a single file (`smoothing-config.ts`), reviewed at every algorithm version, and exposed through the open-source repository so any user or researcher can verify them.

### Why EWMA, not a simple rolling average

A simple rolling average treats a 30-day-old observation identically to today's. EWMA decays continuously, which has three properties we want:

1. **No edge effects.** A simple window introduces a discontinuity when an old observation falls out. EWMA decays smoothly.
2. **No tuning of window length separate from half-life.** The half-life is the only parameter; observations contribute forever, with diminishing weight.
3. **Robust to gaps.** When data is missing for several days, the EWMA continues to use what it has, weighted by recency. A simple window would either silently include very stale data or refuse to compute.

---

## 4. Layer 3: From smoothed metrics to pillar scores

Baseline composes biological age from three pillars, each scored 0–100 from a weighted blend of smoothed metrics.

### 4.1 Rest pillar (35% of overall vitals, ±2.5 years on bio age)

| Sub-metric | Weight | Source | Scoring approach |
|---|---|---|---|
| HRV | 0.27 | Oura | Compared against personal 90-day baseline. ±0% from baseline → 75; +20% → 100; -20% → 50. |
| Resting heart rate | 0.18 | Oura | Compared against personal 90-day baseline. Lower than baseline scores higher; each beat above baseline subtracts 5 points. |
| Sleep score | 0.22 | Oura | Used as-is (Oura's own 0–100 scale), then EWMA-smoothed. |
| Nights with ≥7h sleep | 0.13 | Oura | Frequency over the EWMA window. |
| Body temperature deviation | 0.07 | Oura | Absolute deviation from baseline; ≤0.2°C → 100, ≥1.0°C → 0. |
| Indoor air quality | 0.08 | Airthings | Composite of CO₂, humidity, VOCs, PM2.5, radon — each scored against published health thresholds. |

Personal baselines, not population norms. This matters: a 65 ms HRV is excellent for one person and unremarkable for another. Comparing each user against themselves removes the inter-individual variance that makes population-based scores noisy.

### 4.2 Engine pillar (30% of overall vitals, ±4.5 years on bio age — the largest single contributor)

| Sub-metric | Weight | Source | Scoring approach |
|---|---|---|---|
| Consistency | 0.30 | Strava | Weeks (out of last 4) with ≥3 activities. |
| Acute:chronic workload ratio | 0.30 | Strava | 7-day load divided by 28-day average. Sweet spot 0.8–1.5 → 100; outside that range, score decays. |
| Weekly volume | 0.20 | Strava | Recent 4 weeks compared to prior 8-week trend. |
| Intensity distribution | 0.20 | Strava | Percent of sessions in zone 4+ (high intensity). 10–25% → 100; above or below decays. |

Engine carries the largest bio-age weighting because aerobic fitness has the strongest evidence base as a determinant of all-cause mortality and healthspan. We weight consistency above raw volume because regular moderate training outperforms sporadic heroics for longevity outcomes.

### 4.3 Inside pillar (35% of overall vitals, ±3.0 years on bio age)

The metabolic pillar uses lab-grade biomarkers, weighted by their evidence strength as drivers of cardiovascular and metabolic risk:

| Marker | Weight | Lower threshold (score = 100) | Upper threshold (score = 0) |
|---|---|---|---|
| ApoB | 0.25 | ≤ 0.7 g/L | ≥ 1.3 g/L |
| LDL-C | 0.20 | ≤ 1.8 mmol/L | ≥ 4.5 mmol/L |
| HDL | 0.10 | ≥ 1.6 mmol/L | ≤ 0.9 mmol/L |
| Triglycerides | 0.10 | ≤ 0.8 mmol/L | ≥ 2.3 mmol/L |
| HbA1c | 0.15 | ≤ 32 mmol/mol | ≥ 48 mmol/mol |
| hsCRP | 0.10 | ≤ 0.5 mg/L | ≥ 3.0 mg/L |
| Ferritin | 0.05 | 50–200 µg/L window | < 15 or > 400 µg/L |
| Vitamin D | 0.05 | 75–150 nmol/L window | < 25 or > 250 nmol/L |

ApoB is weighted highest because of its direct mechanistic role in atherosclerotic cardiovascular disease, the leading cause of premature mortality. The marker is increasingly recommended over LDL-C in clinical lipid guidelines.

**Trend modifier.** Each marker's score is nudged ±5 points based on the direction of change across the user's last three panels. Improving markers count, not just current levels — a user trending downward on ApoB from 1.4 to 1.1 to 0.9 g/L is in materially different shape than one stable at 0.9.

**Body composition supplement.** When Withings is connected, body fat percentage, blood pressure, and weight stability (coefficient of variation over 28 days) are added with combined weight 0.25, with blood marker weights proportionally rescaled.

### 4.4 The pillar weights

| Pillar | Vitals weight | Bio age range |
|---|---|---|
| Rest | 35% | ±2.5 years |
| Engine | 30% | ±4.5 years |
| Inside | 35% | ±3.0 years |

These two weight sets are deliberately different. Vitals is a daily readiness composite; bio age is a slow biological estimate. Engine carries more weight on bio age because cardiovascular fitness has the strongest published evidence as a determinant of biological age, while Rest and Inside pull more weight on the daily vitals score because they reflect short-term modifiable state.

---

## 5. From pillar scores to biological age

Each pillar score `s` (0–100) maps to a years-delta against calendar age:

```
yearsDelta(s, range) = -clamp((s - 75) / 25, -1, 1) · range
```

A score of 75 is neutral (zero years delta). A score of 100 subtracts the pillar's full range (e.g., -4.5 years for Engine). A score of 50 adds the full range. Scores between are linearly interpolated and clamped at ±1.

The biological age is then:

```
bioAge = calendarAge + Σ yearsDelta(pillar)
```

When a pillar is missing data, the remaining pillars are scaled up proportionally — but only by a factor of 1.15, so a single-pillar bio age cannot swing the full ±10 years of the complete model. This caps the influence of partial data.

The total range is bounded at ±10 years from calendar age. We chose this bound deliberately: published biological age clocks based on blood biomarkers (Levine PhenoAge, GrimAge) report effect sizes within roughly this range for the general population. Allowing larger deltas would imply a precision the inputs do not support.

---

## 6. Hysteresis: why the score does not jitter

Even with smoothing, composite scores can drift slightly day-to-day as new observations enter the EWMA window. For displayed bio age, we apply a hysteresis gate: the displayed value updates only when the new computed value differs from the displayed value by more than one standard deviation of the score's 30-day variance.

This is implemented in `smoothing.ts` as:

```typescript
function applyHysteresis(newScore, displayedScore, variance30) {
  if (variance30 === null || variance30 <= 0) return newScore;
  const stddev = Math.sqrt(variance30);
  if (Math.abs(newScore - displayedScore) > stddev) return newScore;
  return displayedScore;
}
```

The effect is that the displayed bio age stays still on noise and only moves when the change exceeds the noise floor. This is the single feature most directly responsible for users perceiving the score as trustworthy.

We apply hysteresis selectively. Daily vitals scores update freely — that is what they are for. Slow composites like bio age get the gate. Different scores have different update policies because they answer different questions.

---

## 7. Confidence intervals: making uncertainty visible

Every pillar score and the overall bio age ships with a 95% confidence interval. The interval widens with three independent factors:

1. **Raw input variance.** A user with high day-to-day variance in HRV gets a wider CI than one with stable HRV.
2. **Staleness.** For each day a primary input is more than 48 hours old, the CI widens by 25%.
3. **Missing inputs.** The CI widens proportionally to the fraction of inputs that are missing — at maximum, the CI doubles.

The half-width is then:

```
halfWidth = 1.96 · √variance · (1 + 0.25 · staleExcess) · (1 + missingRatio)
```

capped at 30 score-points to prevent degenerate cases.

Users see this as a visual band around their score. A user who has connected everything and synced today sees a tight band; a user with no recent blood panel and a week-old Oura sync sees a wide one. The band is honest about what the data supports. We believe this is the single biggest information-design upgrade over single-number reporting.

---

## 8. Backtesting and validation

Every algorithm change is validated against historical user data before deployment. The backtest harness (`scoring-backtest.functions.ts`) replays the last 90 days of a user's data day-by-day, computing what each score would have been on each historical day under the proposed algorithm.

We evaluate three criteria:

- **Day-to-day variance.** Day-over-day score changes should be small (within the noise floor) on uneventful days.
- **Responsiveness to real events.** Known events — illness, travel, hard training blocks — should produce visible, directionally correct changes.
- **Sensitivity per input.** No single input should be capable of moving the bio age by more than its pillar's allotted range.

A change that fails any of these criteria does not ship. Users can run the backtest on their own data from the in-app debug view; the harness is part of the open-source release.

---

## 9. What this is not

We want to be precise about the limits.

**This is not an epigenetic clock.** It does not measure DNA methylation, telomere length, or cellular markers of aging. Those are research-grade measurements requiring laboratory work that consumer products cannot provide.

**This is not validated against mortality.** Levine PhenoAge, GrimAge, and similar published clocks have been validated against decades of follow-up data in cohort studies. Baseline's bio age is informed by those clocks' biomarker selections and effect sizes, but has not itself been longitudinally validated.

**This is not a clinical diagnosis.** It is a synthesis of consumer data, intended to make patterns visible. Any flagged marker should be discussed with a physician.

**This is a directional, transparent estimate** of biological age built from inputs the user can collect at home or with a single blood draw. It is designed to be useful, honest, and auditable. Within those bounds, we believe it is more rigorous than any closed-source consumer alternative — not because the math is more complex, but because it is visible.

---

## 10. Open source and roadmap

The algorithm in this paper is implemented in TypeScript and released under the Apache 2.0 license at:

> **github.com/baseline-technologies/bio-age**

The repository contains the complete scoring engine, the smoothing module, the backtest harness, and all configuration files referenced above. We welcome issues, pull requests, and forks.

Planned improvements for subsequent versions:

- **CGM integration.** Once continuous glucose data is available through Dexcom and partner APIs, we will add glucose variability and time-in-range as Inside-pillar metrics. CGM data has the strongest known short-term link to metabolic health of any consumer-collectable signal.
- **VO₂max from running data.** A direct estimate from heart rate and pace, replacing the current proxy via consistency and intensity.
- **Apple Health VO₂max ingestion.** Native VO₂max from Apple Watch when present.
- **Per-user weight calibration.** Using each user's own pillar variance to learn personal weights, rather than fixed cohort weights.
- **External validation cohort.** Partnership with a research group to compare Baseline's bio-age trajectory against an established epigenetic clock in a small longitudinal cohort.

---

## 11. Acknowledgments

The marker thresholds in section 4.3 draw on guidelines from the European Atherosclerosis Society (ApoB, LDL), the American Diabetes Association (HbA1c), and the Endocrine Society (Vitamin D). The acute:chronic workload framework follows the Tim Gabbett model widely used in sports science. The pillar weighting and biological-age-as-delta framing are informed by Levine PhenoAge and the Horvath family of epigenetic clocks, even though Baseline's inputs differ.

We acknowledge the precedent of Oura's per-user-baseline approach to readiness scoring. Where Baseline differs is in publishing the math, in adding hysteresis and confidence intervals, and in extending the model from daily readiness to a slower biological-age estimate.

---

*Baseline · Copenhagen · 2026*

*This document is part of an open methodology release. Errata, corrections, and questions: open an issue on the repository.*
