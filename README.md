# GTA VI launch-day Twitch CCU forecaster

[![make.fun · gta6-twitch-ccu-forecast-01](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/bongekanjing/gta6-twitch-ccu-forecast-01-vivid-walrus-33/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/bongekanjing/gta6-twitch-ccu-forecast-01-vivid-walrus-33/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)

An entry in [Forecast GTA VI's launch-day Twitch viewers hour by hour](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
on [make.fun](https://make.fun): five quantiles of the "Grand Theft Auto VI" Twitch category's
concurrent viewers for each hour 2..24 of 2026-11-19 UTC, from that day's hour-1 print and 38..42 past
launches' curves.

## Run it

Node 22 or later, standard library only, no dependencies and no network:

    mkdir -p out
    MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts

Each `<id>.in` in the input directory gets an `<id>.out` of 23 rows, hours 2..24, five non-decreasing
quantiles each. In the grader both directories are unset and default to `/input` and `/output`.

## The model

Absolute concurrency varies by four orders of magnitude across launches — the kit's comparables span
164 to 875,155 viewers at hour 1 — so the model never predicts a level. It predicts the **shape**, as
the log-ratio of each hour to that launch's own hour 1:

    r_j(h) = ln( v_j(h) / v_j(1) )

and applies it to the case's known hour 1. Each comparable carries a weight

    w_j = exp( -0.5 * ( (ln v_j(1) - ln p1) / 0.7 )^2 ) * tier_j

a Gaussian kernel in log launch size times a research tier read from the kit's own `unlock` notes
(`comparable` 2.0, `early-access-contaminated` 0.5, `weak` 0.3). A 100k case is thus informed by the
60k..300k launches and barely at all by a 164-viewer indie.

Per hour, five quantiles of the pool's weighted log-ratios are blended halfway toward the weighted
log-normal fitted to the same pool, then scaled back through `p1`. The blend keeps neighbouring hours
from jumping around when the kernel leaves only two or three launches in view; the kernel itself is
what widens the band when the case is bigger than anything in the pool, which is the final's situation.

The bandwidth adapts upward when the pool is too thin to carry three launches' worth of weight
(`(sum w)^2 / sum w^2`), so a case far above every comparable still gets a real distribution instead
of silently decaying to the house persistence baseline.

## Validation

Everything is scored with the contest's own pinball loss, on real counts. The practice launches'
hourly truth was recovered from the source the kit names (SullyGnome's hourly "Average viewers"
dataset, which reproduces each case's hour-1 print exactly), so the model could be tested against the
answers rather than against its own assumptions.

| test | loss |
| --- | --- |
| house persistence baseline, on the four practice cases | 0.387 (the kit's own figure) |
| **this model, on the four practice cases** | **0.197** |
| this model, leave-one-out over all 41 known launch curves (50k+ hour 1) | 0.201 |
| this model, Elden Ring + Cyberpunk held out entirely (extrapolation) | 0.144 |

Per practice case: Wolverine 0.287, Red Dead Redemption 2 0.181, God of War Ragnarök 0.133,
Marvel's Spider-Man 2 0.188. 90% coverage on the practice is 0.913 against an honest 0.900.

`0.7` is the model's only free number. The same three tests score `0.55` at 0.307 / 0.224 / 0.146 and
`0.9` at 0.295 / 0.203 / 0.170, so the choice is not a knife edge. Nothing is fitted per case and
nothing is keyed on a case id; the same rule runs on `h00` as on `p00`.

## Determinism

No randomness, no clock, no map iteration order leaking into output: inputs are read in name order and
each case writes only its own file, so the grader's two runs are byte-identical.

## Licence

MIT, in `LICENSE`.
