<cheat-sheet>

# Cheat Sheet: Design Ad Click Aggregator

## 1. Problem Statement & Scope

Design a system to collect, process, and aggregate ad click data in near real-time, allowing advertisers to query performance metrics.

### Functional Requirements

*   **Core:**
    *   Users click an ad and are redirected to the advertiser's site.
    *   Advertisers can query ad click metrics over time with a minimum granularity of 1 minute.
*   **Out of Scope:**
    *   Ad serving and targeting.
    *   Advanced fraud detection (beyond simple idempotency).

### Non-Functional Requirements & Scale

*   **Scale:** Handle **10 million** active ads and a peak of **10,000 clicks per second** (~100M clicks/day).
*   **Low Latency Queries:** Advertiser queries should return in **sub-second**.
*   **Fault Tolerance & Accuracy:** No loss of click data.
*   **Near Real-Time:** Data should be queryable as soon as possible after a click.
*   **Idempotency:** The same click (e.g., a user double-clicking) should not be counted multiple times.

## 2. High-Level Design: Real-Time Stream Processing

This is a data processing pipeline problem. A batch-processing approach (e.g., running a Spark job every 5 minutes) is a viable start but fails to meet the near real-time requirement. The optimal solution is a real-time stream processing architecture.

1.  **Click Ingestion & Redirection:**
    *   A user clicks an ad. The browser sends a request to a stateless **Click Processor Service**.
    *   This service logs the click event and returns an `HTTP 302 Redirect` to send the user to the advertiser's website. This server-side redirect prevents users with ad blockers from bypassing the click tracking.

2.  **Data Stream:**
    *   The **Click Processor Service** writes the raw click event into a distributed, persistent message queue like **Apache Kafka** or **AWS Kinesis**. This decouples ingestion from processing and acts as a durable buffer.

3.  **Real-Time Aggregation:**
    *   A stream processing engine like **Apache Flink** or **Spark Streaming** consumes events from the Kafka/Kinesis stream.
    *   Flink jobs perform stateful aggregations in memory over **1-minute tumbling windows** (grouping clicks by `AdId` and minute).
    *   To provide near real-time data, Flink can be configured with shorter **flush intervals** (e.g., 10 seconds) to write partial, intra-minute results to the database.

4.  **Storage for Analytics:**
    *   The aggregated results (e.g., `AdId`, `TimestampMinute`, `ClickCount`) are written to an **OLAP (Online Analytical Processing) database** like **ClickHouse, Apache Druid, or Pinot**. These databases are column-oriented and optimized for fast analytical queries (scans, aggregations) over large datasets.

5.  **Querying:**
    *   An **Analytics Service** provides an API for advertisers.
    *   When an advertiser requests metrics, this service queries the OLAP database, which returns the pre-aggregated results with very low latency.

## 3. Deep Dive: Common Problems & Pitfalls

### Scalability & Hot Shards

*   **Problem:** A viral ad (e.g., a Super Bowl ad) generates a massive number of clicks, overwhelming the single Kafka/Kinesis partition assigned to its `AdId`. This is the "hot shard" or "hot key" problem.
*   **Solution:**
    1.  **Base Sharding:** Shard the stream by `AdId`. This works for a uniform click distribution.
    2.  **Hot Shard Mitigation:** For known hot ads, the **Click Processor** should use a composite partition key, such as `AdId:random_number_0-N`. This splits the clicks for a single hot `AdId` across `N` different partitions, distributing the write load. The Flink job for that `AdId` would then need to consume from all `N` partitions to perform the final aggregation.

### Fault Tolerance & Data Accuracy

*   **Problem:** How to guarantee no data is lost if a Flink processing node fails or there are bugs in the aggregation logic.
*   **Solution: Hybrid Architecture (Real-time + Batch Reconciliation)**
    1.  **Stream Durability:** Configure the Kafka/Kinesis stream with a long data retention period (e.g., 7 days). If a Flink job fails, it can restart and replay the events it missed directly from the stream. Flink's internal checkpointing is less useful here due to the very short 1-minute aggregation windows.
    2.  **Batch Reconciliation:** For 100% accuracy, dump all raw click events from the stream into a cheap data lake (like **S3**). Run a daily **batch job** (using **Spark**) to re-process all of the previous day's data from S3. This batch job's results are considered the "source of truth."
    3.  A **Reconciliation Worker** compares the batch results with the real-time results in the OLAP database and corrects any discrepancies found. This provides the best of both worlds: low-latency real-time data and guaranteed accuracy.

### Idempotency

*   **Problem:** A user clicks an ad multiple times, or a bot spams the click endpoint. How do we count this as only one unique click?
*   **Solution: Signed Impression IDs**
    1.  When an ad is served to a user, the **Ad Placement Service** generates a unique **`AdImpressionID`** for that specific instance of the ad view.
    2.  Crucially, the server **signs** this `AdImpressionID` using a private key, creating a signature.
    3.  The client sends the `AdImpressionID` and its `signature` along with the click event.
    4.  The **Click Processor** first **verifies the signature** with its public key. This proves the impression ID is legitimate and wasn't fabricated by a bot.
    5.  It then checks a cache (e.g., **Redis**) to see if this verified `AdImpressionID` has been processed recently. If it exists in the cache, the click is a duplicate and is dropped. If not, it's added to the cache (with a short TTL) and sent to the stream.

## 4. Interviewer's Guide: Leveling Expectations

*   **Mid-Level (E4):**
    *   Expected to propose a **batch processing** solution (e.g., Spark job on a schedule).
    *   Should understand the need for pre-aggregation to make queries fast. May need prompting to move towards a real-time system.
*   **Senior (E5):**
    *   Must proactively design a **real-time stream processing** architecture (Kafka/Flink).
    *   Should identify and discuss solutions for key issues like scalability (sharding) and fault tolerance.
    *   Clearly articulates the trade-offs between batch and streaming.
*   **Staff+ (E6+):**
    *   Drives the conversation, demonstrating deep, practical knowledge.
    *   Proactively designs a sophisticated, fault-tolerant solution, including the **batch reconciliation** layer.
    *   Discusses nuanced problems like **hot shards** and proposes robust solutions like **signed impression IDs** for idempotency. Treats the interviewer as a peer in a technical design discussion.

</cheat-sheet>