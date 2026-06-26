# [OPS] ES Query Rogue Alert Hunter — Complete Setup
# Elasticsearch 9.2.2 | Kibana 9.2.2
# Run each step in order in Kibana Dev Console

# ─────────────────────────────────────────────────────────────────────
# STEP 1 — Create the rogue-alert-detections index
# ─────────────────────────────────────────────────────────────────────

PUT rogue-alert-detections
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp":       { "type": "date" },
      "rule.id":          { "type": "keyword" },
      "rule.name":        { "type": "keyword" },
      "rule.type":        { "type": "keyword" },
      "rule.tags":        { "type": "keyword" },
      "rule.created_by":  { "type": "keyword" },
      "rule.created_at":  { "type": "date" },
      "rule.schedule":    { "type": "keyword" },
      "rule.index":       { "type": "keyword" },
      "detection.duration_ms":    { "type": "long" },
      "detection.duration_s":     { "type": "float" },
      "detection.severity":       { "type": "keyword" },
      "detection.threshold_ms":   { "type": "long" },
      "detection.execution_time": { "type": "date" },
      "detection.window_size_h":  { "type": "integer" },
      "detection.excludes_previous_runs": { "type": "boolean" },
      "detection.result_size":    { "type": "integer" }
    }
  }
}

# ─────────────────────────────────────────────────────────────────────
# STEP 2 — Verify index was created
# ─────────────────────────────────────────────────────────────────────

GET rogue-alert-detections/_mapping

# ─────────────────────────────────────────────────────────────────────
# STEP 3 — Create the Elasticsearch Index Connector via Kibana API
#
# Run this in Dev Console (it calls the Kibana API, not ES directly)
# ─────────────────────────────────────────────────────────────────────

POST kbn:/api/actions/connector
{
  "name": "[OPS] Rogue Alert Detections Writer",
  "connector_type_id": ".index",
  "config": {
    "index": "rogue-alert-detections",
    "refresh": true,
    "executionTimeField": "@timestamp"
  }
}

# IMPORTANT: Copy the "id" from the response above.
# You will need it in Step 5 when creating the meta-alert action.
# It will look like: "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# ─────────────────────────────────────────────────────────────────────
# STEP 4 — Verify connector works (optional but recommended)
# Replace CONNECTOR_ID with the id from Step 3
# ─────────────────────────────────────────────────────────────────────

POST kbn:/api/actions/connector/CONNECTOR_ID/_execute
{
  "params": {
    "documents": [
      {
        "@timestamp": "2026-06-26T00:00:00.000Z",
        "rule.id": "test-verify-connector",
        "rule.name": "Connector verification test",
        "rule.type": ".es-query",
        "detection.duration_ms": 9999,
        "detection.duration_s": 9.999,
        "detection.severity": "test",
        "detection.threshold_ms": 1000,
        "detection.execution_time": "2026-06-26T00:00:00.000Z"
      }
    ]
  }
}

# Then verify it landed:
GET rogue-alert-detections/_search
# Should return 1 hit with rule.id = "test-verify-connector"
# Clean it up after:
POST rogue-alert-detections/_delete_by_query
{
  "query": { "term": { "rule.id": "test-verify-connector" } }
}

# ─────────────────────────────────────────────────────────────────────
# STEP 5 — Create the meta-alert via Kibana API
#
# BEFORE RUNNING: Replace CONNECTOR_ID with the id from Step 3
#
# This alert:
#   - Runs every 5 minutes
#   - Queries .kibana-event-log* for .es-query rule executions > 1s
#   - Groups results by rule ID
#   - Fires WARNING at > 1s, CRITICAL at > 5s
#   - Writes each detection to rogue-alert-detections via the connector
#   - Enriches each detection with rule metadata from alerting cases index
# ─────────────────────────────────────────────────────────────────────

POST kbn:/api/alerting/rule
{
  "name": "[OPS] ES Query Rogue Alert Hunter",
  "rule_type_id": ".es-query",
  "consumer": "stackAlerts",
  "tags": ["ops", "rogue-alert-hunter", "es-query-monitoring"],
  "schedule": { "interval": "5m" },
  "params": {
    "searchType": "esQuery",
    "index": [".kibana-event-log*"],
    "timeField": "@timestamp",
    "esQuery": "{\"query\":{\"bool\":{\"filter\":[{\"term\":{\"event.provider\":\"alerting\"}},{\"term\":{\"event.action\":\"execute\"}},{\"term\":{\"kibana.saved_objects.type_id\":\".es-query\"}},{\"range\":{\"event.duration\":{\"gt\":1000000000}}}]}}}",
    "timeWindowSize": 5,
    "timeWindowUnit": "m",
    "threshold": [0],
    "thresholdComparator": ">",
    "size": 100,
    "excludeHitsFromPreviousRun": true,
    "aggType": "count",
    "groupBy": "all"
  },
  "actions": [
    {
      "id": "CONNECTOR_ID",
      "group": "query matched",
      "params": {
        "documents": [
          {
            "@timestamp": "{{context.date}}",
            "rule.id": "{{context.value}}",
            "rule.name": "{{rule.name}}",
            "rule.type": ".es-query",
            "detection.duration_ms": "{{context.hits.[0].event.duration}}",
            "detection.severity": "warning",
            "detection.threshold_ms": 1000,
            "detection.execution_time": "{{context.date}}"
          }
        ]
      }
    }
  ]
}

# ─────────────────────────────────────────────────────────────────────
# STEP 6 — Better approach: Use a custom ES|QL / scripted approach
#
# Since the meta-alert above uses the standard .es-query rule which
# can't dynamically enrich per-offender, use this Watcher instead.
# Watcher gives us full scripted enrichment per matching rule ID.
#
# This Watcher:
#   - Runs every 5 minutes
#   - Finds all .es-query executions > 1s in the last 5 minutes
#   - Groups by rule ID, gets max duration per rule
#   - For each offender, looks up rule metadata (name, creator, tags)
#   - Writes one enriched document per offender to rogue-alert-detections
#   - Assigns WARNING (>1s) or CRITICAL (>5s) severity
# ─────────────────────────────────────────────────────────────────────

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
                          "terms": {
                            "field": "kibana.saved_objects.id",
                            "size": 50
                          },
                          "aggs": {
                            "back_to_root": {
                              "reverse_nested": {},
                              "aggs": {
                                "max_duration": {
                                  "max": { "field": "event.duration" }
                                },
                                "execution_count": {
                                  "value_count": { "field": "event.duration" }
                                },
                                "last_execution": {
                                  "max": { "field": "@timestamp" }
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
                    "alert.name",
                    "alert.tags",
                    "alert.createdBy",
                    "alert.createdAt",
                    "alert.schedule",
                    "alert.params.timeWindowSize",
                    "alert.params.timeWindowUnit",
                    "alert.params.size",
                    "alert.params.excludeHitsFromPreviousRun"
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
            '@timestamp':                         ZonedDateTime.now(ZoneOffset.UTC).toString(),
            'rule.id':                            ruleId,
            'rule.name':                          ruleName,
            'rule.type':                          '.es-query',
            'rule.tags':                          tags,
            'rule.created_by':                    createdBy,
            'rule.schedule':                      schedule,
            'detection.duration_ms':              maxDurationMs,
            'detection.duration_s':               maxDurationS,
            'detection.severity':                 severity,
            'detection.threshold_ms':             severity == 'critical' ? 5000 : 1000,
            'detection.execution_time':           lastExec,
            'detection.executions_in_window':     execCount,
            'detection.window_size_h':            windowSize,
            'detection.excludes_previous_runs':   excludePrev,
            'detection.result_size':              resultSize
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

# ─────────────────────────────────────────────────────────────────────
# STEP 7 — Verify Watcher was created and trigger it manually
# ─────────────────────────────────────────────────────────────────────

GET _watcher/watch/ops-rogue-alert-hunter

# Trigger it manually to test (simulates a run right now)
POST _watcher/watch/ops-rogue-alert-hunter/_execute
{
  "ignore_condition": false,
  "action_modes": {
    "index_detections": "execute",
    "log_detections": "execute"
  }
}

# Verify detections landed
GET rogue-alert-detections/_search
{
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 10
}

# ─────────────────────────────────────────────────────────────────────
# STEP 8 — Dashboard setup
#
# Import the dashboard JSON via:
# Stack Management → Saved Objects → Import
# Use the file: ops_rogue_alert_hunter_dashboard.ndjson
#
# The dashboard contains 6 panels:
#   1. Rogue Detections Over Time (bar chart — warning vs critical)
#   2. Current Offenders Table (rule name, creator, max duration, breach count)
#   3. All Rule Duration Histogram (the outlier bucket view)
#   4. Duration Trend Per Rule (line chart — last 6h)
#   5. Worst Single Execution (metric panel)
#   6. Rule Health Scorecard (p50/p95, executions, status)
# ─────────────────────────────────────────────────────────────────────

# ─────────────────────────────────────────────────────────────────────
# STEP 9 — Useful operational queries (save these in Dev Console)
# ─────────────────────────────────────────────────────────────────────

# 9a. Who are the current offenders right now?
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
        "max_duration": { "max":   { "field": "detection.duration_ms" } },
        "detections":   { "value_count": { "field": "detection.severity" } },
        "worst_severity": { "terms": { "field": "detection.severity", "size": 1,
                            "order": { "_key": "desc" } } }
      }
    }
  },
  "size": 0
}

# 9b. Duration trend for a specific rule (last 6h)
# Replace RULE_ID with the actual rule ID
GET .kibana-event-log*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "event.provider": "alerting" } },
        { "term":  { "event.action":   "execute"  } },
        { "range": { "@timestamp": { "gte": "now-6h" } } },
        { "nested": {
            "path": "kibana.saved_objects",
            "query": { "term": { "kibana.saved_objects.id": "RULE_ID" } }
        }}
      ]
    }
  },
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "5m"
      },
      "aggs": {
        "max_duration_ms": { "max":    { "script": "doc['event.duration'].value / 1000000" } },
        "avg_duration_ms": { "avg":    { "script": "doc['event.duration'].value / 1000000" } },
        "p95_duration_ms": { "percentiles": { "script": "doc['event.duration'].value / 1000000",
                             "percents": [50, 95] } }
      }
    }
  }
}

# 9c. Full rule health scorecard — all .es-query rules last 24h
GET .kibana-event-log*/_search
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term":  { "event.provider": "alerting" } },
        { "term":  { "event.action":   "execute"  } },
        { "range": { "@timestamp": { "gte": "now-24h" } } },
        { "nested": {
            "path": "kibana.saved_objects",
            "query": { "term": { "kibana.saved_objects.type_id": ".es-query" } }
        }}
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
                "p50_ms": { "percentiles": { "script": "doc['event.duration'].value / 1000000",
                            "percents": [50, 95, 99] } },
                "max_ms": { "max": { "script": "doc['event.duration'].value / 1000000" } },
                "count":  { "value_count": { "field": "event.duration" } },
                "failures": {
                  "filter": { "term": { "event.outcome": "failure" } }
                }
              }
            }
          }
        }
      }
    }
  }
}

# 9d. Stop the Watcher (when done testing)
POST _watcher/watch/ops-rogue-alert-hunter/_deactivate

# Reactivate it
POST _watcher/watch/ops-rogue-alert-hunter/_activate
