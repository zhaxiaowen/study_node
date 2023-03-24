# Prometheus 监控 redis

#### prometheus 监控虚拟机上部署的 redis

```
  - job_name: 'redis_exporter_targets'
    static_configs:
      - targets: ['192.168.50.100:16379','192.168.50.101:16379','192.168.50.102:16379']
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.50.100:9121
  - job_name: 'redis_exporter'
    static_configs:
      - targets:
        - 192.168.50.100:9121

```

> ./redis_exporter -redis.addr 192.168.50.100:16379     -is-cluster=true
