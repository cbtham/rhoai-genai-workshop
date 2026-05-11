# vLLM Dashboard in OpenShift Console

## Overview

This guide sets up a vLLM performance dashboard in the OpenShift Console under **Observe > Dashboards**. The dashboard includes a model dropdown filter and displays:

- Time to First Token (TTFT) - P50/P95/P99
- Inter-Token Latency
- GPU Cache Usage
- Request Queue Status
- Token Throughput
- End-to-End Request Latency
- Active Requests across all models

## Prerequisites

- OpenShift 4.x cluster with admin access
- User workload monitoring enabled
- vLLM inference services exposing Prometheus metrics (via ServiceMonitor)
- Metrics available with `vllm:` prefix (e.g. `vllm:num_requests_running`, `vllm:time_to_first_token_seconds_bucket`)

### Enable user workload monitoring (if not already)

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

## Setup

### Step 1: Create the dashboard ConfigMap

The OpenShift Console renders dashboards from ConfigMaps in `openshift-config-managed` with the label `console.openshift.io/dashboard: "true"`.

Key format requirements:
- Must use `rows`-based layout (not root-level `panels`)
- Panel types: `graph`, `singlestat`, `table` (NOT `timeseries`, `gauge`, `stat`)
- Must define a `$datasource` template variable of type `datasource`
- Query-type template variables (like model filter) must follow the same structure as built-in k8s dashboards
- `schemaVersion: 14`

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-vllm-advanced
  namespace: openshift-config-managed
  labels:
    console.openshift.io/dashboard: "true"
data:
  vllm-advanced-performance.json: |
    {
        "annotations": {
            "list": []
        },
        "editable": true,
        "gnetId": null,
        "graphTooltip": 1,
        "hideControls": false,
        "links": [],
        "refresh": "30s",
        "rows": [
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 1,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "histogram_quantile(0.50, rate(vllm:time_to_first_token_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P50 TTFT"
                            },
                            {
                                "expr": "histogram_quantile(0.95, rate(vllm:time_to_first_token_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P95 TTFT"
                            },
                            {
                                "expr": "histogram_quantile(0.99, rate(vllm:time_to_first_token_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P99 TTFT"
                            }
                        ],
                        "title": "Time to First Token (TTFT)",
                        "type": "graph",
                        "yaxes": [
                            {"format": "s", "label": "Time (seconds)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true,
                            "avg": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 2,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "histogram_quantile(0.50, rate(vllm:time_per_output_token_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P50 Inter-Token"
                            },
                            {
                                "expr": "histogram_quantile(0.95, rate(vllm:time_per_output_token_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P95 Inter-Token"
                            }
                        ],
                        "title": "Inter-Token Latency",
                        "type": "graph",
                        "yaxes": [
                            {"format": "s", "label": "Time (seconds)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true,
                            "avg": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Latency Metrics"
            },
            {
                "collapse": false,
                "height": "200px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 3,
                        "span": 4,
                        "targets": [
                            {
                                "expr": "sum(vllm:gpu_cache_usage_perc{model_name=\"$model\"}) * 100",
                                "format": "time_series",
                                "legendFormat": "GPU Cache Usage %"
                            }
                        ],
                        "title": "GPU Cache Usage",
                        "type": "singlestat",
                        "valueName": "current",
                        "valueFontSize": "80%",
                        "format": "percent",
                        "thresholds": "60,80",
                        "colorBackground": false,
                        "colorValue": true,
                        "colors": ["#299c46", "#ed8128", "#d44a3a"],
                        "sparkline": {
                            "show": true,
                            "full": false,
                            "lineColor": "#3B99FC",
                            "fillColor": "rgba(59, 153, 252, 0.15)"
                        },
                        "gauge": {
                            "show": true,
                            "minValue": 0,
                            "maxValue": 100,
                            "thresholdLabels": false,
                            "thresholdMarkers": true
                        }
                    },
                    {
                        "datasource": "$datasource",
                        "id": 4,
                        "span": 4,
                        "targets": [
                            {
                                "expr": "sum(vllm:num_requests_running{model_name=\"$model\"})",
                                "format": "time_series",
                                "legendFormat": "Running"
                            },
                            {
                                "expr": "sum(vllm:num_requests_waiting{model_name=\"$model\"})",
                                "format": "time_series",
                                "legendFormat": "Waiting"
                            },
                            {
                                "expr": "sum(vllm:num_requests_swapped{model_name=\"$model\"})",
                                "format": "time_series",
                                "legendFormat": "Swapped"
                            }
                        ],
                        "title": "Request Queue Status",
                        "type": "graph",
                        "yaxes": [
                            {"format": "short", "label": "Request Count", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 5,
                        "span": 4,
                        "targets": [
                            {
                                "expr": "rate(vllm:prompt_tokens_total{model_name=\"$model\"}[5m])",
                                "format": "time_series",
                                "legendFormat": "Prompt Tokens/s"
                            },
                            {
                                "expr": "rate(vllm:generation_tokens_total{model_name=\"$model\"}[5m])",
                                "format": "time_series",
                                "legendFormat": "Generation Tokens/s"
                            }
                        ],
                        "title": "Token Throughput",
                        "type": "graph",
                        "yaxes": [
                            {"format": "short", "label": "Tokens/sec", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true,
                            "avg": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Resource Usage & Throughput"
            },
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 6,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "histogram_quantile(0.50, rate(vllm:e2e_request_latency_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P50 E2E Latency"
                            },
                            {
                                "expr": "histogram_quantile(0.95, rate(vllm:e2e_request_latency_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P95 E2E Latency"
                            },
                            {
                                "expr": "histogram_quantile(0.99, rate(vllm:e2e_request_latency_seconds_bucket{model_name=\"$model\"}[5m]))",
                                "format": "time_series",
                                "legendFormat": "P99 E2E Latency"
                            }
                        ],
                        "title": "End-to-End Request Latency",
                        "type": "graph",
                        "yaxes": [
                            {"format": "s", "label": "Time (seconds)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true,
                            "avg": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 7,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "sum(vllm:num_requests_running) by (model_name)",
                                "format": "time_series",
                                "legendFormat": "{{model_name}}"
                            }
                        ],
                        "title": "Active Requests (All Models)",
                        "type": "graph",
                        "yaxes": [
                            {"format": "short", "label": "Requests", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {
                            "show": true,
                            "values": true,
                            "current": true
                        },
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "E2E Latency & Model Activity"
            }
        ],
        "schemaVersion": 14,
        "style": "dark",
        "tags": ["vllm", "inference", "llm"],
        "templating": {
            "list": [
                {
                    "current": {
                        "text": "default",
                        "value": "default"
                    },
                    "hide": 0,
                    "label": "Data source",
                    "name": "datasource",
                    "options": [],
                    "query": "prometheus",
                    "refresh": 1,
                    "regex": "",
                    "type": "datasource"
                },
                {
                    "allValue": null,
                    "current": {
                        "text": "",
                        "value": ""
                    },
                    "datasource": "$datasource",
                    "hide": 0,
                    "includeAll": false,
                    "label": "Model",
                    "multi": false,
                    "name": "model",
                    "options": [],
                    "query": "label_values(vllm:num_requests_running, model_name)",
                    "refresh": 2,
                    "regex": "",
                    "sort": 1,
                    "tagValuesQuery": "",
                    "tags": [],
                    "tagsQuery": "",
                    "type": "query",
                    "useTags": false
                }
            ]
        },
        "time": {
            "from": "now-3h",
            "to": "now"
        },
        "timepicker": {
            "refresh_intervals": ["5s", "10s", "30s", "1m", "5m"]
        },
        "timezone": "",
        "title": "vLLM Advanced Performance",
        "version": 1
    }
EOF
```

### Step 2: Verify

```bash
oc get configmap grafana-dashboard-vllm-advanced -n openshift-config-managed -l console.openshift.io/dashboard=true
```

### Step 3: Access

Navigate to **Observe > Dashboards** in the OpenShift Console and select "vLLM Advanced Performance" from the Dashboard dropdown. Use the **Model** dropdown to filter by model.

## Cleanup

```bash
oc delete configmap grafana-dashboard-vllm-advanced -n openshift-config-managed
```

## Format Notes

The OpenShift Console dashboard renderer does NOT support modern Grafana dashboard JSON. Dashboards must use the legacy format:

| Requirement | Correct | Will NOT work |
|---|---|---|
| Layout | `rows` array with nested `panels` | Root-level `panels` array |
| Time series | `"type": "graph"` | `"type": "timeseries"` |
| Single value | `"type": "singlestat"` | `"type": "stat"` or `"type": "gauge"` |
| Gauge | `singlestat` with `"gauge": {"show": true}` | `"type": "gauge"` |
| Datasource | `"$datasource"` template variable | Hardcoded datasource UIDs |
| Schema | `"schemaVersion": 14` | `schemaVersion` >= 30 |
| Dropdown filters | `"type": "query"` with `label_values()` | Same, but `includeAll: true` with `allValue` can break rendering |

Template variable for dropdown filters must match the structure used by built-in k8s dashboards (see the `$model` variable above as reference).
