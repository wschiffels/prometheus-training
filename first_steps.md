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
---

<!-- _class: lead gaia -->

# Prometheus

---

# Agenda

- Prometheus At Kudona
- Learn Some PromQL
- Metric Types
- Metric Naming
- Do's And Dont's

---

<!-- _class: lead -->

# Prometheus At Kudona

---

## (Simplified) Overview

```text
         ┌──Grafana──────┐              ┌─Slack─┐
         │               │              │       │
         │               ├─Notify───────►       │
         └──────┬────────┘              │       │
                │                       │       │
                │                       └───────┘
                │
              Query
                │
                │
                │
┌──AWS ACCOUNT 1├────────────────────────────────────────┐
│               │                                        │
│               │                                        │
│               │                                        │
│               │                     ┌──Service 1────┐  │
│               │             ┌───────►               │  │
│               │             │       │               │  │
│               │             │       └───────────────┘  │
│               │             │                          │
│ ┌──Prometheus─▼─┐           │       ┌──Service 2────┐  │
│ │               ├─Scrapes───┼───────►               │  │
│ │               │           │       │               │  │
│ └───────────────┘           │       └───────────────┘  │
│                             │                          │
│                             │       ┌──Service 3────┐  │
│                             └───────►               │  │
│                                     │               │  │
│                                     └───────────────┘  │
│                                                        │
│                                                        │
│                                                        │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## Where Prometheus Can Help Us

- Receive notifications if a service is not working as expected
- Investigation of problems

---

## Where Prometheus Cannot Help Us

- Prometheus can't report why an error occured - for this we have logs

---

<!-- _class: lead -->

# Learn Some PromQL

---

👉 https://grafana.blabla.app/

- Return all time series with the metric `http_requests_total`

  ```text
  http_requests_total
  ```

- Return all time series with the metric `http_requests_total` and the labels `code`and `job`

  ```text
  http_requests_total{code="200", job="thanos-query"}
  ```

- Use regular expressions. In this case get reserved cpu for all services
  containing "api" in their names

  ```text
  aws_ecs_containerinsights_cpu_reserved_average{dimension_ServiceName=~".*api"}
  ```

- Use regular expressions: Select all status codes exept `4xx`

  ```text
  http_requests_total{code!~"4.."}
  ```

---

## Metric Types

There are four standard metric types in Prometheus:

- Counter
- Gauge
- Summary
- Histogram

---

## Counter

A counter counts things.

Query: `http_requests_total{instance="thanos-query.fawkes-prometheus.local:9090", code="200", method="get"}`

Result:

```text
http_requests_total{code="200", handler="label_values", instance="thanos-query.fawkes-prometheus.local:9090", job="thanos-query", method="get"} 18
http_requests_total{code="200", handler="metadata", instance="thanos-query.fawkes-prometheus.local:9090", job="thanos-query", method="get"}	15
http_requests_total{code="200", handler="rules", instance="thanos-query.fawkes-prometheus.local:9090", job="thanos-query", method="get"}	4

```

- only goes up, resets on restarts, easy to understand
- functions like `rate()` or `increase()` only work with Counter
- guarantees that values are not missed between scrapes (scraping interval)

---

## Counter

### When To Use?

Basically all the time. Counters are useful to track how many times errors
occur or how many times a function is called.

---

## Counter

### Calculate A Ratio

The number of errors itself is not very useful without knowing, how many
requests were successful. With a second counter you can calculate the ratio

```
rate(exception_counter_total[1m]) / rate(request_counter_total[1m])
```

This is how to generally expose ratios: expose two counters, then rate and
divide them in PromQL

---

## Counter

internally Promethues uses 64bit floating-point numbers for counters. You're not
limited to icrement a counter by one, you can increment by any positive number.

### Real Live Examples

- Electricity meter, Gas meter
- Infections worldwide with the Corona virus

---

## Gauge

Query: `sqs_idverification_size{queue_name="idverification-dlq"}[5m]`

Result:

```text
sqs_idverification_dlq_size{queue_name="idverification-dlq"}
345 @1648132938
489 @1648132941
123 @1648132953

```

- Value can take on any number (positive and negative)
- Value can arbitrarily go up or down
- High-frequency updates are missed due to the scrape interval (30s)

---

## Gauge

Gauges are a snapshot of some current state. While for counters how fast it is
increasing is what you care about, for gauges it is the actual value of the
gauge. The values can go both up and down.

### When To Use?

Only to capture the value of something at a point in time.

---

## Gauge

### Real Life Examples

- thermometer
- messages in queues

---

## Summary And Histogram

Summary and Histogram (👇) are more complex types of metrics. Summaries and
Histograms share a lot of simularities. It's recommended to use histograms where
possible. They both sample _observations_, usually request durations or response
sizes.

Important differences between Summary and Histogram:

- Summary quantiles are calculated on the application server
- Histogram quantiles are calculated on the prometheus server

---

## Summary

A Summery will give you the average latency.

---

## Histogram

A Histogram will provide you with a quantile.

Query: `prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query"}`
Result:

```text
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="0.1"}  107
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="0.2"}  107
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="1"}	  108
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="3"}	  109
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="60"}   109
prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query",le="+Inf"} 109
```

---

## Histogram

One request that took 2 seconds gets sorted into buckets:

- Is `2` less or equal `0.1` --> No --> Do not increase counter
- Is `2` less or equal `0.2` --> No --> Do not increase counter
- Is `2` less or equal `1` --> No --> Do not increase counter
- Is `2` less or equal `3` --> Yes --> Increase counter
- Is `2` less or equal `60` --> Yes --> Increase counter
- Is `2` less or equal `+Inf` --> Yes --> Increase counter

Buckets are cumulative, so adding a value to a bucket also adds it to the larger
buckets.

---

## Histogram

### When To Use?

Only if the duration of something is of interest and quantiles need to be
calculated (using `histogram_quantile()`), e.g. Apdex or 95th/99th percentiles.

---

## Summary of Metric Types

|                               | Counter | Gauge | Summary | Histogram |
| ----------------------------- | ------- | ----- | ------- | --------- |
| can go up and down            | x       | ✓     | ✓       | ✓         |
| complex type                  | x       | x     | ✓       | ✓         |
| is an approximation           | x       | x     | ✓       | ✓         |
| can query with rate           | ✓       | x     | x       | x         |
| can calculate percentiles     | x       | x     | ✓       | ✓         |
| do I really need to know this | ✓       | ✓     | x       | (✓)       |

---

<!-- _class: lead -->

# Metric Naming

- The overall structure of a metric name is generally `library_name_unit_suffix`.
- Names should start with a letter, followed by letters, numbers and underscores
- The name of the metric should give an idea of it's meaning

---

## Metric Naming

The convention is to use snake_case for metric names: Each component of the name
should be lowercase and separated by an underscore

---

## Metric Naming

- prefer using un-prefixed base units such as seconds, bytes
- include the unit of your metric in the name. For example,
  `mymetric_seconds_total`

<!-- _class: lead -->

# Do's And Dont's

---

## Use Indicators That Show The Status Of Your Service Immediately

### Do 👍

```text
(
  sum(rate(http_requests_total{application="iam-manager",code=~"5.*"}[1m]))
  /
  sum(rate(http_requests_total{application="iam-manager"}[1m]))
) > 0.1
```

Good because:

Errors, e.g. after the deployment of a faulty version, are discovered fast (low Mean Time To Detect)

---

## Use Indicators That Show The Status Of Your Service Immediately

### Don't 👎

Queries that only gain relevance after several hours (or even days, weeks):

```text
(
  sum(rate(http_requests_total{application="iam-manager",code=~"5.*"}[6h]))
  /
  sum(rate(http_requests_total{application="iam-manager"}[6h]))
) > 0.1
```

Bad because:

- storing metrics long-term puts significant load on Prometheus (a general trade-off of Time Series Databases)
- realizing that something is wrong after hours or days is often too late

---

## Use Labels With Low Cardinality

### Do 👍

```text
rubarb_frontend_web_logins_total{os="android"} 123
rubarb_frontend_web_logins_total{os="ios"} 456
rubarb_frontend_web_logins_total{os="web"} 789
```

Label "os" can take on only one of three values.

Good because:

Low number of time series puts less stress on Prometheus.

---

## Use Labels With Low Cardinality

### Don't 👎

```text
rubarb_frontend_logins_total{os="android",osVersion="1.0.0"} 65
rubarb_frontend_logins_total{os="android",osVersion="1.0.1"} 50
rubarb_frontend_logins_total{os="ios",osVersion="2.0.0"} 8
rubarb_frontend_logins_total{os="web",osVersion="..."}
```

Label "osVersion" can take on thousands of values.

Bad because:

Querying for a time series with that many values can kick a Prometheus into OOM.

---

## Avoid Tracking Business Kpis Through Prometheus

### Do 👍

```text
rubarb_payout_processed_total{result="success"} 123
rubarb_payout_processed_total{result="error"} 5
```

Good because:

Indicates if something is wrong (metric with label `result="error"` goes up) but it is not
possible to base business decisions on this metric.

---

## Avoid Tracking Business Kpis Through Prometheus

### Don't 👎

```text
rubarb_payout_processed_amount_euros_total{country="DE|ES|GB|..."} 10000.01
```

Bad because:

- Does not indicate if something is wrong.
- Tempting to make business decisions based on such data. It is not guaranteed that the data set is clean!

Contact a BI Department if such metrics are needed.

---

```
 ___________
< Thank You >
 -----------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
```
