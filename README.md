# AIOps Assessment: Payment Service Monitoring

This repository contains a small AIOps example built around a synthetic
`payment-service` workload. The service normally processes requests quickly,
but the sample includes a short incident where requests slow down and the
service reports timeouts.

## What is being monitored?

The data in [`data/service_data.json`](data/service_data.json) contains ten
records covering `2026-09-20T10:00:00` through `2026-09-20T10:09:00`. Each record
has a timestamp and service name, three numeric measurements, and a log entry.

# Task 2.1
The metrics are:

- `response_time_ms`: request latency in milliseconds
- `cpu_percent`: CPU utilization
- `memory_percent`: memory utilization

# Task 2.2
The log information is held in `log_level` and `message`.

# Task 2.3
The timestamps are
one minute apart, which makes the change in behaviour easy to follow: the
service is stable before 10:05, degrades at 10:05 and 10:06, and returns to
normal from 10:07 onward.

# Task 2.4
The records from 10:00-10:04 and 10:07-10:09 look normal.Response times stay
between 120 and 150 ms, CPU stays between 42% and 50%, memory stays between 51%
and 57%, and the logs say that payment requests were processed successfully.

# Task 2.5
The two unusual records are:

- At 10:05, response time reaches 610 ms and the service logs a payment timeout.
- At 10:06, response time reaches 640 ms, CPU reaches 94%, memory reaches 91%,
	and the service logs a database connection timeout.

## How the example is organised

The main pieces of the workflow are kept intentionally small:

- [`src/anomaly_detector.py`](src/anomaly_detector.py) checks the response-time,
	CPU, and memory thresholds and builds an anomaly event when a check fails.
- [`src/event_topic.py`](src/event_topic.py) is the in-memory topic used to hold
	events.
- [`src/event_producer.py`](src/event_producer.py) publishes detected events,
	while [`src/event_consumer.py`](src/event_consumer.py) reads events from a
	topic.
- [`src/aiops_pipeline.py`](src/aiops_pipeline.py) loads the JSON data, runs the
	detector, and coordinates the producer and consumer.

The separate calculation example in [`src/calculations.py`](src/calculations.py)
is unrelated to the AIOps flow. The workflow tests are in
[`tests/test_aiops_pipeline.py`](tests/test_aiops_pipeline.py).

## Detection results

# Task 3.1
The provided detector was run against all ten records using its default
thresholds: response time above 500 ms, CPU above 80%, or memory above 80%.

# Task 3.2
The detector reported two anomalies:

- At `2026-09-20T10:05:00`, response time was 610 ms. This exceeded the
	response-time threshold. The related log was an `ERROR` with the message
	`Payment service timeout`.
- At `2026-09-20T10:06:00`, response time was 640 ms, CPU was 94%, and memory
	was 91%. All three values exceeded their thresholds. The related log was an
	`ERROR` with the message `Database connection timeout`.

# Task 3.3
The relevant log information is that both unusual records have `ERROR` level
entries. The detector now treats `ERROR` as a concerning log level, so both
messages are included in the anomaly reasons. The first message reports a
payment service timeout, while the second reports a database connection
timeout.

# Task 3.4
The eight remaining observations were not flagged and appear to represent
normal behaviour. No expected metric anomaly was missed, and no normal event
was incorrectly flagged. The generated anomaly events include the timestamp,
service, detection reasons, and original source record, so the report provides
enough information to understand why each observation was flagged.

# Task 3.5
The resulting report is readable because each anomaly includes its timestamp,
service, detection reasons, and original source record. The producer and
consumer now share the same in-memory topic, so the pipeline also returns both
detected events in `events_consumed`. This makes it possible to connect the
flagged metric values with the related log message.

# Task 3.6
The detector now identifies both `WARNING` and `ERROR` log entries. One
remaining limitation is that it uses fixed thresholds rather than adapting to a
service's normal baseline. A possible improvement would be to calculate
service-specific baselines and detect gradual changes as well as fixed
threshold breaches.

# Task 4.1
When the detector identifies an abnormal record, it returns an anomaly event.
For this data, the records at `2026-09-20T10:05:00` and
`2026-09-20T10:06:00` each produced an event containing the service, timestamp,
type, reasons, and original source record.

# Task 4.2
The pipeline passes each non-empty event to `EventProducer.publish`. The
producer is responsible for accepting the event before it is sent to the topic.

# Task 4.3
`EventProducer` publishes the event to the `service-events` `EventTopic`. The
topic stores the event in memory, preserving it for the consumer.

# Task 4.4
`EventConsumer` is connected to the same `service-events` topic and receives
the two published anomaly events through `consume`.

# Task 4.5
The consumer processes the topic contents by retrieving the stored messages as
a list. Both anomaly events are returned with their detection reasons and
source data intact.

# Task 4.6
The consumed events reach the downstream AIOps result through the
`events_consumed` value returned by `run_pipeline`. The command-line workflow
then prints each event's service, timestamp, type, and reasons.

# Task 4.7
The component roles are:

- **Event/message:** the anomaly dictionary created by `AnomalyDetector`.
- **Producer:** `src/event_producer.py`, which publishes an event.
- **Topic:** `src/event_topic.py`, the in-memory event store.
- **Consumer:** `src/event_consumer.py`, which retrieves events from the topic.
- **Downstream AIOps component:** `src/aiops_pipeline.py`, which coordinates
	detection and reports the consumed events.

# Task 4.8
The workflow was executed with `python3 src/aiops_pipeline.py`. The result was:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
```

The consumed events were for `payment-service` at 10:05 and 10:06. This
confirms that an anomaly travelled through detection, production, the topic,
consumption, and the downstream AIOps result.

# Task 5.1 - Investigate the workflow
The workflow was checked by running the detector tests, importing the pipeline
through `src.aiops_pipeline`, and executing `python3 src/aiops_pipeline.py`.
This exposed problems in log detection, topic wiring, and module imports.

# Task 5.2 - Correct log detection
The affected component was `src/anomaly_detector.py`. Its log check only
treated `WARNING` as concerning, but the supplied operational data uses
`ERROR` for both timeout records. The detector was corrected to recognise both
`WARNING` and `ERROR` while keeping the existing threshold checks and event
format. The detector test was rerun and now confirms that the `ERROR` reason is
included in the generated event.

# Task 5.3 - Correct event-topic wiring
The affected components were `src/aiops_pipeline.py`, `EventProducer`, and
`EventConsumer`. The producer originally published to `service-events`, while
the consumer listened to a separate `anomaly-events` instance, so no events
could reach the consumer. The consumer was connected to the producer's
existing `service-events` topic. Rerunning the pipeline confirmed that two
events were published and two events were consumed.

# Task 5.4 - Correct module imports
The affected components were the pipeline, producer, and consumer modules. The
modules used top-level imports that worked when the pipeline was run as a
script, but failed when the tests imported it as `src.aiops_pipeline`. Relative
imports with a script-execution fallback were added. The focused pipeline tests
were rerun successfully, and direct script execution continued to work.

# Task 5.5 - Verify the corrected workflow
After the corrections, the complete workflow produced the following result:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
```

The two consumed events were printed with their service, timestamp, anomaly
type, and reasons. The full test suite also passed with 9 tests, confirming
that the corrections work within the existing detector, producer, topic,
consumer, and pipeline architecture.

# Task 6.1 - Process operational data
The complete workflow loaded `data/service_data.json` and processed all 10
records.

# Task 6.2 - Detect anomalous behaviour
The detector identified the two abnormal records at 10:05 and 10:06 based on
high response time, and additionally high CPU and memory at 10:06.

# Task 6.3 - Generate anomaly events
Each detected record produced an `ANOMALY` event containing the timestamp,
service, detection reasons, and original source record.

# Task 6.4 - Publish events
The pipeline passed both generated events to `EventProducer`, which published
them to the `service-events` topic.

# Task 6.5 - Consume events
`EventConsumer` read both events from the shared `service-events` topic. The
execution result confirms `Events consumed: 2`.

# Task 6.6 - Process events successfully
The downstream pipeline returned both consumed events in `events_consumed` and
the command-line workflow printed their details without errors.

# Task 6.7 - Verify the final AIOps output
The end-to-end command `python3 src/aiops_pipeline.py` produced:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2

Service: payment-service
Timestamp: 2026-09-20T10:05:00
Type: ANOMALY
Reasons: High response time, Error log detected

Service: payment-service
Timestamp: 2026-09-20T10:06:00
Type: ANOMALY
Reasons: High response time, High CPU utilization, High memory utilization, Error log detected
```

This output represents the operational issue: the payment service experienced
slow responses and timeout-related error logs, with CPU and memory saturation
also present at 10:06. The complete flow was therefore verified as
operational data -> anomaly detection -> event -> producer -> topic -> consumer
-> AIOps result.

# Task 7.1 - AIOps scenario
This assessment monitors a synthetic `payment-service`. The scenario is a
short service degradation in which payment requests become slow, timeout logs
appear, and resource utilization increases.

# Task 7.2 - Operational data
The operational data is in `data/service_data.json`. It contains 10 timestamped
records with the service name, response time, CPU percentage, memory percentage,
log level, and log message.

# Task 7.3 - Log and metric observations
The normal records have response times of 120-150 ms, CPU between 42% and 50%,
memory between 51% and 57%, and successful `INFO` messages. The unusual records
are at 10:05 and 10:06, where response times reach 610 and 640 ms and timeout
logs are recorded. At 10:06, CPU reaches 94% and memory reaches 91%.

# Task 7.4 - Anomaly-detection findings
The detector found two anomalies. It identified high response time at both
timestamps, plus high CPU and memory at 10:06. It also includes the `ERROR` log
signal in each anomaly's reasons. No normal record was incorrectly flagged.

# Task 7.5 - Event-processing flow
`AnomalyDetector` creates an event, `EventProducer` publishes it to the
`service-events` topic, and `EventConsumer` retrieves it from that same topic.
`aiops_pipeline.py` returns and prints the consumed events as the downstream
AIOps result.

# Task 7.6 - Final workflow execution
The final execution processed 10 records, detected 2 anomalies, and consumed 2
events. The printed output identifies the affected `payment-service` timestamps
and the metric and log reasons for each anomaly.

# Task 7.7 - Issues corrected
The detector was corrected to recognise `ERROR` logs, the consumer was attached
to the producer's topic instead of a separate topic, and import fallbacks were
added so the workflow works both as `python3 src/aiops_pipeline.py` and through
the package-based tests.

# Task 7.8 - Limitation and possible improvement
The detector uses fixed thresholds, so it may miss gradual changes that remain
below those limits or flag services whose normal baseline is naturally high. A
possible improvement is to calculate service-specific baselines and detect
deviations from them.

# Task 7.9 - Reproduce the demonstration
From the repository root, install the listed dependencies and run the workflow
and tests:

```bash
python3 -m pip install -r requirements.txt
python3 src/aiops_pipeline.py
python3 -m pytest
```

The expected final counts are `Records processed: 10`, `Anomalies detected: 2`,
and `Events consumed: 2`. The demonstration is documented in text here; no
screenshots are required to reproduce or verify it.

## Running it

From the repository root:

```bash
python3 src/aiops_pipeline.py
python3 -m pytest
```

