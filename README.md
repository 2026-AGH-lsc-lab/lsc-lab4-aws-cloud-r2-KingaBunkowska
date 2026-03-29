# AWS Cloud Lab Report

## Assignment 1: Deployment Check
All four targets were exercised in later scenarios (Lambda zip, Lambda container, Fargate, EC2), and non-Lambda scenarios returned HTTP 200.

Note: the dedicated `assignment-1-endpoints.txt` file is not present in this repository snapshot, because lab took me more than 4 hours and EC2 machine I was working disconnected with all the files.

## Assignment 2: Scenario A - Cold Start Characterization
`scenario-A.txt` (30 sequential requests per Lambda variant):

- Lambda zip: p50 `96.1 ms`, p95 `148.3 ms`, p99 `1206.2 ms`, max `1206.2 ms`
- Lambda container: p50 `95.7 ms`, p95 `157.3 ms`, p99 `1261.2 ms`, max `1261.2 ms`

From CloudWatch REPORT parsing:
- Zip: `1060` reports, `12` cold starts, init avg `614.55 ms` (min `491.61`, max `646.70`), warm duration avg `78.63 ms`
- Container: `1090` reports, `13` cold starts, init avg `600.99 ms` (min `537.87`, max `628.65`), warm duration avg `74.07 ms`

Latency decomposition (approximation with available data):
- Cold path is dominated by init time (~`600 ms`), then handler (~`74-86 ms`), then network/overhead.
- Warm path is stable near `90-100 ms` end-to-end.
- Zip and container cold starts are similar in this run (no large practical difference).

## Assignment 3: Scenario B - Warm Steady-State Throughput

| Environment | Concurrency | p50 (ms) | p95 (ms) | p99 (ms) | Max (ms) |
|---|---:|---:|---:|---:|---:|
| Lambda (zip) | 5 | 97.07 | 115.55 | 140.12 | 155.85 |
| Lambda (zip) | 10 | 95.17 | 114.67 | 150.60 | 880.39 |
| Lambda (container) | 5 | 94.43 | 111.93 | 133.17 | 162.04 |
| Lambda (container) | 10 | 90.88 | 114.96 | 148.82 | 694.32 |
| Fargate | 10 | 794.20 | 1005.50 | 1167.80 | 1198.10 |
| Fargate | 50 | 4098.10 | 4479.50 | 4602.30 | 4794.90 |
| EC2 | 10 | 191.90 | 249.98 | 277.77 | 315.05 |
| EC2 | 50 | 913.10 | 1062.80 | 1150.40 | 1247.80 |

Observations:
- Lambda p50 is almost unchanged from c=5 to c=10, because its load balanced by AWS.
- Fargate/EC2 latency rises strongly with higher concurrency due to queueing on a single running unit.

## Assignment 4: Scenario C - Burst from Zero

| Environment | p50 (ms) | p95 (ms) | p99 (ms) | Max (ms) |
|---|---:|---:|---:|---:|
| Lambda (zip)| 101.20 | 742.30 | 1238.40 | 1322.70 |
| Lambda (container)| 97.80 | 701.60 | 1196.20 | 1284.90 |
| Fargate | 3917.50 | 4298.30 | 4485.00 | 4564.60 |
| EC2 | 889.00 | 1524.20 | 1667.70 | 1687.60 |

SLO check (p99 < 500 ms):
- Lambda (zip): fails.
- Lambda (container): fails.
- Fargate and EC2 failed, because they didn't autoscale - so burst was too much for them.


## Assignment 5: Cost at Zero Load
Calculations in file `assignment5.txt`
- Lambda idle cost: `$0` (scales to zero).
- Fargate (0.5 vCPU, 1 GB): monthly always-on cost reported as `$17.77248`.
- EC2 (t3.small): monthly always-on cost `$14.976`.

Monthly idle-only cost ranking: `Lambda < EC2 < Fargate`.

## Assignment 6: Cost Model, Break-Even, Recommendation
Traffic model:
- Peak: 100 RPS for 30 min/day
- Normal: 5 RPS for 5.5 h/day
- Idle: 18 h/day
- Requests/day = `279,000`, requests/month = `8,370,000`

Lambda monthly cost formula:
- `request_cost = requests * 0.20 / 1,000,000`
- `compute_cost = requests * duration_s * 0.5 GB * 0.0000166667`

Using Scenario B p50 duration for Lambda container (`90.88 ms`):
- Request cost: `8.37 * 0.20 = $1.674`
- Compute cost: `8,370,000 * 0.09088 * 0.5 * 0.0000166667 = $6.339`
- Total Lambda monthly: `~$8.013`

Comparison with always-on options:
- Lambda: `~$8.01/month`
- EC2: `$14.98/month`
- Fargate: `$17.77/month`

Recommendation:
- For this traffic profile and cost objective, Lambda is the cheapest option, but don't meet the SLO without additional warming-up. If the traffic is predictable measures can be taken to ensure Lambda is warmed up.
- As deployed in this lab, no validated burst result clearly satisfies `p99 < 500 ms` across spike conditions.
- Recommendation would change if sustained average load rises well above `~7.16 RPS` and stays high enough to justify always-on capacity with scaling.

## Figures
Existing generated figures are available in `results/figures/`:
- `fig1_latency_decomposition.png`
- `fig2_cost_vs_rps.png`
- `fig3_pareto_frontier.png`
- `fig4_latency_table.png`
