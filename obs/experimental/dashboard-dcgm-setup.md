# NVIDIA DCGM Dashboard in OpenShift Console

## Overview

This sets up an NVIDIA DCGM GPU metrics dashboard in the OpenShift Console under **Observe > Dashboards**. It displays GPU temperature, utilization, power, memory, clock speeds, encoder/decoder usage, and NVLink bandwidth with instance and GPU dropdown filters.

## Prerequisites

- OpenShift 4.x cluster with admin access
- NVIDIA GPU Operator installed with DCGM exporter enabled
- DCGM exporter ServiceMonitor enabled (so metrics flow into cluster Prometheus)

### Enable DCGM exporter ServiceMonitor

The GPU Operator must be configured to create a ServiceMonitor for the DCGM exporter. Without this, DCGM metrics will not appear in Prometheus.

```bash
oc patch clusterpolicy gpu-cluster-policy --type=merge \
  -p '{"spec":{"dcgmExporter":{"serviceMonitor":{"enabled":true}}}}'
```

Verify the ServiceMonitor was created:

```bash
oc get servicemonitor nvidia-dcgm-exporter -n nvidia-gpu-operator
```

Verify metrics are flowing (may take ~60s after ServiceMonitor creation):

```bash
TOKEN=$(oc whoami -t)
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://$(oc get route thanos-querier -n openshift-monitoring -o jsonpath='{.spec.host}')/api/v1/query?query=DCGM_FI_DEV_GPU_TEMP"
```

## Setup

Create the dashboard ConfigMap:

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-dcgm
  namespace: openshift-config-managed
  labels:
    console.openshift.io/dashboard: "true"
data:
  dcgm-exporter.json: |
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
                "height": "200px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 1,
                        "span": 3,
                        "targets": [
                            {
                                "expr": "avg(DCGM_FI_DEV_GPU_TEMP{instance=~\"$instance\", gpu=~\"$gpu\"})",
                                "format": "time_series",
                                "legendFormat": ""
                            }
                        ],
                        "title": "GPU Avg. Temp",
                        "type": "singlestat",
                        "valueName": "current",
                        "valueFontSize": "80%",
                        "format": "celsius",
                        "thresholds": "50,75",
                        "colorBackground": false,
                        "colorValue": true,
                        "colors": ["#299c46", "#ed8128", "#d44a3a"],
                        "sparkline": {"show": true, "full": false, "lineColor": "#3B99FC", "fillColor": "rgba(59,153,252,0.15)"},
                        "gauge": {"show": true, "minValue": 0, "maxValue": 100, "thresholdLabels": false, "thresholdMarkers": true}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 2,
                        "span": 3,
                        "targets": [
                            {
                                "expr": "avg(DCGM_FI_DEV_GPU_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"})",
                                "format": "time_series",
                                "legendFormat": ""
                            }
                        ],
                        "title": "GPU Avg. Utilization",
                        "type": "singlestat",
                        "valueName": "current",
                        "valueFontSize": "80%",
                        "format": "percent",
                        "thresholds": "50,80",
                        "colorBackground": false,
                        "colorValue": true,
                        "colors": ["#299c46", "#ed8128", "#d44a3a"],
                        "sparkline": {"show": true, "full": false, "lineColor": "#3B99FC", "fillColor": "rgba(59,153,252,0.15)"},
                        "gauge": {"show": true, "minValue": 0, "maxValue": 100, "thresholdLabels": false, "thresholdMarkers": true}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 3,
                        "span": 3,
                        "targets": [
                            {
                                "expr": "sum(DCGM_FI_DEV_POWER_USAGE{instance=~\"$instance\", gpu=~\"$gpu\"})",
                                "format": "time_series",
                                "legendFormat": ""
                            }
                        ],
                        "title": "GPU Power Total",
                        "type": "singlestat",
                        "valueName": "current",
                        "valueFontSize": "80%",
                        "format": "watt",
                        "thresholds": "150,250",
                        "colorBackground": false,
                        "colorValue": true,
                        "colors": ["#299c46", "#ed8128", "#d44a3a"],
                        "sparkline": {"show": true, "full": false, "lineColor": "#3B99FC", "fillColor": "rgba(59,153,252,0.15)"},
                        "gauge": {"show": true, "minValue": 0, "maxValue": 400, "thresholdLabels": false, "thresholdMarkers": true}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 4,
                        "span": 3,
                        "targets": [
                            {
                                "expr": "avg(DCGM_FI_DEV_MEM_COPY_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"})",
                                "format": "time_series",
                                "legendFormat": ""
                            }
                        ],
                        "title": "GPU Mem Copy Utilization",
                        "type": "singlestat",
                        "valueName": "current",
                        "valueFontSize": "80%",
                        "format": "percent",
                        "thresholds": "50,80",
                        "colorBackground": false,
                        "colorValue": true,
                        "colors": ["#299c46", "#ed8128", "#d44a3a"],
                        "sparkline": {"show": true, "full": false, "lineColor": "#3B99FC", "fillColor": "rgba(59,153,252,0.15)"},
                        "gauge": {"show": true, "minValue": 0, "maxValue": 100, "thresholdLabels": false, "thresholdMarkers": true}
                    }
                ],
                "title": "Overview"
            },
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 5,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_GPU_TEMP{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "GPU {{gpu}} - {{Hostname}}"
                            }
                        ],
                        "title": "GPU Temperature",
                        "type": "graph",
                        "yaxes": [
                            {"format": "celsius", "label": "Temperature (C)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 6,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_POWER_USAGE{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "GPU {{gpu}} - {{Hostname}}"
                            }
                        ],
                        "title": "GPU Power Usage",
                        "type": "graph",
                        "yaxes": [
                            {"format": "watt", "label": "Power (W)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Temperature & Power"
            },
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 7,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_GPU_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "GPU {{gpu}} - {{Hostname}}"
                            }
                        ],
                        "title": "GPU Utilization",
                        "type": "graph",
                        "yaxes": [
                            {"format": "percent", "label": "Utilization %", "show": true, "min": 0, "max": 100},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 8,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_MEM_COPY_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "GPU {{gpu}} - {{Hostname}}"
                            }
                        ],
                        "title": "GPU Mem Copy Utilization",
                        "type": "graph",
                        "yaxes": [
                            {"format": "percent", "label": "Utilization %", "show": true, "min": 0, "max": 100},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Utilization"
            },
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 9,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_FB_USED{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "Used - GPU {{gpu}} {{Hostname}}"
                            },
                            {
                                "expr": "DCGM_FI_DEV_FB_FREE{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "Free - GPU {{gpu}} {{Hostname}}"
                            }
                        ],
                        "title": "GPU Framebuffer Memory",
                        "type": "graph",
                        "yaxes": [
                            {"format": "decmbytes", "label": "Memory (MiB)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "stack": false,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 10,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_SM_CLOCK{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "SM - GPU {{gpu}} {{Hostname}}"
                            },
                            {
                                "expr": "DCGM_FI_DEV_MEM_CLOCK{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "Mem - GPU {{gpu}} {{Hostname}}"
                            }
                        ],
                        "title": "GPU Clock Speeds",
                        "type": "graph",
                        "yaxes": [
                            {"format": "hertz", "label": "Clock (MHz)", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Memory & Clock"
            },
            {
                "collapse": false,
                "height": "250px",
                "panels": [
                    {
                        "datasource": "$datasource",
                        "id": 11,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_ENC_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "Encoder - GPU {{gpu}} {{Hostname}}"
                            },
                            {
                                "expr": "DCGM_FI_DEV_DEC_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "Decoder - GPU {{gpu}} {{Hostname}}"
                            }
                        ],
                        "title": "Encoder/Decoder Utilization",
                        "type": "graph",
                        "yaxes": [
                            {"format": "percent", "label": "Utilization %", "show": true, "min": 0, "max": 100},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    },
                    {
                        "datasource": "$datasource",
                        "id": 12,
                        "span": 6,
                        "targets": [
                            {
                                "expr": "DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                                "format": "time_series",
                                "legendFormat": "GPU {{gpu}} - {{Hostname}}"
                            }
                        ],
                        "title": "NVLink Bandwidth",
                        "type": "graph",
                        "yaxes": [
                            {"format": "Bps", "label": "Bandwidth", "show": true},
                            {"show": false}
                        ],
                        "xaxis": {"show": true},
                        "lines": true,
                        "linewidth": 2,
                        "fill": 1,
                        "legend": {"show": true, "values": true, "current": true},
                        "tooltip": {"shared": true, "sort": 0}
                    }
                ],
                "title": "Encoder/Decoder & NVLink"
            }
        ],
        "schemaVersion": 14,
        "style": "dark",
        "tags": ["nvidia", "dcgm", "gpu"],
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
                    "allValue": ".*",
                    "current": {
                        "text": "All",
                        "value": "$__all"
                    },
                    "datasource": "$datasource",
                    "hide": 0,
                    "includeAll": true,
                    "label": "Instance",
                    "multi": true,
                    "name": "instance",
                    "options": [],
                    "query": "label_values(DCGM_FI_DEV_GPU_TEMP, instance)",
                    "refresh": 2,
                    "regex": "",
                    "sort": 1,
                    "tagValuesQuery": "",
                    "tags": [],
                    "tagsQuery": "",
                    "type": "query",
                    "useTags": false
                },
                {
                    "allValue": ".*",
                    "current": {
                        "text": "All",
                        "value": "$__all"
                    },
                    "datasource": "$datasource",
                    "hide": 0,
                    "includeAll": true,
                    "label": "GPU",
                    "multi": true,
                    "name": "gpu",
                    "options": [],
                    "query": "label_values(DCGM_FI_DEV_GPU_TEMP{instance=~\"$instance\"}, gpu)",
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
            "from": "now-1h",
            "to": "now"
        },
        "timepicker": {
            "refresh_intervals": ["5s", "10s", "30s", "1m", "5m"]
        },
        "timezone": "",
        "title": "NVIDIA DCGM Exporter Dashboard",
        "version": 1
    }
EOF
```

## Verify

```bash
oc get configmap grafana-dashboard-dcgm -n openshift-config-managed -l console.openshift.io/dashboard=true
```

## Access

Navigate to **Observe > Dashboards** in the OpenShift Console and select "NVIDIA DCGM Exporter Dashboard" from the dropdown.

## Cleanup

```bash
oc delete configmap grafana-dashboard-dcgm -n openshift-config-managed
```
