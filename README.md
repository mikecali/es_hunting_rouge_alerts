# 🔍 ES Query Rogue Alert Hunter

> Behavioural detection, automated remediation, and cluster impact monitoring for runaway Kibana `.es-query` alerting rules on Elasticsearch 9.x clusters.

---

## The Problem

Kibana alerting rules are invisible to the cluster until they cause damage. A single poorly-configured `.es-query` rule — running `match_all` over 24 hours of data every minute with no filters and `excludeHitsFromPreviousRun: false` — can consume enough search thread pool capacity to degrade every other query on the cluster. By the time you notice CPU saturation, the rule has been running for days.

The challenge: you cannot detect a rogue alert by looking at its name or tags.. What matters is **how long it takes to execute & How frequent the alerts executes and WHAT is the alerts looking for?**.

---

## How It Works

This solution detects rogue alerts **purely by execution behaviour**, not by metadata. It uses an **Elastic Workflow** (GA in 9.4) running every 5 minutes to:

1. Detect any `.es-query` rule whose execution time exceeds 1 second
2. Enrich the detection with full rule metadata via the Kibana alerting API
3. Write a detection document to `rogue-alert-detections`
4. Auto-disable **critical** offenders (>5s) and notify via email
5. Email-notify for **warning** tier (>1s) without disabling

### Detection Signal

```
event.duration  >  1,000,000,000 ns  (1 second)
AND event.provider  =  "alerting"
AND event.action    =  "execute"
AND rule.id         =  <any .es-query rule>
```

Read from `.kibana-event-log*` using ES|QL — no nested query required, `rule.id` and `rule.name` are available as top-level fields in 9.x.

### Detection Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     ELASTICSEARCH CLUSTER                               │
│                                                                         │
│  ┌─────────────────────┐      executes      ┌──────────────────────┐   │
│  │   Kibana Task       │ ─────────────────► │  .es-query Rule      │   │
│  │   Manager           │                    │  (every 1–5 min)     │   │
│  └─────────────────────┘                    └──────────┬───────────┘   │
│                                                        │               │
│                                              writes execution record    │
│                                                        ▼               │
│                                           ┌────────────────────────┐   │
│                                           │  .kibana-event-log*    │   │
│                                           │                        │   │
│                                           │  event.provider:       │   │
│                                           │    "alerting"          │   │
│                                           │  event.action:         │   │
│                                           │    "execute"           │   │
│                                           │  event.duration:       │   │
│                                           │    1,101,000,000 ns    │   │
│                                           │  rule.id: <uuid>       │   │
│                                           │  rule.name: <name>     │   │
│                                           └────────────┬───────────┘   │
└────────────────────────────────────────────────────────┼───────────────┘
                                                         │
                         ┌───────────────────────────────▼───────────────┐
                         │   Elastic Workflow: OPS - Rogue Alert Detector │
                         │   Schedule: every 5 minutes (GA in 9.4)        │
                         │                                                │
                         │  Step 1 — ES|QL query on .kibana-event-log*   │
                         │    event.duration > 1,000,000,000 ns           │
                         │    STATS max_duration, execution_count         │
                         │    BY rule.id, rule.name                       │
                         │    EVAL severity = CASE(>5s, critical,         │
                         │                         warning)               │
                         │                                                │
                         │  Step 2 — foreach offending rule:             │
                         │    GET /api/alerting/rule/{id}                │
                         │    → rule.created_by, rule.tags               │
                         │    → rule.schedule, params.timeWindowSize      │
                         │    → params.excludeHitsFromPreviousRun        │
                         │                                                │
                         │  Step 3 — Write enriched detection            │
                         │    → rogue-alert-detections index             │
                         │                                                │
                         │  Step 4 — if CRITICAL (>5s):                 │
                         │    POST /api/alerting/rule/{id}/_disable      │
                         │    cases.createCase                           │
                         │    kibana.request → email connector           │
                         │                                                │
                         │  Step 5 — if WARNING (>1s):                  │
                         │    kibana.request → email connector           │
                         └───────────────────────────────┬───────────────┘
                                                         │
                         ┌───────────────────────────────▼───────────────┐
                         │   rogue-alert-detections index                 │
                         │                                                │
                         │  {                                             │
                         │    "rule.name": "rogue alert metrics 24h",    │
                         │    "rule.created_by": "2763457433",           │
                         │    "rule.tags": "rogue",                      │
                         │    "rule.schedule": "1m",                     │
                         │    "detection.duration_ms": 1101,             │
                         │    "detection.severity": "warning",           │
                         │    "detection.window_size_h": 24,             │
                         │    "detection.excludes_previous_runs": false, │
                         │    "detection.source": "workflow"             │
                         │  }                                             │
                         └───────────────────────────────┬───────────────┘
                                                         │
                         ┌───────────────────────────────▼───────────────┐
                         │   [OPS] ES Query Rogue Alert Hunter            │
                         │   Kibana Dashboard — auto-refresh 30s          │
                         │   Time range: inherits from dashboard picker   │
                         └───────────────────────────────────────────────┘
```

### Why Behavioural Detection?

| Detection approach | Catches renamed rules | Catches rules with clean tags | Catches new accounts | Catches rules that look legitimate |
|---|---|---|---|---|
| Tag-based (`_DIAG_TEMP`) | ❌ | ❌ | ❌ | ❌ |
| Name-based pattern match | ❌ | ❌ | ❌ | ❌ |
| Creator-based allowlist | ❌ | ❌ | ❌ | ❌ |
| **Duration-based (this solution)** | ✅ | ✅ | ✅ | ✅ |

Validated: a rule tagged `monitoring, ops` with a clean name and known creator was still detected because its execution exceeded 1 second.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  INPUTS (read-only)                                          │
│                                                              │
│  .kibana-event-log*     — rule execution history (ES|QL)    │
│  Kibana Alerting API    — rule metadata per offender        │
└──────────────────────────┬───────────────────────────────────┘
                           │  read only
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  DETECTION & REMEDIATION ENGINE                              │
│                                                              │
│  Elastic Workflow: OPS - Rogue Alert Detector                │
│  ID: ops-rogue-alert-detector                                │
│  Schedule: every 5 minutes                                   │
│  Requires: Kibana 9.4+ (Workflows GA)                        │
└──────────────────────────┬───────────────────────────────────┘
                           │  writes detections
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  OUTPUT                                                      │
│                                                              │
│  rogue-alert-detections index                                │
│  1 document per offending rule per Workflow run              │
└──────────────────────────┬───────────────────────────────────┘
                           │  reads
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  DASHBOARD                                                   │
│                                                              │
│  [OPS] ES Query Rogue Alert Hunter                           │
│  Kibana Dashboard — default space                            │
│  Time range: last 24h — auto-refresh 30s                     │
└──────────────────────────────────────────────────────────────┘
```

---

## Severity Classification

```
Rule execution duration (ms)
│
0 ──────────────────── 999 ms ─────────────── 4,999 ms ──────── ∞
│                          │                       │
│       NORMAL             │       WARNING         │   CRITICAL
│   (not detected)         │   (>1s threshold)     │  (>5s threshold)
│                          │                       │
│  Legitimate rules run    │  Broad query, large   │  match_all over days,
│  under 1 second.         │  time window, no      │  no filter, full
│  Fleet Agent rules:      │  filter applied.      │  shard scan.
│  60–130ms typical.       │  Email notification.  │  Auto-disabled +
│                          │  No auto-disable.     │  case + email.
```

---

## Dashboard

The `[OPS] ES Query Rogue Alert Hunter` dashboard has 9 panels across 4 rows, auto-refreshing every 30 seconds. All panels inherit the dashboard time range — no fixed overrides.

**Row 1 — Summary metrics**

| Panel | Type | Data source | What it shows |
|---|---|---|---|
| Rogue Detections Over Time | Lens bar chart (stacked) | `rogue-alert-detections` | Warning vs critical detections over time |
| Worst Single Execution (ms) | Legacy metric | `rogue-alert-detections` | Max `detection.duration_ms` — green <1s, amber 1–5s, red >5s |
| Total Detections | Lens metric | `rogue-alert-detections` | Count of all detection documents in the time window |
| Critical Detections (>5s) | Lens metric | `rogue-alert-detections` | Count where `detection.severity = critical` |
| Warning Detections (>1s) | Lens metric | `rogue-alert-detections` | Count where `detection.severity = warning` |

**Row 2 — Offenders table**

| Panel | Type | Data source | What it shows |
|---|---|---|---|
| Current Offenders — Enriched Detection Table | Legacy table | `rogue-alert-detections` | Rule ID, name, creator, severity, schedule, tags, detection count, max duration — sorted by max duration |

**Row 3 — Cluster-level signals**

| Panel | Type | Data source | What it shows |
|---|---|---|---|
| Rule Health Scorecard | Legacy table | `.kibana-event-log*` | All `.es-query` rules ranked by max execution duration with execution count and outcome |

**Row 4 — Cluster impact correlation**

| Panel | Type | Data source | What it shows |
|---|---|---|---|
| Task Manager Drift Over Time | Lens line chart | `.kibana-event-log*` | p95 and max Task Manager task-run duration at 5-minute intervals. Healthy baseline: p95 < 500ms |
| Alert Execution vs Task Manager Pressure | Lens line chart (2 series) | `.kibana-event-log*` | Max alerting execution duration overlaid against Task Manager p95 — shows the causal chain |

---

## Cluster Impact Correlation

On Elastic Cloud Hosted, monitoring data is stored in a separate deployment. The Row 4 panels use `.kibana-event-log*` as a proxy for cluster impact.

### Task Manager drift interpretation

| p95 range | Status | Meaning |
|---|---|---|
| < 500ms | ✅ Healthy | Task Manager running on schedule |
| 500ms – 1,000ms | ⚠️ Mild pressure | Monitor — possible low-impact rogue rule |
| 1,000ms – 2,000ms | 🔴 Saturated | Expensive alert tasks consuming worker threads |
| > 2,000ms | 🔴 Severe | Queue backing up, other tasks significantly delayed |

**Evidence from cluster (confirmed):**
```
Task Manager p95 sustained: 1,000–1,800ms  (healthy baseline: <500ms)
Task Manager max ceiling:   ~2,200ms
Schedule delay observed:    1,455ms  (tasks waiting 1.45s before starting)
Rule Success Ratio:         97.60%   (Stack Monitoring)
Queued Rules:               7
```

---

## What Gets Detected (and What Doesn't)

### Detected ✅
- `.es-query` rules with `match_all` and large time windows
- Rules with `excludeHitsFromPreviousRun: false` causing full re-scans
- Rules scanning broad index patterns (`logs-*`, `metrics-*`) without filters
- Rules on tight schedules (1m) with wide windows (24h+)
- Any rule from any account — creator is irrelevant to detection
- Rules that look legitimate by name and tags but are slow in practice

### Not Detected ❌
- Rules that run fast but generate excessive actions (action flood)
- Other rule types (`.index-threshold`, `slo.rules.burnRate`, ML rules)
- Rules that are slow due to cluster degradation rather than query design

> To extend detection to other rule types, add additional `WHERE rule.type` filters or remove the type constraint entirely from the ES|QL query.

---

## Production Deployment

### Prerequisites

| Requirement | Detail |
|---|---|
| Elasticsearch | 9.4+ |
| Kibana | 9.4+ (Elastic Workflows GA) |
| Email connector | Configured in Stack Management → Connectors |
| User role | Can create/manage Workflows, read `.kibana-event-log*`, write to `rogue-alert-detections` |

### Step 1 — Create the detections index

```json
PUT rogue-alert-detections
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp":                         { "type": "date" },
      "rule.id":                            { "type": "keyword" },
      "rule.name":                          { "type": "keyword" },
      "rule.type":                          { "type": "keyword" },
      "rule.tags":                          { "type": "keyword" },
      "rule.created_by":                    { "type": "keyword" },
      "rule.created_at":                    { "type": "date" },
      "rule.schedule":                      { "type": "keyword" },
      "detection.duration_ms":              { "type": "long" },
      "detection.severity":                 { "type": "keyword" },
      "detection.threshold_ms":             { "type": "long" },
      "detection.execution_time":           { "type": "date" },
      "detection.executions_in_window":     { "type": "integer" },
      "detection.window_size_h":            { "type": "integer" },
      "detection.excludes_previous_runs":   { "type": "boolean" },
      "detection.result_size":              { "type": "integer" },
      "detection.source":                   { "type": "keyword" }
    }
  }
}
```

### Step 2 — Create Kibana data views

```json
POST kbn:/api/data_views/data_view
{
  "data_view": {
    "title": "rogue-alert-detections",
    "timeFieldName": "@timestamp",
    "name": "[OPS] Rogue Alert Detections"
  }
}
```

```json
POST kbn:/api/data_views/data_view
{
  "data_view": {
    "title": ".kibana-event-log*",
    "timeFieldName": "@timestamp",
    "name": "[OPS] Kibana Event Log"
  }
}
```

Note the `id` from both responses — needed for Step 4.

### Step 3 — Deploy the Workflow

Go to **Stack Management → Workflows → Create workflow** and paste the contents of `workflow.yaml`.

Before saving replace:
- `<your-email-connector-id>` — get from Stack Management → Connectors (appears twice)
- `ops-team@yourcompany.com` — your ops distribution list (appears twice)
- `https://<your-kibana>` — your Kibana URL (appears in case description and emails)

**Workflow — tested and validated (`workflow_minimal.yaml`):**

> ✅ This is the version confirmed working on ES/Kibana 9.4.1.

```yaml
name: OPS - Rogue Alert Detector
description: |
  Detects rogue .es-query rules by execution duration,
  enriches with rule metadata via Kibana API,
  and writes to rogue-alert-detections index.
enabled: true
tags:
  - ops
  - rogue-alert-detection

triggers:
  - type: scheduled
    with:
      every: "5m"

steps:

  # ── Step 1: Find slow .es-query rules in last 5 minutes ──
  # Column order: [0]execution_count [1]last_execution [2]rule_id
  #               [3]rule_name [4]max_duration_ms [5]severity
  - name: find_slow_rules
    type: elasticsearch.esql.query
    with:
      query: |
        FROM .kibana-event-log*
        | WHERE @timestamp >= NOW() - 5 minutes
        | WHERE event.provider == "alerting"
        | WHERE event.action == "execute"
        | WHERE event.duration > 1000000000
        | STATS
            max_duration_ns = MAX(event.duration),
            execution_count = COUNT(*),
            last_execution = MAX(@timestamp)
          BY rule_id = rule.id, rule_name = rule.name
        | EVAL
            max_duration_ms = max_duration_ns / 1000000,
            severity = CASE(max_duration_ns >= 5000000000, "critical", "warning")
        | DROP max_duration_ns
        | WHERE max_duration_ms IS NOT NULL
        | SORT max_duration_ms DESC

  # ── Step 2: For each offender — enrich + write ──
  - name: process_detections
    type: foreach
    foreach: "{{ steps.find_slow_rules.output.values }}"
    steps:

      # 2a — Get rule metadata from Kibana alerting API
      - name: get_rule
        type: kibana.request
        with:
          method: GET
          path: "/api/alerting/rule/{{ foreach.item[2] }}"

      # 2b — Write enriched detection to index
      - name: index_detection
        type: elasticsearch.index
        with:
          index: rogue-alert-detections
          document:
            "@timestamp": "{{ foreach.item[1] }}"
            rule.id: "{{ foreach.item[2] }}"
            rule.name: "{{ foreach.item[3] }}"
            rule.type: ".es-query"
            rule.created_by: "{{ steps.get_rule.output.created_by }}"
            rule.created_at: "{{ steps.get_rule.output.created_at }}"
            rule.tags: "{{ steps.get_rule.output.tags }}"
            rule.schedule: "{{ steps.get_rule.output.schedule.interval }}"
            detection.duration_ms: "{{ foreach.item[4] }}"
            detection.severity: "{{ foreach.item[5] }}"
            detection.executions_in_window: "{{ foreach.item[0] }}"
            detection.execution_time: "{{ foreach.item[1] }}"
            detection.window_size_h: "{{ steps.get_rule.output.params.timeWindowSize }}"
            detection.excludes_previous_runs: "{{ steps.get_rule.output.params.excludeHitsFromPreviousRun }}"
            detection.result_size: "{{ steps.get_rule.output.params.size }}"
            detection.threshold_ms: 1000
            detection.source: "workflow"
```

**Workflow — remediation steps (not yet tested, `workflow.yaml`):**

> ⚠️ Steps 2c and 2d below extend the validated workflow above. Add them inside the `process_detections` foreach after step 2b. Test in a non-production environment before enabling auto-disable.

```yaml
      # 2c — If CRITICAL: auto-disable rule + create case + send email
      - name: handle_critical
        type: if
        condition: "foreach.item[5] : critical"
        steps:
          - name: disable_rule
            type: kibana.request
            with:
              method: POST
              path: "/api/alerting/rule/{{ foreach.item[2] }}/_disable"

          - name: create_case
            type: cases.createCase
            with:
              title: "Rogue Alert Auto-Disabled: {{ foreach.item[3] }}"
              severity: critical
              tags:
                - rogue-alert
                - auto-disabled

          - name: send_email_critical
            type: kibana.request
            with:
              method: POST
              path: "/api/actions/connector/<your-email-connector-id>/_execute"
              body:
                params:
                  to:
                    - "ops-team@yourcompany.com"
                  subject: "CRITICAL: Rogue Alert Auto-Disabled — {{ foreach.item[3] }}"
                  message: |
                    Rule {{ foreach.item[3] }} ({{ foreach.item[2] }}) has been
                    automatically disabled. Max execution: {{ foreach.item[4] }}ms.
                    Re-enable: POST /api/alerting/rule/{{ foreach.item[2] }}/_enable

      # 2d — If WARNING: email only, no disable
      - name: handle_warning
        type: if
        condition: "foreach.item[5] : warning"
        steps:
          - name: send_email_warning
            type: kibana.request
            with:
              method: POST
              path: "/api/actions/connector/<your-email-connector-id>/_execute"
              body:
                params:
                  to:
                    - "ops-team@yourcompany.com"
                  subject: "WARNING: Slow Alert Rule — {{ foreach.item[3] }}"
                  message: |
                    Rule {{ foreach.item[3] }} took {{ foreach.item[4] }}ms.
                    Rule has NOT been disabled — investigate if this persists.
```

### Step 4 — Smoke test the Workflow

After saving, click **Run** in the Workflow UI. Check the execution log — Step 1 should return your rogue rules, Step 2 should enrich and write them.

Verify detections landed:

```json
GET rogue-alert-detections/_search
{
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 3,
  "query": { "term": { "detection.source": "workflow" } }
}
```

Expected document:
```json
{
  "@timestamp": "2026-06-29T07:17:15.787Z",
  "rule.id": "d387ad5c-76ff-4667-ae2d-c637b586e50b",
  "rule.name": "rogue alert metrics 24h",
  "rule.created_by": "2763457433",
  "rule.tags": "rogue",
  "rule.schedule": "1m",
  "detection.duration_ms": "1101",
  "detection.severity": "warning",
  "detection.window_size_h": "24",
  "detection.excludes_previous_runs": "false",
  "detection.source": "workflow"
}
```

### Step 5 — Deploy the Lens saved objects

The dashboard requires three Lens saved objects created before import:

```
POST kbn:/api/saved_objects/lens/ops-rogue-detections-over-time
```
→ See `devtools_dashboard.txt` Step 1

```
POST kbn:/api/saved_objects/lens/ops-tm-drift-over-time
```
→ See `devtools_correlation_panels.txt` Step 1

```
POST kbn:/api/saved_objects/lens/ops-alert-vs-tm-overlay
```
→ See `devtools_correlation_panels.txt` Step 2

### Step 6 — Deploy the dashboard

```
POST kbn:/api/saved_objects/dashboard/ops-rogue-alert-hunter-dash
```

Replace the two data view IDs from Step 2 in `devtools_dashboard.txt`, then run it.

After creating, open the dashboard → Edit → **Add panel → Add from library** → add the three Lens panels → Save.

Open the 3 metric panels (Total, Critical, Warning) in the Lens editor and click **Save and return** on each — this clears any migration-added time range overrides so they respect the dashboard time picker.

### Step 7 — Open the dashboard

```
https://<your-kibana>/app/dashboards#/view/ops-rogue-alert-hunter-dash
```

---

## Operational Queries

### Current offenders (last 24h)

```
POST /_query
{
  "query": "FROM .kibana-event-log* | WHERE @timestamp >= NOW() - 24 hours | WHERE event.provider == \"alerting\" | WHERE event.action == \"execute\" | WHERE event.duration > 1000000000 | STATS max_duration_ns = MAX(event.duration), execution_count = COUNT(*) BY rule_id = rule.id, rule_name = rule.name | EVAL max_duration_ms = max_duration_ns / 1000000 | SORT max_duration_ms DESC"
}
```

### Task Manager drift (last 24h)

```
POST /_query
{
  "query": "FROM .kibana-event-log* | WHERE @timestamp >= NOW() - 24 hours | WHERE event.provider == \"taskManager\" | WHERE event.action == \"task-run\" | STATS p95_ns = PERCENTILE(event.duration, 95), max_ns = MAX(event.duration), count = COUNT(*) BY bucket = DATE_TRUNC(5 minutes, @timestamp) | EVAL p95_ms = p95_ns / 1000000, max_ms = max_ns / 1000000 | SORT bucket ASC"
}
```

### Re-enable a disabled rule

```
POST kbn:/api/alerting/rule/<rule-id>/_enable
```

### Disable a rule manually

```
POST kbn:/api/alerting/rule/<rule-id>/_disable
```

---

## Threshold Tuning

The default thresholds (`warning: >1s`, `critical: >5s`) suit a busy production cluster where legitimate Fleet Agent rules run at 60–130ms.

To change, edit the ES|QL query in the Workflow Step 1:

```
| EVAL
    max_duration_ms = max_duration_ns / 1000000,
    severity = CASE(max_duration_ns >= 5000000000, "critical", "warning")
    --                                 ^^^^^^^^^^
    --                         change to 2000000000 for 2s critical threshold
```

| Cluster type | Warning threshold | Critical threshold |
|---|---|---|
| Small / low data volume | 500ms | 2,000ms |
| Medium production | 1,000ms (default) | 5,000ms (default) |
| Large / high cardinality | 2,000ms | 10,000ms |

---

## Workflow Management

```
# View execution history
Stack Management → Workflows → OPS - Rogue Alert Detector → Executions tab

# Disable workflow (pause without deleting)
Stack Management → Workflows → toggle Enabled off

# Run manually
Stack Management → Workflows → Run button
```

---

## Known Limitations

| Limitation | Detail |
|---|---|
| System index deprecation | `.kibana-event-log*` direct access generates a deprecation warning in 9.x. Safe currently; Elastic will provide an official Event Log API in a future version |
| Numeric fields stored as strings | Workflow Liquid templating wraps all values in quotes. `detection.duration_ms` is stored as `"1101"` not `1101`. Does not affect dashboard display but breaks numeric aggregations on that field |
| 5-minute detection window | Rules that happen to run fast in a given window won't be detected that cycle. The next cycle will catch them if they're consistently slow |
| `.es-query` rules only | Other rule types not in scope. Extend by modifying the ES|QL filters |
| No auto-remediation for warnings | Warning-tier rules are email-notified only. Human review required before disabling |
| Critical auto-disable risk | A legitimately slow rule (e.g. a security detection scanning a large index) may be disabled. Add a `rule.tags` whitelist check before the disable step if needed |
| Elastic Cloud monitoring | Cluster CPU and node stats are stored in a separate monitoring deployment on Elastic Cloud. Use Stack Monitoring UI for CPU correlation. Task Manager drift from `.kibana-event-log*` is the available in-cluster signal |

---

## Files in This Repository

| File | Purpose |
|---|---|
| `README.md` | This document |
| `workflow.yaml` | Complete Elastic Workflow definition — detection, enrichment, auto-disable, case creation, email notification |
| `workflow_minimal.yaml` | Detection and write only — use for initial deployment and testing before adding remediation |
| `dashboard_payload.json` | Kibana dashboard saved object — replace data view IDs before deploying |
| `devtools_dashboard.txt` | Dev Console commands — Step 1 creates Lens object, Step 2 creates dashboard |
| `devtools_correlation_panels.txt` | Dev Console commands — creates Task Manager drift and Alert vs TM Lens objects |

---

## Test Cases

Create these rules in **Stack Management → Rules → Create rule → Elasticsearch query** to validate detection:

### Test 1 — Warning tier
- **Name:** `_TEST_ROGUE_Warning_Tier`
- **Index:** `logs-*` or `metrics-*`
- **Query:** empty (match_all)
- **Window:** 24 hours
- **Schedule:** 1 minute
- **Exclude previous runs:** unchecked

### Test 2 — Critical tier
- **Name:** `_TEST_ROGUE_Critical_Tier`
- **Index:** `logs-*` or `metrics-*`
- **Query:** empty (match_all)
- **Window:** 7 days
- **Schedule:** 1 minute
- **Exclude previous runs:** unchecked

### Test 3 — Control (should be slow but demonstrates duration-only detection)
- **Name:** `_TEST_Legit_Looking_Rule`
- **Tags:** `monitoring, ops`
- **Index:** `logs-*`
- **Query:** empty
- **Window:** 5 minutes
- **Schedule:** 1 minute
- **Exclude previous runs:** unchecked

Wait 5 minutes, run the Workflow manually, then verify all three appear in `rogue-alert-detections`. The legitimately-named Test 3 being detected proves detection is purely duration-based.

---

## Severity Reference

| Metric | Normal | Warning | Critical |
|---|---|---|---|
| Alert task duration | < 1s | 1–5s | > 5s |
| Task Manager p95 | < 500ms | 500ms–2s | > 2s |
| TM schedule delay | < 100ms | 100ms–1s | > 1s |
| Rule Success Ratio | 100% | < 100% | Dropping |

---

*Tested on Elasticsearch 9.4.1 / Kibana 9.4.1 — Elastic Cloud (us-east-2). Elastic Workflows GA from 9.4.*
