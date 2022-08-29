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

# Loki & LogQL

---

# Agenda

- What Is Loki
- What Is LogQL
- Log Queries
- Metric Queries

---

<!-- _class: lead -->

# What Is Loki?

---

## What Is Loki?

Loki is a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus.

We ingest Cloudwatch Logs into Loki in order to make it easier to query logs
and create notifications based on patterns.

- Metrics 👉 Prometheus
- Logs 👉 Loki

---

## (Where Is Loki?)

Loki's frontend is https://grafana.rubarb.app

- Metrics 👉 Prometheus
- Logs 👉 Loki
- Frontend 👉 Grafana

---

<!-- _class: lead -->

# What Is LogQL?

---

## What Is LogQL?

LogQL is the query language that is used by Loki.
There are two types of queries:

- LogQL queries which return the contents of log lines.
- Metric queries that calculate values based on the counts of logs from a log query

---

<!-- _class: lead -->

# Log Queries

---

## Log Stream Selector

All queries contain a log stream selector.

```
{__aws_cloudwatch_log_group="/aws/ecs/api/api"}
{__aws_cloudwatch_log_group=~"/aws/ecs/api/api|/aws/lambda/sqs-blabla-trigger"}
{__aws_cloudwatch_log_group=~"/aws/ecs/fawkes.+|/aws/lambda/.+"}
```

---

## Log Stream Selector Operators

`=` exactly equal
`!=` not equal
`=~` regex matches
`!~` regex does not match

#### \* there is an issue with special characters (esp `~`) in the query editor when using keyboard layouts with dead keys. Switch layout to 🇺🇸 (https://github.com/grafana/grafana/issues/43177)

---

## Log Pipelines

Optionally the stream selector can be followed by a pipeline.

- search for the word `error`
  `{stream selector} |= "error"`
- search for a string (case insensitive)
  `{stream selector} |~"(?i)iban"`

Log pipelines can

- filter
- parse
- format

---

## Log Pipelines: Filters

Filter expressions "_grep_" over the aggregated logs from the log streams.

`|=` Log line contains string.
`!=` Log line does not contain string.
`|~` Log line matches regular expression.
`!~` Log line does not match regular expression

---

## Log Pipelines: Parser

Parsers can parse and extract labels from the log lines.

👉 `json`: extract all json properties as labels
👉 `logfmt`: extract key/value pairs to labels (Logfmt-formatted lines)
`regexp` extract labels based on Golang RE2 expressions

Example:
`{stream selector} | json | level=~"warn|error"`

---

## Log Pipelines: Labels

What Is This Label Thing?

Labels are the index to Loki’s log data.
Labels are key value pairs. They are metadata to describe and define a log stream

`{__aws_cloudwatch_log_group="/aws/ecs/api/api"}` 👈 show me logs where the job label is "/aws/ecs/api/api"
It's possible to query many streams by using a single lable.

---

### Log Pipelines: Formatter

`line_format` can rewrite the log line contents using Golang’s templating syntax.

```
{__aws_cloudwatch_log_group="/aws/ecs/api/api"} | json | line_format "{{ upper .msg }}"
```

#### \* see [Template functions](https://grafana.com/docs/loki/latest/logql/template_functions/)

---

### Emojis 🙌

`line_format` even supports Emojis and conditions

```json
{__aws_cloudwatch_log_group="/aws/ecs/api/api"}
  | json
  | level=~"warn|error"
  | line_format "
    {{ if contains `error` .level }}
      🔥 {{ .caller }} - {{ upper .level }} - {{ .msg }}
    {{ else }}
      🤷 {{ .caller }} - {{ .msg }}
    {{ end }}"
```

---

<!-- _class: lead -->

# Metric Queries

---

## Metric Queries

Another thing we can do with Loki is to calculate values based on query results.
👉 We can create metrics from logs 🤯

Example:

- Number of errors in the last 5 minutes:
  `rate({__aws_cloudwatch_log_group="/aws/ecs/api/api"} |~ "error"[5m])`

---

## Metric Queries

common types of aggregation operations are

- `rate()`: Calculate the number of log entries per second
- `count_over_time()`: Count the number of entries in each log stream within a given range
- `sum_over_time()`: the sum of all values in the specified time interval
- `avg_over_time()`: the average value of all points in the specified interval

---

## Alerts

based on those metric queries we can now create alerts

```yaml
- name: PossibleAttack
  rules:
    - alert: PossibleAttack
      expr: |
        rate(
          {__aws_cloudwatch_log_group="/aws/ecs/api/api"}
          | json
          |= "No Record Exists for the Entered Code"
          [1m]
        ) > 5
      for: 2m
      labels:
        severity: critical
       annotations:
           summary: Possible Attack detected in api
           description: More that 5 invalid codes have been entered within 1 minutes
```
