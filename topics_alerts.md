Certainly! Here are example Prometheus alert rules for the most important Redpanda metrics related to **Throughput**, **Consumer Group Lag (Queues)**, and **Topics**, based strictly on the metrics and guidance from the knowledge sources.

---

## 1. Throughput Alerts

Monitor for abnormal drops or spikes in producer and consumer throughput using the following metrics:

- **Producer Throughput:** `redpanda_rpc_received_bytes` (filter: `redpanda_server="kafka"`)
- **Consumer Throughput:** `redpanda_rpc_sent_bytes` (filter: `redpanda_server="kafka"`)

**Example Prometheus alert rules:**

```yaml
groups:
  - name: Redpanda Throughput Alerts
    rules:
      - alert: RedpandaProducerThroughputLow
        expr: rate(redpanda_rpc_received_bytes{redpanda_server="kafka"}[5m]) < 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Producer throughput is low"
          description: "Producer throughput has dropped below 1000 bytes/sec for 5 minutes."

      - alert: RedpandaConsumerThroughputLow
        expr: rate(redpanda_rpc_sent_bytes{redpanda_server="kafka"}[5m]) < 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer throughput is low"
          description: "Consumer throughput has dropped below 1000 bytes/sec for 5 minutes."
```
Adjust the threshold (`1000` bytes/sec) to fit your environment’s baseline.  
[Reference: Throughput metrics](https://docs.redpanda.com/current/manage/monitoring/#throughput)

---

## 2. Consumer Group Lag (Queues) Alerts

Monitor for lag in consumer groups using:

- **Maximum Lag:** `redpanda_kafka_consumer_group_lag_max`
- **Total Lag:** `redpanda_kafka_consumer_group_lag_sum`

**Example Prometheus alert rules:**

```yaml
groups:
  - name: Redpanda Consumer Lag Alerts
    rules:
      - alert: RedpandaConsumerGroupLagHigh
        expr: redpanda_kafka_consumer_group_lag_max > 10000
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High consumer group lag"
          description: "The maximum consumer group lag has exceeded 10,000 messages for 10 minutes."

      - alert: RedpandaConsumerGroupTotalLagHigh
        expr: redpanda_kafka_consumer_group_lag_sum > 50000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High total consumer group lag"
          description: "The total consumer group lag has exceeded 50,000 messages for 10 minutes."
```
Adjust lag thresholds as appropriate for your workload.  
[Reference: Consumer group lag metrics](https://docs.redpanda.com/current/reference/public-metrics-reference/#consumer-group-metrics)

---

## 3. Topics Alerts

Monitor the number of topics being consumed by each consumer group:

- **Topics Consumed:** `redpanda_kafka_consumer_group_topics`

**Example Prometheus alert rule:**

```yaml
groups:
  - name: Redpanda Topic Alerts
    rules:
      - alert: RedpandaConsumerGroupNoTopics
        expr: redpanda_kafka_consumer_group_topics == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Consumer group not consuming any topics"
          description: "A consumer group is not consuming any topics for 5 minutes."
```
[Reference: Consumer group topics metric](https://docs.redpanda.com/current/reference/public-metrics-reference/#consumer-group-metrics)

---

**Note:**  
- These are example alert rules. You should tune thresholds and durations to match your environment and business requirements.
- To use these alerts, ensure you have enabled the relevant metrics in your Redpanda configuration as described in the documentation.

If you need more specific alerting scenarios or have a particular use case, let me know!

---

Thank you for clarifying which metrics you have available. If you do not have `redpanda_kafka_consumer_group_lag_sum`, you can still monitor consumer group lag by calculating it using the available metrics:  
- `redpanda_kafka_consumer_group_committed_offset`
- `redpanda_kafka_max_offset` (this one is required for lag calculation, and is available in all recent Redpanda versions)

### Calculating Consumer Group Lag

You can calculate consumer group lag by subtracting the committed offset from the broker’s latest offset for each partition. The recommended Prometheus query is:

```prometheus
max by(redpanda_namespace, redpanda_topic, redpanda_partition)(redpanda_kafka_max_offset{redpanda_namespace="kafka"})
  - on(redpanda_topic, redpanda_partition)
    group_right
    max by(redpanda_group, redpanda_topic, redpanda_partition)(redpanda_kafka_consumer_group_committed_offset)
```
This query gives you the lag per partition and consumer group. You can then use this result to create alerts for high lag situations.  
[See: Offset-based calculation](https://docs.redpanda.com/current/manage/monitoring/#offset-based-calculation)

---

### Example Prometheus Alert for Consumer Group Lag

```yaml
groups:
  - name: Redpanda Consumer Lag Alerts
    rules:
      - alert: RedpandaConsumerGroupLagHigh
        expr: |
          max by(redpanda_group, redpanda_topic, redpanda_partition)(
            max by(redpanda_namespace, redpanda_topic, redpanda_partition)(redpanda_kafka_max_offset{redpanda_namespace="kafka"})
            - on(redpanda_topic, redpanda_partition)
              group_right
              max by(redpanda_group, redpanda_topic, redpanda_partition)(redpanda_kafka_consumer_group_committed_offset)
          ) > 10000
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High consumer group lag"
          description: "Consumer group lag has exceeded 10,000 messages for 10 minutes."
```
Adjust the threshold as needed for your environment.

---

### Other Available Metrics

You can also create alerts for the other metrics you mentioned:

#### Number of Consumers in a Group

```yaml
- alert: RedpandaConsumerGroupNoConsumers
  expr: redpanda_kafka_consumer_group_consumers == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Consumer group has no active consumers"
    description: "A consumer group has had zero active consumers for 5 minutes."
```
[See: Consumer group metrics](https://docs.redpanda.com/current/reference/public-metrics-reference/#consumer-group-metrics)

#### Number of Topics Consumed by a Group

```yaml
- alert: RedpandaConsumerGroupNoTopics
  expr: redpanda_kafka_consumer_group_topics == 0
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Consumer group not consuming any topics"
    description: "A consumer group is not consuming any topics for 5 minutes."
```
[See: Consumer group metrics](https://docs.redpanda.com/current/reference/public-metrics-reference/#consumer-group-metrics)

---

**Summary:**  
- Use the offset-based calculation for consumer group lag with your available metrics.
- You can alert on the number of consumers and topics per group using the metrics you listed.
- If you need more advanced lag metrics (like `redpanda_kafka_consumer_group_lag_sum`), you may need to enable additional consumer group metrics in your Redpanda configuration if your version supports it.

If you need help with a specific alert or query, let me know!ç

--

