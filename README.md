# Zabbix Template -- Public Pool API

This template monitors a **public-pool backend** (as used by
`public-pool` / `public-pool-ui`) via its HTTP API and provides:

-   Pool health & activity metrics
-   Bitcoin network sanity checks
-   Pool hashrate monitoring & stagnation detection
-   Expected time-to-find-block visualization (years + days)
-   Practical, low-noise triggers for real failure modes

The template is designed and tested for **Zabbix 7.4**.

------------------------------------------------------------------------

## Monitored API Endpoints

The template uses HTTP Agent items to poll:

  Endpoint            Purpose
  ------------------- -----------------------------------------------------
  `/api/info`         Pool state, miners, user agents, uptime, highscores
  `/api/network`      Bitcoin network state from bitcoind
  `/api/info/chart`   Pool hashrate time series

All other metrics are derived from these endpoints using **dependent**
and **calculated** items to minimize HTTP load.

------------------------------------------------------------------------

## Template Macros

  -------------------------------------------------------------------------------------------
  Macro                              Default                   Description
  ---------------------------------- ------------------------- ------------------------------
  `{$API_URL}`                       `http://localhost:3334`   Base URL of public-pool
                                                               backend

  `{$CHAIN_EXPECTED}`                `main`                    Expected Bitcoin chain
                                                               (`main`, `test`, `regtest`,
                                                               etc.)

  `{$HASHRATE_FLAT_THRESHOLD_REL}`   `0.03`                    Relative change threshold (3%)
                                                               for hashrate stagnation
                                                               detection

  `{$HASHRATE_FLAT_THRESHOLD_ABS}`   `100000000`               Absolute H/s delta override
                                                               for stagnation detection
  -------------------------------------------------------------------------------------------

### Hashrate Flat Detection Logic

The template detects when pool hashrate is **not changing
meaningfully**, which can indicate:

-   Backend freeze
-   Stuck chart updates
-   Broken API responses
-   Constant synthetic values

The trigger fires when:

-   Relative change is below `{$HASHRATE_FLAT_THRESHOLD_REL}`
-   AND absolute change is below `{$HASHRATE_FLAT_THRESHOLD_ABS}`

This dual threshold prevents false positives on: - Small pools (low
absolute variation) - Very large pools (low relative noise)

------------------------------------------------------------------------

## Collected Metrics

### Pool Activity

-   Total miners (sum of all `userAgents[].count`)
-   Total pool hashrate (raw H/s)
-   Converted pool hashrate (PH/s)
-   Pool uptime start timestamp (restart detection)
-   Latest highscore update timestamp (extracted from array)

------------------------------------------------------------------------

### Bitcoin Network

-   Block height
-   Difficulty
-   Network hashrate (`networkhashps`)
-   Mempool transaction count
-   Chain name
-   bitcoind warnings count

------------------------------------------------------------------------

### Pool Hashrate Chart

From `/api/info/chart`:

-   Latest hashrate value (H/s)
-   Latest timestamp (epoch)
-   Converted PH/s value

------------------------------------------------------------------------

### Calculated Metrics

-   **Expected time to find a Bitcoin block (years)**
-   **Expected time to find a Bitcoin block (days)**

The dual units allow flexible graphing and alerting.

------------------------------------------------------------------------

## Graphs

### Public-Pool Hashrate + Expected Time

Single graph with dual Y-axes:

-   **Left axis:** Pool hashrate (PH/s)
-   **Right axis:** Expected time to find a block (years)

This mirrors the semantics of the public-pool UI while keeping
operational visibility in Zabbix.

------------------------------------------------------------------------

## Triggers (Alerting)

### API Availability

-   **/api/info unreachable or no data for 10m**
-   **/api/network unreachable or no data for 15m**

------------------------------------------------------------------------

### Pool Health

-   **Hashrate not changing (relative + absolute threshold guarded)**
-   **Total miners dropped \>30% vs 1h average**
-   **Total miners collapsed (\>60% vs 2h average)**
-   **Total hashrate dropped \>40% vs 2h average**
-   **Backend restarted within last 10 minutes**

------------------------------------------------------------------------

### Bitcoin Network Sanity

-   **Chain mismatch** (controlled via `{$CHAIN_EXPECTED}`)
-   **bitcoind warnings present**
-   **Block height not advancing for 2 hours**

------------------------------------------------------------------------

## Installation

1.  Import the template YAML into Zabbix:\
    `Configuration → Templates → Import`

2.  Create or select a host and link **Template Public Pool API**

3.  Set required macros on the host if needed

4.  Ensure the Zabbix server or proxy can reach the API endpoint

No Zabbix agent is required on the pool host.

------------------------------------------------------------------------

## Design Principles

-   Uses **dependent items** to minimize HTTP load
-   Uses **relative thresholds** to avoid alert noise
-   Uses **guarded collapse detection** to prevent false positives
-   YAML structure matches **Zabbix 7.4 import schema**
-   Avoids overly aggressive freshness triggers on chart data

------------------------------------------------------------------------

## Future Extensions

-   User-agent LLD
-   Trigger prototypes per miner type
-   Dashboard widgets
-   Additional expected-time units (months, weeks)
-   SLA-style availability metrics
