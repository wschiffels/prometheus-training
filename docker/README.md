# docker prometheus test

this is a simple docker-compose bringing up a prometheus container with thaos sidecar.
Originally created to test the rewrite feature of the thanos sidecar (`"--reloader.config-envsubst-file`).
It might be useful for other tests, so I keep it here.

dummy prometheus config
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    config: dummy

alerting:
  alertmanagers:
    - static_configs:
        - targets:

rule_files:

scrape_configs:
  - job_name: "prometheus"

    static_configs:
      - targets: ["localhost:9090"]
```

## Usage
$ docker-compose up|down

the config file `/etc/prometheus/prometheus.yml` will be overridden with the rendered content of `/etc/prometheus/_prometheus.yml`
