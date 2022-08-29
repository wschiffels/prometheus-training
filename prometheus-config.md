---
marp: true
theme: nord
size: 16:9
style: |
  section {font-size: 170%;}
  th {background-color: #4c566a;}
  tbody tr:nth-child(even) {background-color: #434c5e;}
  tbody tr {background-color: #3b4252}
  h1 {color: #d8dee9;}
  h4 {font-size: small;}
---

<!-- _class: lead gaia -->

# Prometheus Config

---

# Overview

1. Thanos
2. Prometheus
3. Persistant Storage
4. Update Prometheus Configuration

---

## Overview

We run 4 services in the prometheus ECS cluster:

- prometheus
- thanos-store
- thanos-query
- yace-exporter

and occasionally 1 additional task:

- promtheus-update

---

<!-- _class: lead -->

# Thanos

---

## Thanos

Thanos is an open-source extension to Prometheus.
We primarily use Thanos to push historical data to affordable storage (S3).
It can also watch changes in configuration files and reload Prometheus (👇).

---

## Thanos Store

The `thanos store` implements the Store API on top of historical data S3. It joins a Thanos cluster on startup and advertises the data it can access.

```json
[...]

  "entryPoint": [
      "/bin/thanos",
      "store",
      "--data-dir=/tmp",
👉    "--objstore.config-file=/objstore.yml", # S3-bucket config, rendered during build
      "--grpc-address=0.0.0.0:10901",
      "--http-address=0.0.0.0:10902"
  ],

[...]

```

---

## Thanos Query

`thanos query` queries data directly from prometheus and from `thanos store`.
It's the endpoint for the prometheus datasources in Grafana, so we're not querying Prometheus directly, but Thanos because it also includes historical data already moved to S3.

```json
[...]

  "entryPoint": [
      "/bin/thanos",
      "query",
      "--http-address=0.0.0.0:9090",
👉    "--store=prometheus.prometheus.local:10901",  # current metrics (prometheus container)
👉    "--store=thanos-store.prometheus.local:10901" # historical metrics (S3 bucket)
  ],

[...]
```

#### Thanos Query runs behind an internal Loadbalancer and is accessible for Grafana through VPC-Peering only

---

## Thanos Config And Deployment

Dockerfile

```
FROM quay.io/thanos/thanos:v0.24.0

👉 ARG AWS_ACCOUNT_ALIAS                        # set in pipeline script
👉 ENV AWS_ACCOUNT_ALIAS=${AWS_ACCOUNT_ALIAS}
👉 ARG AWS_ACCOUNT_ID                           # Deployment variable
👉 ENV AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}

COPY       objstore.yml /objstore.yml
RUN        sed -i "s/%%%AWS_ACCOUNT_ALIAS%%%/${AWS_ACCOUNT_ALIAS}-${AWS_ACCOUNT_ID}/g" /objstore.yml
VOLUME     [ "/etc/prometheus" ]
```

objstore.yml

```yml
type: S3
config:
  bucket: "%%%AWS_ACCOUNT_ALIAS%%%-prometheus-tsdb"
  endpoint: "s3.eu-central-1.amazonaws.com"
  region: "eu-central-1"
```

---

<!-- _class: lead -->

# yace-cloudwatch-exporter

---

## yace-cloundwatch-exporter

Yet Another Cloudwatch Exporter: Get data from AWS CloudWatch into Prometheus.

It's a common pattern to use an exporter to fetch metrics from systems that don't provide Prometheus metrics.

---

<!-- _class: lead -->

# Prometheus Task

---

## Prometheus Task

The Prometheus task runs 3 - 4 containers

0. initContainer
   - Make sure configuration is available on startup. Exits after completion.
1. Prometheus
   - The Prometheus server
2. Thanos sidecar
   - Access to prometheus config
   - Render `prometheus.yml` on change.
   - Reload prometheus on config change (http://localhost:9090/-/reload)
   - Access to prometheus tsdb storage. Upload to S3.
3. ecs service discovery
   - Discover ECS Services based on Docker Labels

---

<!-- _class: lead -->

# Persistent Storage (EFS)

---

## Persistent Storage (EFS)

To avoid building a container with a baked-in `prometheus.yml` and alerting rules we need to provide prometheus with some sort of persistent storage.

The storage needs to be shared between the prometheus container, thanos and the update process (👇).

---

## Update Prometheus (Part I)

To refresh the configuration or change notification rules Prometheus can be reloaded at runtime.
The updateContainer is another Task running in the prometheus cluster that takes care of that.

Triggered by a bitbucket-pipeline in the prometheus-config repo the UpdateContainer will:

- start and mount the shared EFS volume
- fetch the latest config versions from git
- dump them into the shared volume
- exit

---

## Update Promtheus (Part II)

Thanos sidecar can:

- (not only) upload metrics to S3 (👆)
- watch config files
- render config files and relace ENV variables (in `$(VAR)` notation)
- reload prometheus once it detects changes

```json
[...]

    "entryPoint": [
      "/bin/thanos",
      "sidecar",
      "--tsdb.path=/prometheus",
      "--prometheus.url=http://localhost:9090/",                        # Prometheus URL
      "--http-address=0.0.0.0:10903",
      "--objstore.config-file=/objstore.yml",                           # where to upload data to
      "--grpc-address=0.0.0.0:10901",
👉    "--reloader.config-file=/etc/prometheus/_prometheus.yml",         # file to watch for changes
👉    "--reloader.config-envsubst-file=/etc/prometheus/prometheus.yml", # file to render on changes
👉    "--reloader.rule-dir=/etc/prometheus/rules.d/"                    # rules to watch
    ],

[...]
```

---

## Update Prometheus (Part III)

replacing ENV variables

```yaml
global:
 external_labels:
   region: eu-central
👉  environment: "$(AWS_ACCOUNT_ALIAS)"
👉  product: "$(PRODUCT)"
[...]

```

---

## Update Prometheus (Part IV)

```json
      [...]

      "environment": [
        {
👉        "name": "AWS_ACCOUNT_ALIAS",
          "value": "dev"
        },
        {
👉        "name": "PRODUCT",
          "value": "blabla"
        }
      ],
      [...]
      "secrets": [
        {
          "valueFrom": "arn:aws:secretsmanager::...-alertmanager-basic-auth-...:password::",
👉        "name": "BASIC_AUTH"
        }
      ],
      [...]
```

---

## Update Prometheus (Part V)

```yaml
global:
 external_labels:
   region: eu-central
👉  environment: "dev"
👉  product: "blabla"
[...]

```

👉 external labels get attached to any metric

```
 whatever_metric{..., environment="dev", product="blabla", region="eu-central"}
```

---

```
 ___________
< Thank you >
 -----------
  \
   \   \_\_    _/_/
    \      \__/
           (oo)\_______
           (__)\       )\/\
               ||----w |
               ||     ||
```
