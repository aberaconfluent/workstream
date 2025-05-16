# Watermarks in Stream Processing

Watermarks are a crucial mechanism in stream processing systems for handling out-of-order data and reasoning about the completeness of data over time, especially in unbounded streams.

## What is a Watermark?

A watermark represents the temporal completeness of an out-of-order data stream. Its current value signals to a processor that all messages with an event time timestamp lower than the watermark's timestamp have been received. In simpler terms, if a watermark with a timestamp *t* is observed, the system understands that it will not see any more records with an event time earlier than or equal to *t*.

Watermarks establish a correspondence between processing time (when events are observed or processed) and event time (when events actually occurred in the real world). They must be monotonic, meaning they should never move backward, and ideally conformant, meaning they accurately reflect the event times. However, in practice, heuristic or non-conformant watermarks are also used.

## Why are Watermarks Needed in Stream Processing?

The unbounded and temporally disordered nature of real-world data streams presents a significant challenge: determining the completeness of a stream that never ends. Watermarks are essential for addressing this challenge and enable several key functionalities:

* Computing a Single Correct Answer: For use cases like notifications or alerts, where a single, definitive result is required rather than a stream of incrementally refined partial results. Watermarks provide a completeness signal to generate this conclusive result based on inputs from a specific time range.
* Reasoning About a Lack of Data: In scenarios like dip detection (e.g., identifying when website visits drop below a threshold), watermarks help distinguish between a genuine dip in event rate and lagging input data. They allow the system to delay evaluation until all necessary inputs for the period are likely observed.
* Performing Non-Incremental Processing: Some computations are inherently non-incremental, such as complex statistical models that require full datasets for a given time window. Watermarks allow delaying such computations until all relevant inputs have arrived, avoiding costly recomputations.
* Safe and Punctual Garbage Collection: Watermarks are vital for managing state in stateful computations over infinite streams. They help determine when it's safe to discard obsolete input data and intermediate state, preventing infinite storage growth.
* Surfacing Pipeline Health: A well-instrumented watermark system offers a reliable signal of the overall health and progress of a data processing pipeline. Delays in watermarks can often help pinpoint issues within the pipeline.

## How Watermarks Work in Apache Flink

Apache Flink implements a watermark mechanism that supports event-time processing semantics, where watermarks are generally computed and propagated through the pipeline along with the data itself.

### 1. Watermark Generation (At the Source)

In Flink, watermarks can be generated either directly at the source nodes or in dedicated watermark generation nodes.

* Source-Generated Watermarks: Source nodes can compute watermarks based on the characteristics of the ingested elements or leverage metadata provided by the external system they are reading from (e.g., partitions, offsets, or timestamps in Kafka). For instance, if a Kafka source knows its partitions contain monotonically increasing timestamps, it can generate a conformant watermark based on the minimum of the latest timestamps seen across all partitions.
* Dedicated Watermark Generators: These nodes compute watermarks based solely on the timestamps of the input elements they have observed.
* Publishing Mechanisms: Flink offers two ways to publish watermarks:
    * Periodically: Nodes can maintain a watermark and publish it at a configurable interval.
    * Per-Record: Nodes can directly publish watermarks, potentially for every input record processed.
* Monotonicity: Flink ensures the strict monotonicity of watermarks by only forwarding watermarks that are larger than the previously emitted one.
* Types of Watermarks:
    * Conformant ("Perfect") Watermarks: These accurately reflect that no earlier events will arrive. Generating these is challenging and often depends on specific characteristics of the data source.
    * Non-Conformant ("Heuristic") Watermarks: Used when guaranteeing conformance is impractical. These rely on heuristics like bounded disorder assumptions (advancing the watermark to `event_time - Δ`) or timeouts. More sophisticated heuristics might involve statistical modeling of event source behavior. Late data (elements arriving after the watermark has passed their timestamp) can occur with non-conformant watermarks.

### 2. Watermark Propagation and Consumption (At Operators)

Once generated, watermarks need to be propagated through the dataflow graph to downstream operators.

* Propagation:
    * Flink nodes propagate watermarks by emitting them as special metadata messages alongside regular data records within the data streams.
    * Since Flink's communication channels preserve message order, a receiving operator ingests watermarks and data messages in the same sequence they were sent.
    * Each operator node keeps track of the maximum watermark received on each of its input edges. The operator's current watermark is typically the minimum of these input watermarks. When a new watermark message arrives, the operator updates the corresponding input edge's watermark and recalculates its own watermark.
* Consumption (Reacting to Advancing Watermarks):
    * Operators use watermarks to understand the progress of event time. The primary way they react to advancing watermarks is through watermark timers.
    * Programmers can schedule callbacks (timers) to be executed when an operator's watermark reaches or exceeds a certain timestamp.
    * This is commonly used in windowing operations. For example, when processing hourly windows, a timer can be set for the end of each hour. When the watermark passes the end-of-window timestamp, the timer triggers, and the operator can finalize and emit the result for that window.
    * When an operator's watermark advances, it processes all pending timers whose timestamps are less than the new watermark value, invoking their respective callbacks. These callbacks can then send messages downstream or modify the operator's state.
* Handling Multiple Inputs: When an operator has multiple input streams, its event time clock is typically determined by the minimum of the watermarks from all its input streams. This ensures that the operator only processes data up to the point where all its inputs are considered complete.
* Stateful Operators: Stateful operators (like those performing aggregations, joins, or grouping) use watermarks to know when it's safe to finalize computations for a given time window and potentially clear out state associated with that window. For example, an aggregation operator might buffer incoming messages or partial aggregates in its state until the watermark indicates that all messages for a particular aggregate (e.g., a time window) have arrived.

### 3. Watermarks at Sinks

While the provided document focuses extensively on watermark generation at sources and propagation/consumption at operators, the behavior at sinks is a natural extension:

* Determining Finality: Sinks, like any other operator, receive watermarks from their upstream operators. A sink can use the incoming watermark to understand when data for a particular event time window is considered complete by the upstream processing logic.
* Triggering Actions: This "completeness" signal can then be used by the sink to trigger actions like:
    * Writing final aggregated results to an external database or file system.
    * Sending notifications or alerts.
    * Committing transactions.
* In essence, the watermark tells the sink that it is unlikely (or impossible, in the case of perfect watermarks) to receive further data for event times preceding the watermark. This allows the sink to perform its final operations with a degree of certainty about the completeness of the data it has processed for that time period.

### Challenges and Considerations in Flink:

* Fault Recovery: In Flink, if a node fails, its watermark is reset, and the pipeline typically needs to rewind to the last checkpoint. Watermarks must then be propagated again from the sources.
* Idle Sources: If a source task has no new input data, it can stall watermark progress for the entire pipeline. Flink allows such sources to declare themselves "idle," temporarily excluding them from watermark calculations by downstream operators.
* Bottlenecks and Skew: A single slow or bottlenecked operator can obstruct watermark progress for all downstream operators. Similarly, if different sources have watermarks progressing at vastly different rates (watermark skew), it can lead to increased state size for buffering. Flink provides mechanisms to help manage skew.

In summary, watermarks in Flink provide a robust system for managing event time and out-of-order data, enabling correct and efficient stream processing across sources, operators, and ultimately influencing actions at sinks.
```
