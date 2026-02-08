# Zabbix Template – Public Pool API

This template monitors a **public-pool backend** (as used by `public-pool` / `public-pool-ui`) via its HTTP API and provides:

- Pool health & activity metrics
- Bitcoin network sanity checks
- Pool hashrate and expected time-to-find-block visualization
- Practical, low-noise triggers for real failure modes

The template is designed and tested for **Zabbix 7.4**.

---

## Monitored API Endpoints

The template uses HTTP Agent items to poll:

| Endpoint | Purpose |
|--------|--------|
| `/api/info` | Pool state, miners, user agents, uptime, highscores |
| `/api/network` | Bitcoin network state from bitcoind |
| `/api/info/chart` | Pool hashrate time series |

All other metrics are derived from these endpoints using **dependent** and **calculated** items.

---

## Template Macros

| Macro | Default | Description |
|-----|--------|-------------|
| `{$API_URL}` | `http://localhost:3334` | Base URL of public-pool backend |
| `{$CHAIN_EXPECTED}` | `main` | Expected Bitcoin chain (`main`, `test`, `regtest`, etc.) |

You can override these macros per-host if needed.

---

## Collected Metrics (Highlights)

### Pool activity
- Total miners (sum of all user agents)
- Total pool hashrate (raw H/s)
- Pool uptime (restart detection)
- Latest highscore update timestamp

### Bitcoin network
- Block height
- Difficulty
- Network hashrate
- Mempool transaction count
- Chain name
- bitcoind warning count

### Pool hashrate chart
From `/api/info/chart`:
- Latest hashrate value (H/s)
- Converted hashrate (PH/s)
- Timestamp of latest chart point

### Calculated metrics
- **Expected time to find a Bitcoin block (years)**

---

## Graphs

### Public-Pool Hashrate + Expected Time

Single graph with dual Y-axes:

- **Left axis:** Pool hashrate (PH/s)
- **Right axis:** Expected time to find a block (years)

This mirrors the semantics of the public-pool UI.

---

## Triggers (Alerting)

### API & data freshness
- API `/api/info` stopped updating (10 min)
- API `/api/network` stopped updating (15 min)
- Hashrate chart data stale (>30 min)

### Pool activity health
- Miner count drop >30% vs 1h average
- Miner count collapse (>60% vs 2h average, guarded)
- Hashrate drop >40% vs 2h average

### Mining progress
- No new highscores for 48 hours

### Stability
- Backend restarted recently

### Bitcoin network sanity
- Chain mismatch (configurable via `{$CHAIN_EXPECTED}`)
- bitcoind warnings present
- Block height not advancing for 2 hours

---

## Installation

1. Import the template YAML into Zabbix:
   `Configuration → Templates → Import`

2. Create or select a host and link **Template Public Pool API**.

3. Set required macros on the host if needed.

4. Ensure the Zabbix server or proxy can reach the API endpoint.

No Zabbix agent is required on the pool host.

---

## Notes

- Uses dependent items to minimize HTTP load
- Uses relative thresholds to avoid noise
- YAML structure matches Zabbix 7.4 import schema

---

## Future Extensions

- User-agent LLD
- Trigger prototypes per miner type
- Dashboard widgets
- Additional time-to-block units (days/months)
