# Baseline · Biological Age Algorithm

> The open-source biological age engine that powers [Baseline](https://baseline.ai).

[github.com/baseline-technologies/bio-age](https://github.com/baseline-technologies/bio-age)

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6.svg)](tsconfig.json)
[![Whitepaper](https://img.shields.io/badge/whitepaper-PDF-black.svg)](WHITEPAPER.md)

Most consumer biological age scores recompute every day, hide their math, and present single point estimates with no acknowledgment of uncertainty. We think that's wrong on three counts. This repository is our attempt to do it differently — and to publish the work so it can be checked.

**Read the whitepaper:** [`WHITEPAPER.md`](WHITEPAPER.md).

---

## What's in here

| File | What it does |
|---|---|
| [`src/vitals-index.ts`](src/vitals-index.ts) | The composite scoring engine. Takes raw rows from each data source and returns a `VitalsSnapshot` and a `BioAgeResult`. |
| [`src/smoothing.ts`](src/smoothing.ts) | Layer 2 — EWMA smoothing, rolling variance, hysteresis, confidence intervals. Pure functions, no I/O. |
| [`src/smoothing-config.ts`](src/smoothing-config.ts) | Half-life table. The single source of truth for how fast each metric is allowed to influence composite scores. |
| [`src/backtest.ts`](src/backtest.ts) | Replay harness — runs the algorithm against historical data day-by-day so you can see how it behaves. |

Everything is pure TypeScript with no database or network dependencies. You can drop it into any runtime that can execute TS.

---

## Why this exists

Three problems with the status quo:

**1. Daily-recomputing bio age is dishonest.** A score that drops 0.4 years because someone slept badly Tuesday is reporting a change in the *estimate*, not in the underlying biology. Sleep does not age a body overnight.

**2. Opacity is the norm.** Recovery scores from major brands can differ by 20+ points for the same night, with weights and thresholds undocumented. Users can't reason about what they're seeing.

**3. Point estimates without uncertainty are wrong.** A bio age of 38.2 with no confidence band implies precision the data doesn't support — particularly when the last blood panel is six months old.

This repo addresses all three. The how is in the whitepaper; the code below is the canonical implementation.

---

## The architecture in 30 seconds

```
Layer 1 — Raw store         every observation from every source, untouched
   ↓
Layer 2 — Smoothed state    EWMA per metric, half-lives matched to biology
   ↓
Layer 3 — Composite scores  pillars and bio age, computed from Layer 2 only
   ↓
Layer 4 — Display           what the user sees, gated by hysteresis
```

A single bad night cannot move biological age more than a fraction of a year, because the bad night first has to move a 7-day EWMA, which is anchored against weeks of prior data. That's the whole trick.

---

## Quick start

```bash
npm install @baseline-technologies/bio-age
```

```typescript
import { computeVitalsSnapshot, computeBioAge } from "@baseline-technologies/bio-age";

const snapshot = computeVitalsSnapshot(
  ouraRows,        // OuraRow[]
  stravaRows,      // StravaRow[]
  bloodRows,       // BloodRow[]
  "2026-05-08",    // today
  withingsRows,    // optional
  airthingsRows,   // optional
);

const bioAge = computeBioAge(snapshot, calendarAge);

console.log(bioAge);
// {
//   bioAge: 36.4,
//   calendarAge: 39,
//   yearsDelta: -2.6,
//   ci: { low: 64, high: 78 },             // confidence interval on the 0-100 score
//   ciYears: { low: 35.1, high: 38.2 },    // CI mapped back to years
//   contributions: [
//     { key: "recovery", label: "Rest",   yearsDelta: -1.2 },
//     { key: "fitness",  label: "Engine", yearsDelta: -2.8 },
//     { key: "metabolic", label: "Inside", yearsDelta:  1.4 },
//   ],
//   priorBioAge: 36.6,
//   priorYearsDelta: -2.4,
// }
```

Every input row type, every weight, and every threshold is exported. There's nothing hidden.

---

## The half-life table

This is the most important configuration in the system. It says how fast each metric is allowed to change a composite score. From [`src/smoothing-config.ts`](src/smoothing-config.ts):

| Metric group | Half-life | Why |
|---|---|---|
| HRV, RHR, sleep score, sleep duration, body temperature | 7 days | ANS metrics shift week-to-week |
| Indoor air quality | 3 days | Environment changes faster than physiology |
| Body fat, blood pressure, weight stability | 14 days | Real tissue change is measurable in fortnights |
| Acute:chronic workload ratio | 14 days | Load balance shifts on the order of a training block |
| Training consistency, weekly volume, intensity | 30 days | Aerobic adaptation requires weeks |
| Lab values (ApoB, LDL, HDL, trig, HbA1c, hsCRP, ferritin, vit D) | None | Discrete measurements with their own date stamp |

If you think a half-life is wrong, **open an issue or a PR**. Tuning these is exactly the kind of thing the open-source process is good for.

---

## Pillar weights

| Pillar | Vitals weight | Bio age range |
|---|---|---|
| Rest (Oura, Airthings) | 35% | ±2.5 years |
| Engine (Strava) | 30% | ±4.5 years |
| Inside (blood panel, Withings) | 35% | ±3.0 years |

Engine carries the most weight on bio age because cardiovascular fitness has the strongest published evidence as a determinant of biological age. The vitals weights skew toward Rest and Inside because those reflect short-term modifiable state.

The full per-metric weights live in [`src/vitals-index.ts`](src/vitals-index.ts) and are documented section by section in the [whitepaper](WHITEPAPER.md).

---

## Hysteresis: why the score doesn't jitter

The displayed bio age only updates when the new computed value differs from the displayed value by more than one standard deviation of the score's 30-day variance:

```typescript
function applyHysteresis(newScore, displayedScore, variance30) {
  if (variance30 === null || variance30 <= 0) return newScore;
  const stddev = Math.sqrt(variance30);
  if (Math.abs(newScore - displayedScore) > stddev) return newScore;
  return displayedScore;
}
```

Stillness is a feature. If the score doesn't change for three days, that means we're confident nothing meaningful changed. We don't add artificial movement to make the dashboard feel "alive."

Daily readiness scores update freely — that's what they're for. Bio age gets the gate. Different scores answer different questions.

---

## Confidence intervals

Every score has a 95% CI. The interval widens with:

1. **Raw input variance** — high day-to-day swings → wider CI
2. **Staleness** — each day a primary input is >48h old → 25% wider
3. **Missing inputs** — proportional widening, doubles when all primaries are missing

```typescript
halfWidth = 1.96 · √variance · (1 + 0.25 · staleExcess) · (1 + missingRatio)
```

Users see this as a band around the score. A user who synced today and has a recent panel sees a tight band. A user with a week-old Oura sync and no recent blood work sees a wide one. The band is honest about what the data supports.

---

## Backtesting

Every algorithm change is validated by replaying it against historical data. The harness in [`src/backtest.ts`](src/backtest.ts) takes a user's last 180 days of inputs and walks the algorithm forward day-by-day, returning a series of what-the-score-would-have-been. Three things to check:

- **Day-to-day variance** is within the noise floor on uneventful days
- **Responsiveness** — known events (illness, travel, training blocks) produce directionally correct changes
- **Single-input sensitivity** — no single input can move bio age by more than its pillar's allotted range

A change that fails any of these does not ship.

---

## What this is not

- **Not an epigenetic clock.** No DNA methylation, no telomere length, no cellular markers.
- **Not validated against mortality.** Informed by Levine PhenoAge and similar published clocks; not itself longitudinally validated.
- **Not a clinical diagnosis.** Discuss flagged markers with a physician.

A directional, transparent estimate of biological age built from inputs you can collect at home or with a single blood draw. Within those bounds, we believe it's more rigorous than any closed-source alternative — not because the math is fancier, but because it's visible.

---

## Contributing

We'd particularly welcome:

- **Tuning of half-lives or thresholds** — open an issue with your reasoning and ideally some data
- **New marker scorers** — additional lab markers, especially with cited reference ranges
- **Adapter modules** — for data sources we don't yet support (Garmin, Whoop, Polar, Coros, CGMs)
- **Validation studies** — independent comparisons against published clocks

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the contribution flow.

---

## License

Apache 2.0. See [`LICENSE`](LICENSE).

You can use this in commercial products. We ask that significant derivative implementations cite the whitepaper.

---

## Citation

If you reference this work academically:

```
Baseline Bio-Age v1.0 (2026).
A transparent biological age algorithm for consumer wearable inputs.
github.com/baseline-technologies/bio-age
```

Built with care in Copenhagen.
