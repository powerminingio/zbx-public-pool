# Zabbix Template — Public Pool API

This template monitors a **public-pool backend** used by
`public-pool` / `public-pool-ui` through its HTTP API and provides:

- Pool health and activity metrics
- Automatic per-`userAgent` monitoring
- Bitcoin network sanity checks
- Pool hashrate monitoring and stagnation detection
- Expected time-to-find-block visualization
- Configurable, low-noise triggers for operational failures

The template is designed and tested for **Zabbix 7.4**.

---

## Monitored API Endpoints

The template uses HTTP Agent items to poll:

| Endpoint | Purpose |
|---|---|
| `/api/info` | Pool state, miners, user agents, uptime, and highscores |
| `/api/network` | Bitcoin network state from bitcoind |
| `/api/info/chart` | Pool hashrate time series |

All other metrics are derived using **dependent**, **calculated**, and
**low-level discovery** items to minimize HTTP load.

---

## Template Macros

| Macro | Default | Description |
|---|---:|---|
| `{$API_URL}` | `http://localhost:3334` | Base URL of the public-pool backend |
| `{$CHAIN_EXPECTED}` | `main` | Expected Bitcoin chain, such as `main`, `test`, or `regtest` |
| `{$NODATA}` | `10m` | General no-data interval used by applicable availability checks |
| `{$HASHRATE_FLAT_THRESHOLD_REL}` | `0.03` | Relative hashrate variation threshold; `0.03` means 3% |
| `{$HASHRATE_FLAT_THRESHOLD_ABS}` | `100000000` | Absolute minimum hashrate variation threshold in H/s; default is 100 MH/s |
| `{$MINERS_DROP_THRESHOLD_PCT}` | `30` | Miner-count drop percentage required to trigger the relative drop alert |
| `{$MINERS_COLLAPSE_THRESHOLD_PCT}` | `60` | Severe miner-count collapse percentage required to trigger the high-severity alert |
| `{$HASHRATE_DROP_THRESHOLD_PCT}` | `5` | Pool-hashrate drop percentage required to trigger the relative drop alert |
| `{$DROP_BASELINE_WINDOW}` | `2h` | Shared historical baseline window used by the miner-count and hashrate drop triggers |

Macros can be overridden per host without changing the template.

### Relative drop detection

The miner-count and total-hashrate drop triggers compare a recent
5-minute average against the average over `{$DROP_BASELINE_WINDOW}`.

Default behavior:

- Miner-count warning: more than **30%** below the **2-hour** baseline
- Miner-count collapse alert: more than **60%** below the **2-hour** baseline, when the baseline is above 20 miners
- Hashrate alert: more than **5%** below the **2-hour** baseline

For example, changing:

```text
{$DROP_BASELINE_WINDOW} = 4h
{$HASHRATE_DROP_THRESHOLD_PCT} = 10
```

makes the hashrate trigger alert when the recent value is more than 10%
below its 4-hour average.

### Hashrate stagnation detection

The template also detects when the hashrate series is not changing
meaningfully over its observation window.

The effective flatness threshold is the larger of:

```text
average hashrate × {$HASHRATE_FLAT_THRESHOLD_REL}
```

and:

```text
{$HASHRATE_FLAT_THRESHOLD_ABS}
```

With the defaults, variation must exceed both the practical 100 MH/s
floor and, for sufficiently large pools, 3% of the average hashrate.

To force a strictly absolute threshold for a host, set:

```text
{$HASHRATE_FLAT_THRESHOLD_REL} = 0
{$HASHRATE_FLAT_THRESHOLD_ABS} = <desired threshold in H/s>
```

---

## Collected Metrics

### Pool activity

- Total connected miners, summed across all `userAgents`
- Total pool hashrate in H/s
- Converted pool hashrate in PH/s
- Pool uptime start timestamp
- Latest highscore update timestamp

The highscore timestamp is collected for visibility only. The template
does not alert on highscore inactivity because `/api/info` returns only
the top entries and long-running pools may legitimately have no changes
for extended periods.

### Per-user-agent metrics

A low-level discovery rule automatically discovers every distinct
`userAgent` returned by `/api/info`.

For each discovered user agent, the template creates items for:

- Connected miner count
- Total hashrate
- Best difficulty

The discovery is not based on a hardcoded list. New user-agent values
are added automatically, and the discovery and item prototypes reuse
the existing `/api/info` master item without additional HTTP requests.

### Bitcoin network

- Block height
- Difficulty
- Network hashrate (`networkhashps`)
- Mempool transaction count
- Chain name
- bitcoind warning count

### Pool hashrate chart

From `/api/info/chart`:

- Latest hashrate value in H/s
- Latest data-point timestamp
- Converted hashrate in PH/s

### Calculated metrics

- Effective hashrate-flatness threshold in H/s
- Expected time to find a Bitcoin block in years
- Expected time to find a Bitcoin block in days

---

## Graphs

### Public-Pool Hashrate + Expected Time

A single graph with two Y-axes:

- **Left axis:** Pool hashrate in PH/s
- **Right axis:** Expected time to find a block in years

This mirrors the semantics of the public-pool UI while retaining the
data in Zabbix.

---

## Triggers

### API availability

- `/api/info` unreachable or no data for 10 minutes
- `/api/network` unreachable or no data for 15 minutes

### Pool health

- Hashrate not changing beyond the configured relative/absolute threshold
- Total miners dropped by more than `{$MINERS_DROP_THRESHOLD_PCT}` percent versus the `{$DROP_BASELINE_WINDOW}` average
- Total miners collapsed by more than `{$MINERS_COLLAPSE_THRESHOLD_PCT}` percent versus the `{$DROP_BASELINE_WINDOW}` average, with a minimum-baseline guard
- Total hashrate dropped by more than `{$HASHRATE_DROP_THRESHOLD_PCT}` percent versus the `{$DROP_BASELINE_WINDOW}` average
- Backend restarted within the last 10 minutes

### Bitcoin network sanity

- Chain mismatch, controlled by `{$CHAIN_EXPECTED}`
- bitcoind warnings present
- Block height not advancing for 2 hours

---

## Installation

1. Import the template YAML in Zabbix:

   ```text
   Data collection → Templates → Import
   ```

2. Create or select a host and link **Template Public Pool API**.

3. Override macros on the host when required.

4. Ensure the Zabbix server or proxy can reach `{$API_URL}`.

No Zabbix agent is required on the pool host for these HTTP checks.

---

## Design Principles

- Reuses HTTP master items through dependent items and LLD
- Discovers user-agent categories automatically
- Uses configurable percentages for miner drops, severe miner collapse, and hashrate drops
- Uses one shared, configurable baseline window for related drop checks
- Uses guarded comparisons to reduce short-term alert noise
- Avoids unreliable highscore-inactivity and overly aggressive chart-freshness alerts
- Follows the Zabbix 7.4 YAML import structure

---

## Future Extensions

- Trigger prototypes for selected user-agent categories
- Dashboard widgets
- Additional expected-time units such as months or weeks
- SLA-style availability metrics
