Here's an example Prometheus configuration to scrape `cdc-sink`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: cdc-sink
    metrics_path: /_/varz
    scheme: https
    tls_config:
      insecure_skip_verify: true
    static_configs:
      - targets: [ '127.0.0.1:30004' ]
```
