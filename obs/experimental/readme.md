# Work In Progres!
## Advanced Grafana Dashboard Features
#### **Time to First Token (TTFT)**
- **P50, P95, P99 percentiles** - Complete latency distribution
- **Real-time tracking** - 5-second refresh intervals  
- **Threshold indicators** - Green/Yellow/Red status alerts
- **Performance baselines** - Track improvements over time

#### **KV Cache Performance**
- **Overall Hit Rate Gauge** - System-wide cache efficiency (target: >60%)
- **Per-Pod Breakdown** - Individual decode pod performance
- **Traffic Distribution** - Session affinity visualization
- **Cache Effectiveness** - Red/Yellow/Green threshold indicators

#### **System Health Monitoring**
- **Request Queue Status** - Running vs waiting requests
- **GPU Memory Utilization** - Resource usage tracking  
- **Throughput Metrics** - Success rates and token processing
- **Pod-Level Metrics** - Individual instance performance

To deploy, make sure you are in the /experimental folder

1. ```oc new-project llm-d-monitoring```

1. ```oc apply -f cluster-monitoring-config.yaml -n openshift-monitoring```

1. ```oc apply -f metrics_allowlist.yaml -n llm-d-monitoring```

1. ```oc apply -f grafana-prometheus-token.yaml -n grafana --namespace=openshift-monitoring```

1. ```oc apply -f prometheus-config.yaml -n llm-d-monitoring```

1. ```oc apply -f prometheus.yaml -n llm-d-monitoring```

1. ```oc apply -f grafana-config.yaml -n llm-d-monitoring```

1. ```oc apply -f grafana-datasources.yaml -n llm-d-monitoring```

1. ```oc apply -f grafana-dashboards-config.yaml -n llm-d-monitoring```

1. ```oc apply -f grafana-dashboard-llm-performance.yaml -n llm-d-monitoring```

1. ```oc apply -f expose-grafana.yaml -n llm-d-monitoring```