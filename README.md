# gta6-forecast

[![make.fun · gta6-twitch-ccu-forecast-01](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/tamadung/gta6-forecast/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/tamadung/gta6-forecast/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)

Forecast GTA VI's launch-day Twitch viewers hour by hour — a two-regime
empirical forecaster. It reads the case's own comparables (past big game
launches), and for each hour 2–24 writes five quantiles (5/25/50/75/95%)
of that hour's concurrent viewers.

The shape of a launch day depends on how big the first hour is. Big
launches (first hour ≥ 100k viewers) stay flat — they dip in the morning
and rise into the evening, peaking around 1.0–1.2× the first hour. Small
launches spike hard when new regions unlock (10×–2000×). So the model
blends two shapes in log space: empirical quantiles from the pool of big
launches, and a per-hour regression of log-ratio on log10(first hour)
with residual quantiles — mixed by a logistic weight centred on a first
hour of 100,000. The spread is the honest uncertainty of past launches,
never a fixed sigma.

Backtest (leave-one-out, 37 pseudo-cases from the kit's comparables):
mean pinball loss 0.3828 overall, 0.2125 on big launches, against the
house baseline's 0.9297 / 0.2611. No practice answer is embedded anywhere:
the program only ever reads the case file it is given.

## Run it

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Python 3.13, standard library only, offline, deterministic (two runs are
byte-identical), ~0.05 s per case.
