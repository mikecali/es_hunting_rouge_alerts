# 🔍 ES Query Rogue Alert Hunter

> Behavioural detection and monitoring for runaway Kibana `.es-query` alerting rules on Elasticsearch 9.x clusters.

---

## The Problem

Kibana alerting rules are invisible to the cluster until they cause damage. A single poorly-configured `.es-query` rule — running `match_all` over 24 hours of data every minute with `track_total_hits: 2147483647` — can consume enough search thread pool capacity to degrade every other query on the cluster. By the time you notice CPU saturation, the rule has been running for days.

The challenge: you cannot detect a rogue alert by looking at its name or tags. What matters is **how long it takes to execute**.

---

## How It Works

This solution detects rogue alerts **purely by execution behaviour**, not by metadata. It monitors the Kibana event log every 5 minutes and flags any `.es-query` rule whose execution time crosses a threshold — regardless of who created it, what it's named, or what tags it carries.

### Detection Signal

```
event.duration  >  1,000,000,000 ns  (1 second)
AND event.provider  =  "alerting"
AND event.action    =  "execute"
AND kibana.saved_objects.type_id  =  ".es-query"
```

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
│                                                        │               │
│                                                        ▼               │
│                                           ┌────────────────────────┐   │
│                                           │  .kibana-event-log*    │   │
│                                           │                        │   │
│                                           │  event.provider:       │   │
│                                           │    "alerting"          │   │
│                                           │  event.action:         │   │
│                                           │    "execute"           │   │
│                                           │  event.duration:       │   │
│                                           │    1,336,000,000 ns ◄──┼── rogue
│                                           │  kibana.saved_objects  │   │
│                                           │    .type_id:           │   │
│                                           │    ".es-query"         │   │
│                                           └────────────┬───────────┘   │
│                                                        │               │
└────────────────────────────────────────────────────────┼───────────────┘
                                                         │
                              ┌──────────────────────────▼──────────────┐
                              │      Watcher: ops-rogue-alert-hunter     │
                              │      Schedule: every 5 minutes           │
                              │                                          │
                              │  Step 1 — Query event log               │
                              │    event.duration > 1,000,000,000        │
                              │    AND type_id = ".es-query"             │
                              │    AND action  = "execute"               │
                              │    Window: last 5 minutes                │
                              │                                          │
                              │  Step 2 — Group by rule ID               │
                              │    max_duration per rule                 │
                              │    execution_count per rule              │
                              │                                          │
                              │  Step 3 — Enrich                        │
                              │    Lookup .kibana_alerting_cases_9*      │
                              │    → rule.name, rule.created_by          │
                              │    → rule.tags, rule.schedule            │
                              │    → detection.window_size_h             │
                              │    → detection.excludes_previous_runs    │
                              │                                          │
                              │  Step 4 — Classify severity              │
                              │    ≥ 5,000 ms  →  CRITICAL               │
                              │    ≥ 1,000 ms  →  WARNING                │
                              │                                          │
                              │  Step 5 — Write detection                │
                              │    → rogue-alert-detections index        │
                              └──────────────────────────┬──────────────┘
                                                         │
                              ┌──────────────────────────▼──────────────┐
                              │   rogue-alert-detections index           │
                              │                                          │
                              │  {                                       │
                              │    "rule.name": "Elasticsearch query..", │
                              │    "rule.created_by": "2763457433",      │
                              │    "rule.tags": ["rouge"],               │
                              │    "detection.duration_ms": 1336,        │
                              │    "detection.severity": "warning",      │
                              │    "detection.executions_in_window": 4   │
                              │  }                                       │
                              └──────────────────────────┬──────────────┘
                                                         │
                              ┌──────────────────────────▼──────────────┐
                              │   [OPS] ES Query Rogue Alert Hunter      │
                              │   Kibana Dashboard — auto-refresh 30s    │
                              │                                          │
                              │  • Detections over time (bar chart)      │
                              │  • Worst single execution (ms metric)    │
                              │  • Total / Critical / Warning counts     │
                              │  • Current offenders table               │
                              │    (name, creator, tags, severity)       │
                              │  • All rule duration histogram           │
                              │  • Rule health scorecard                 │
                              └─────────────────────────────────────────┘
```

### Why Behavioural Detection?

| Detection approach | Catches renamed rules | Catches rules with clean tags | Catches new accounts | Catches rules that look legitimate |
|---|---|---|---|---|
| Tag-based (`_DIAG_TEMP`) | ❌ | ❌ | ❌ | ❌ |
| Name-based pattern match | ❌ | ❌ | ❌ | ❌ |
| Creator-based allowlist | ❌ | ❌ | ❌ | ❌ |
| **Duration-based (this solution)** | ✅ | ✅ | ✅ | ✅ |

The `_TEST_Legit_Looking_Rule` test case validated this — a rule with tags `monitoring, ops`, created by a known account, with a reasonable-sounding name was still detected because its execution time crossed 1 second.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  INPUTS (read-only)                                          │
│                                                              │
│  .kibana-event-log*          — rule execution history        │
│  .kibana_alerting_cases_9*   — rule metadata (name, tags)    │
└──────────────────────────┬───────────────────────────────────┘
                           │  read only
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  DETECTION ENGINE                                            │
│                                                              │
│  Elasticsearch Watcher                                       │
│  ID: ops-rogue-alert-hunter                                  │
│  Schedule: every 5 minutes                                   │
│  Impact: ~30–120ms per run                                   │
└──────────────────────────┬───────────────────────────────────┘
                           │  writes
                           ▼
┌──────────────────────────────────────────────────────────────┐
│  OUTPUT                                                      │
│                                                              │
│  rogue-alert-detections index                                │
│  1 document per offending rule per Watcher run               │
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
│  All legitimate rules    │  Broad query, large   │  match_all over days,
│  on your cluster run     │  time window, or      │  no filter, full
│  under 1 second.         │  no filter applied.   │  shard scan.
│  Fleet Agent rules:      │  Investigate.         │  Immediate action.
│  60–130ms typical.       │                       │
```

---

## What Gets Detected (and What Doesn't)

### Detected ✅
- `.es-query` rules with `match_all` and large time windows
- Rules with `excludeHitsFromPreviousRun: false` causing full re-scans
- Rules with `track_total_hits: 2147483647` (Kibana default for `.es-query`)
- Rules scanning broad index patterns (`logs-*`, `metrics-*`) without filters
- Rules on tight schedules (1m) with wide windows (24h+)
- Any rule from any account — creator is irrelevant to detection

### Not Detected ❌
- Rules that run fast but generate excessive actions (action flood)
- Other rule types (`.index-threshold`, `slo.rules.burnRate`, ML rules)
- Rules that are slow due to cluster degradation rather than query design
- Indexing pressure or mapping issues

> To extend detection to other rule types, change `kibana.saved_objects.type_id: ".es-query"` in the Watcher input query to `"*"` or add additional type IDs.

---

## Production Deployment

### Prerequisites

| Requirement | Detail |
|---|---|
| Elasticsearch | 9.x (tested on 9.2.2) |
| Kibana | 9.x |
| Watcher | Requires Platinum or Enterprise licence |
| User role | `superuser` or `cluster:monitor`, `indices:admin/read` on `.kibana*` |

### Step 1 — Check cluster shard headroom

Before creating anything, verify you have at least 2 free shards:

```
GET _cluster/stats?filter_path=indices.shards.total
```

If at or near 5000, clean up unused indices before proceeding. Do not set `cluster.max_shards_per_node` higher without understanding the memory impact.

### Step 2 — Create the detections index

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
      "detection.duration_s":               { "type": "float" },
      "detection.severity":                 { "type": "keyword" },
      "detection.threshold_ms":             { "type": "long" },
      "detection.execution_time":           { "type": "date" },
      "detection.executions_in_window":     { "type": "integer" },
      "detection.window_size_h":            { "type": "integer" },
      "detection.excludes_previous_runs":   { "type": "boolean" },
      "detection.result_size":              { "type": "integer" }
    }
  }
}
```

Verify:
```
GET rogue-alert-detections/_mapping
```

### Step 3 — Deploy the Watcher

> **Important:** The Watcher reads `.kibana-event-log*` (system index). In ES 9.x this generates a deprecation warning in logs. This is safe and expected. In a future major version, use the Kibana Event Log API instead.

```json
PUT _watcher/watch/ops-rogue-alert-hunter
{
  "trigger": {
    "schedule": { "interval": "5m" }
  },
  "input": {
    "chain": {
      "inputs": [
        {
          "executions": {
            "search": {
              "request": {
                "indices": [".kibana-event-log*"],
                "body": {
                  "size": 0,
                  "query": {
                    "bool": {
                      "filter": [
                        { "term": { "event.provider": "alerting" } },
                        { "term": { "event.action": "execute" } },
                        { "range": { "@timestamp": { "gte": "now-5m" } } },
                        { "range": { "event.duration": { "gt": 1000000000 } } }
                      ],
                      "must": [
                        {
                          "nested": {
                            "path": "kibana.saved_objects",
                            "query": {
                              "term": { "kibana.saved_objects.type_id": ".es-query" }
                            }
                          }
                        }
                      ]
                    }
                  },
                  "aggs": {
                    "by_rule": {
                      "nested": { "path": "kibana.saved_objects" },
                      "aggs": {
                        "rule_id": {
                          "terms": { "field": "kibana.saved_objects.id", "size": 50 },
                          "aggs": {
                            "back_to_root": {
                              "reverse_nested": {},
                              "aggs": {
                                "max_duration": { "max": { "field": "event.duration" } },
                                "execution_count": { "value_count": { "field": "event.duration" } },
                                "last_execution": { "max": { "field": "@timestamp" } }
                              }
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        },
        {
          "rule_metadata": {
            "search": {
              "request": {
                "indices": [".kibana_alerting_cases_9*"],
                "body": {
                  "size": 100,
                  "query": {
                    "bool": {
                      "must": [
                        { "term": { "type": "alert" } },
                        { "term": { "alert.alertTypeId": ".es-query" } }
                      ]
                    }
                  },
                  "_source": [
                    "alert.name", "alert.tags", "alert.createdBy",
                    "alert.createdAt", "alert.schedule",
                    "alert.params.timeWindowSize", "alert.params.timeWindowUnit",
                    "alert.params.size", "alert.params.excludeHitsFromPreviousRun"
                  ]
                }
              }
            }
          }
        }
      ]
    }
  },
  "condition": {
    "script": {
      "source": "return ctx.payload.executions.aggregations.by_rule.rule_id.buckets.size() > 0"
    }
  },
  "transform": {
    "script": {
      "source": """
        def docs = [];
        def metaHits = ctx.payload.rule_metadata.hits.hits;
        for (def bucket : ctx.payload.executions.aggregations.by_rule.rule_id.buckets) {
          def ruleId = bucket.key;
          def maxDurationNs = bucket.back_to_root.max_duration.value;
          def maxDurationMs = (long)(maxDurationNs / 1000000);
          def maxDurationS  = maxDurationNs / 1000000000.0;
          def execCount     = bucket.back_to_root.execution_count.value;
          def lastExec      = bucket.back_to_root.last_execution.value_as_string;
          def severity      = maxDurationMs >= 5000 ? 'critical' : 'warning';
          def ruleName    = ruleId;
          def createdBy   = 'unknown';
          def createdAt   = null;
          def tags        = [];
          def schedule    = '1m';
          def windowSize  = 0;
          def windowUnit  = 'h';
          def resultSize  = 0;
          def excludePrev = false;
          for (def hit : metaHits) {
            def hitId = hit._id.replace('alert:', '');
            if (hitId == ruleId) {
              def a = hit._source.alert;
              ruleName    = a.name;
              createdBy   = a.containsKey('createdBy')  ? a.createdBy  : 'unknown';
              createdAt   = a.containsKey('createdAt')  ? a.createdAt  : null;
              tags        = a.containsKey('tags')        ? a.tags       : [];
              schedule    = a.containsKey('schedule')    ? a.schedule.interval : '1m';
              if (a.containsKey('params')) {
                windowSize  = a.params.containsKey('timeWindowSize') ? a.params.timeWindowSize : 0;
                windowUnit  = a.params.containsKey('timeWindowUnit') ? a.params.timeWindowUnit : 'h';
                resultSize  = a.params.containsKey('size')           ? a.params.size           : 0;
                excludePrev = a.params.containsKey('excludeHitsFromPreviousRun') ? a.params.excludeHitsFromPreviousRun : false;
              }
              break;
            }
          }
          def doc = [
            '@timestamp':                       ctx.execution_time,
            'rule.id':                          ruleId,
            'rule.name':                        ruleName,
            'rule.type':                        '.es-query',
            'rule.tags':                        tags,
            'rule.created_by':                  createdBy,
            'rule.schedule':                    schedule,
            'detection.duration_ms':            maxDurationMs,
            'detection.duration_s':             maxDurationS,
            'detection.severity':               severity,
            'detection.threshold_ms':           severity == 'critical' ? 5000 : 1000,
            'detection.execution_time':         lastExec,
            'detection.executions_in_window':   execCount,
            'detection.window_size_h':          windowSize,
            'detection.excludes_previous_runs': excludePrev,
            'detection.result_size':            resultSize
          ];
          if (createdAt != null) { doc['rule.created_at'] = createdAt; }
          docs.add(doc);
        }
        return [ '_doc': docs ];
      """
    }
  },
  "actions": {
    "index_detections": {
      "foreach": "ctx.payload._doc",
      "index": {
        "index": "rogue-alert-detections"
      }
    },
    "log_detections": {
      "logging": {
        "text": "[OPS] Rogue alert detected: {{ctx.payload._doc}}"
      }
    }
  }
}
```

### Step 4 — Smoke test the Watcher

Trigger it manually and confirm it executes without errors:

```json
POST _watcher/watch/ops-rogue-alert-hunter/_execute
{
  "ignore_condition": false,
  "action_modes": {
    "index_detections": "execute",
    "log_detections": "execute"
  }
}
```

Confirm detections landed:

```json
GET rogue-alert-detections/_search
{
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 5
}
```

If no results, your cluster has no `.es-query` rules exceeding 1 second — that is a healthy result. Verify by checking the event log directly:

```json
GET .kibana-event-log*/_search
{
  "size": 1,
  "query": {
    "bool": {
      "filter": [
        { "term": { "event.provider": "alerting" } },
        { "term": { "event.action": "execute" } },
        { "range": { "event.duration": { "gt": 1000000000 } } }
      ]
    }
  }
}
```

### Step 5 — Create Kibana data views

Run in Kibana Dev Console:

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

Copy both `id` values from the responses — you need them for Step 6.

### Step 6 — Deploy the dashboard

Replace `<DETECTIONS_DV_ID>` and `<EVENTLOG_DV_ID>` with your data view IDs from Step 5, then run:

```json
POST kbn:/api/saved_objects/dashboard/ops-rogue-alert-hunter-dash
{
  "attributes": {
    "title": "[OPS] ES Query Rogue Alert Hunter",
    "description": "Behavioural detection for rogue .es-query Kibana rules. Powered by ops-rogue-alert-hunter Watcher (runs every 5m).",
    "timeRestore": true,
    "timeFrom": "now-24h",
    "timeTo": "now",
    "refreshInterval": { "pause": false, "value": 30000 },
    "kibanaSavedObjectMeta": {
      "searchSourceJSON": "{\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"filter\":[]}"
    },
    "optionsJSON": "{\"useMargins\":true,\"syncColors\":false,\"hidePanelTitles\":false}"
  },
  "references": []
}
```

> **Note:** For the full panel configuration, see [`dashboard_payload.json`](./dashboard_payload.json) in this repository. The full payload includes 8 panels with pre-wired aggregations. Replace the two data view ID placeholders before importing.

### Step 7 — Open the dashboard

Navigate to:
```
https://<your-kibana>/app/dashboards#/view/ops-rogue-alert-hunter-dash
```

Or: **Kibana → Dashboards → search `[OPS] ES Query Rogue Alert Hunter`**

---

## Operational Queries

### Who are the current offenders? (last 24h)

```json
GET rogue-alert-detections/_search
{
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "query": { "range": { "@timestamp": { "gte": "now-24h" } } },
  "aggs": {
    "by_rule": {
      "terms": { "field": "rule.id", "size": 20 },
      "aggs": {
        "rule_name":    { "terms": { "field": "rule.name", "size": 1 } },
        "created_by":   { "terms": { "field": "rule.created_by", "size": 1 } },
        "max_duration": { "max": { "field": "detection.duration_ms" } },
        "detections":   { "value_count": { "field": "detection.severity" } }
      }
    }
  },
  "size": 0
}
```

### Duration trend for a specific rule (replace RULE_ID)

```json
GET .kibana-event-log*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "event.provider": "alerting" } },
        { "term": { "event.action": "execute" } },
        { "range": { "@timestamp": { "gte": "now-6h" } } },
        {
          "nested": {
            "path": "kibana.saved_objects",
            "query": { "term": { "kibana.saved_objects.id": "RULE_ID" } }
          }
        }
      ]
    }
  },
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "5m" },
      "aggs": {
        "max_duration_ms": { "max": { "script": "doc['event.duration'].value / 1000000" } },
        "avg_duration_ms": { "avg": { "script": "doc['event.duration'].value / 1000000" } }
      }
    }
  }
}
```

### Full rule health scorecard (all .es-query rules, last 24h)

```json
GET .kibana-event-log*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "event.provider": "alerting" } },
        { "term": { "event.action": "execute" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } },
        {
          "nested": {
            "path": "kibana.saved_objects",
            "query": { "term": { "kibana.saved_objects.type_id": ".es-query" } }
          }
        }
      ]
    }
  },
  "aggs": {
    "by_rule": {
      "nested": { "path": "kibana.saved_objects" },
      "aggs": {
        "rule_id": {
          "terms": { "field": "kibana.saved_objects.id", "size": 50 },
          "aggs": {
            "metrics": {
              "reverse_nested": {},
              "aggs": {
                "p95_ms": { "percentiles": { "script": "doc['event.duration'].value / 1000000", "percents": [50, 95] } },
                "max_ms": { "max": { "script": "doc['event.duration'].value / 1000000" } },
                "count":  { "value_count": { "field": "event.duration" } }
              }
            }
          }
        }
      }
    }
  }
}
```

### Cancel a running task (emergency)

```
POST _tasks/<node-id>:<task-id>/_cancel
```

Get the task ID from:
```
GET _tasks?actions=indices:data/read/search&detailed=true&group_by=parents
```

Look for tasks with `X-Opaque-Id` containing `alerting:.es-query`.

### Disable a specific rule (by ID)

```
POST kbn:/api/alerting/rule/<rule-id>/disable
```

---

## Threshold Tuning

The default thresholds (`warning: >1s`, `critical: >5s`) are calibrated for a busy production cluster where legitimate Fleet Agent rules run at 60–130ms. Tune based on your cluster:

| Cluster type | Recommended warning threshold | Recommended critical threshold |
|---|---|---|
| Small / low data volume | 500ms | 2,000ms |
| Medium production | 1,000ms (default) | 5,000ms (default) |
| Large / high cardinality | 2,000ms | 10,000ms |

To change thresholds, update the Watcher transform script:

```painless
// Change these two values:
def severity = maxDurationMs >= 5000 ? 'critical' : 'warning';
//                             ^^^^
// and in the doc:
'detection.threshold_ms': severity == 'critical' ? 5000 : 1000,
//                                                 ^^^^    ^^^^
```

Then re-PUT the Watcher with the updated script.

---

## Watcher Management

```bash
# Check Watcher status
GET _watcher/watch/ops-rogue-alert-hunter

# Trigger manually
POST _watcher/watch/ops-rogue-alert-hunter/_execute
{ "ignore_condition": false, "action_modes": { "index_detections": "execute", "log_detections": "execute" } }

# Pause (deactivate) without deleting
POST _watcher/watch/ops-rogue-alert-hunter/_deactivate

# Resume
POST _watcher/watch/ops-rogue-alert-hunter/_activate

# Remove entirely
DELETE _watcher/watch/ops-rogue-alert-hunter
```

---

## Known Limitations

| Limitation | Detail |
|---|---|
| System index deprecation | `.kibana-event-log*` is a system index. Direct access generates a deprecation warning in ES 9.x logs. Safe in 9.x; migrate to Kibana Event Log API in future major versions |
| 5-minute detection window | Rules that complete before the Watcher runs may be missed in a single cycle. Use the `.kibana-event-log*` historical queries for full coverage |
| No auto-remediation | The Watcher detects and reports — it never disables or cancels rules. Human review is required before any action. This is intentional |
| `.es-query` rules only | Other rule types (`slo.rules.burnRate`, `.index-threshold`, ML rules) are not in scope. Extend by modifying the `type_id` filter |
| Watcher licence | Requires Elastic Platinum or Enterprise licence |

---

## Files in This Repository

| File | Purpose |
|---|---|
| `README.md` | This document |
| `dashboard_payload.json` | Full Kibana dashboard saved object (8 panels, replace data view IDs before import) |
| `watcher.json` | Standalone Watcher definition for CI/CD or IaC deployment |

---

## Test Cases

To validate detection after deployment, create these rules in **Stack Management → Rules → Create rule → Elasticsearch query**:

### Test 1 — Warning tier
- **Name:** `_TEST_ROGUE_Warning_Tier`
- **Index:** `logs-*`
- **Query:** empty (match_all)
- **Window:** 24 hours
- **Schedule:** 1 minute
- **Exclude previous runs:** unchecked

### Test 2 — Critical tier
- **Name:** `_TEST_ROGUE_Critical_Tier`
- **Index:** `logs-*`
- **Query:** empty (match_all)
- **Window:** 7 days
- **Schedule:** 1 minute
- **Exclude previous runs:** unchecked

### Test 3 — Should NOT trigger (control)
- **Name:** `_TEST_Legit_Control`
- **Index:** `logs-*`
- **Query:** specific filter (not empty)
- **Window:** 5 minutes
- **Schedule:** 5 minutes
- **Exclude previous runs:** checked

Wait 5 minutes, trigger the Watcher manually, then verify Test 1 and 2 appear in `rogue-alert-detections` and Test 3 does not.

> **Note:** In practice, even a narrow-window rule can appear if the underlying index is large enough. Duration is an honest signal — if it's slow, it's consuming resources.

---

## Severity Reference

| Metric | Normal | Warning | Critical |
|---|---|---|---|
| Alert task duration | < 1s | 1–5s | > 5s |
| Task Manager drift (p99) | < 1s | 1–5s | > 10s |
| Search thread pool queue | 0 | 1–10 | > 10 |
| Search rejections (cumulative) | 0 | Any | Increasing |

---

*Tested on Elasticsearch 9.2.2 / Kibana 9.2.2. Compatible with 9.x.*

