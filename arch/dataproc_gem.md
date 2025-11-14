# Billing Data Aggregation System for Open Banking APIs

## 1. Overview

This document outlines the proposed architecture for a new billing data aggregation system. The system is designed to capture, process, and store all billable events from our Open Banking API ecosystem, which includes API calls from Apigee and consent events from the Consent Management System.

The goal is to create a scalable, durable, and cost-effective data pipeline that can support billing for a complex multi-party ecosystem (Bank, Customer, Aggregator, 4th Party App) and handle an estimated volume of **2 billion events per month**.

---

## 2. Core Requirements

| Requirement | Description |
| --- | --- |
| **Data Sources** | Ingest event streams from two primary systems: <br> • **Apigee API Gateway**: For all API usage events. <br> • **Consent System**: For all consent lifecycle events (creation, renewal, revocation). |
| **Event Volume** | The system must handle approximately **2,000,000,000 events per month** (~770 events/second). |
| **Data Retention** | - **Hot Data**: Must be readily available for billing operations for **14 months**. <br> - **Cold Data**: Must be archived for long-term retention (e.g., 7 years) for compliance and analysis. |
| **Billing Logic** | - Support different rates for different API products/endpoints (e.g., Accounts API @ $0.01, Payments API @ $0.02). <br> - Bill for both API usage and consent creation events. <br> - Attribute usage to all relevant parties (Customer, Aggregator, App). |
| **Technology Stack** | The solution must be built using the approved technologies: **Kafka, MongoDB, and an S3-compatible Object Storage.** |
| **Operational Simplicity** | The architecture should be as simple as possible to manage, operate, and debug. |

---

## 3. Architectural Options

We have evaluated two primary architectural approaches that meet the requirements.

---

### Option 1: Kafka -> Object Storage -> MongoDB (Data Lakehouse Pattern)

This architecture prioritizes creating a durable, cost-effective data lake first, using Object Storage as the central repository for all raw data. MongoDB serves as a high-performance, "hot" processing layer for recent data.

#### 3.1. Architecture Diagram

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────────────┐
│   Apigee    │    │  Consent     │    │     Kafka       │    │     MongoDB          │
│ (API Gateway│───▶│  System      │───▶│  (Raw Events)   │───▶│  (Operational Data) │
│   Events)   │    │  Events)     │    │                 │    │                      │
└─────────────┘    └──────────────┘    └───────────────┬─┘    └──────────┬───────────┘
                                                   │                  │
                                                   │ (Archival)       │ (Aggregation)
                                                   ▼                  ▼
                                            ┌──────────────┐   ┌──────────────────────┐
                                            │  Object      │   │ MongoDB Collections   │
                                            │  Storage     │   │ (Billing Reports)     │
                                            │  (S3 Bucket) │   └──────────────────────┘
                                            └──────────────┘
```

#### 3.2. Data Flow

1.  **Ingestion**: Apigee and the Consent System publish JSON event messages to Kafka topics (`api-usage-events`, `consent-events`).
2.  **Landing**: A **Kafka Connect S3 Sink Connector** consumes messages from Kafka and writes them to an Object Storage bucket in an efficient, columnar format (Parquet), partitioned by date and event type. This creates a permanent, immutable raw data lake.
3.  **Processing**: A scheduled job (e.g., Lambda, CronJob) periodically scans the Object Storage bucket for new files, reads them, and bulk-inserts the data into the `raw_events` collection in MongoDB.
4.  **Aggregation**: A separate scheduled job runs MongoDB's Aggregation Pipeline on the `raw_events` collection to calculate usage and charges, storing the results in a pre-aggregated `monthly_bill_summary` collection for fast billing queries.
5.  **Retention**:
    *   **MongoDB**: Data in the `raw_events` collection is deleted after 90 days using a simple TTL index.
    *   **Object Storage**: Data is retained indefinitely. An S3 Lifecycle Policy automatically transitions data to cheaper storage tiers (e.g., Infrequent Access after 90 days, Glacier after 1 year).

#### 3.3. Pros & Cons

| Pros | Cons |
| --- | --- |
| **Highest Durability**: Data is landed permanently in Object Storage first, making it resilient to downstream processing failures. | **Higher Latency**: Data is not available in MongoDB for processing until the scheduled job runs (e.g., 5-15 minute delay). |
| **Most Cost-Effective**: Cheapest long-term storage for the vast majority of data. Reduces load and storage requirements on the more expensive MongoDB cluster. | **More Components**: Involves managing the Kafka S3 Connector and the S3-to-MongoDB ingestion job. |
| **Superior Flexibility**: The raw data lake in Parquet format can be queried by multiple tools (e.g., AWS Athena, Spark) for ad-hoc analysis without impacting MongoDB. | |
| **Decoupled System**: Ingestion is decoupled from processing, allowing each to scale and fail independently. | |

---

### Option 2: Kafka -> MongoDB -> Archive to Object Storage

This architecture prioritizes low-latency availability of data in MongoDB first. Object Storage is used as a final destination for data that has aged out of the "hot" database.

#### 3.2. Architecture Diagram

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────────────┐
│   Apigee    │    │  Consent     │    │     Kafka       │    │     MongoDB          │
│ (API Gateway│───▶│  System      │───▶│  (Raw Events)   │───▶│  (Operational Data) │
│   Events)   │    │  Events)     │    │                 │    │                      │
└─────────────┘    └──────────────┘    └─────────────────┘    └──────────┬───────────┘
                                                                         │
                                                                         │ (Archival)
                                                                         ▼
                                                                ┌──────────────┐
                                                                │  Object      │
                                                                │  Storage     │
                                                                │  (S3 Bucket) │
                                                                └──────────────┘
```

#### 3.2. Data Flow

1.  **Ingestion**: Apigee and the Consent System publish JSON event messages to Kafka.
2.  **Processing**: A **Kafka Connect MongoDB Sink Connector** consumes messages from Kafka and writes them directly to the `raw_events` collection in MongoDB in near real-time.
3.  **Aggregation**: A scheduled job runs MongoDB's Aggregation Pipeline on the `raw_events` collection to populate the `monthly_bill_summary` collection.
4.  **Retention**:
    *   **MongoDB**: A scheduled job (e.g., running monthly) queries the `raw_events` collection for data older than 14 months, exports it, and uploads it to Object Storage before deleting it from MongoDB.
    *   **Object Storage**: Acts as a static archive for the exported data.

#### 3.3. Pros & Cons

| Pros | Cons |
| --- | --- |
| **Low Latency**: Data is available in MongoDB for processing and aggregation within seconds of the event occurring. | **Higher Operational Complexity**: The custom archival job (MongoDB -> S3) is complex to build, test, and maintain. Risk of data loss during the export/delete process. |
| **Simpler Ingestion**: Uses only the MongoDB connector, which is one less component to configure than Option 1. | **Higher MongoDB Costs**: The MongoDB cluster must be provisioned with significantly more storage and IOPS to handle 14 months of hot data, increasing operational costs. |
| **Familiar Pattern**: A more traditional database-centric architecture that may be more familiar to some development teams. | **Tightly Coupled**: The database is a single point of failure for both ingestion and archival. If MongoDB is down, new data cannot be ingested. |

---

## 4. Comparison & Recommendation

### 4.1. Complexity Comparison

| Aspect | Option 1 (Kafka -> S3 -> Mongo) | Option 2 (Kafka -> Mongo -> S3) |
| --- | --- | --- |
| **Ingestion** | Medium (Configure S3 Sink Connector + scheduled job) | Low (Configure MongoDB Sink Connector) |
| **Archival** | Low (Declarative S3 Lifecycle Policy) | High (Custom code for export, upload, delete) |
| **Data Access** | High (Native access via Athena, Spark, etc.) | Low (Requires loading from S3 into a query engine) |
| **Overall** | **Moderate** (More components, but each has a clear, simple role) | **Moderate** (Fewer components, but the archival job is a complex, high-risk piece of custom code) |

### 4.2. Cost & Storage Calculations

**Assumptions:**
*   **Event Volume**: 2 billion events/month.
*   **Avg Event Size**: 1 KB (including JSON overhead).
*   **Parquet Compression**: 80% size reduction (1 KB -> 200 bytes).
*   **Retention Period**: 14 months hot, 7 years cold.
*   **Pricing (Illustrative)**:
    *   MongoDB (M60 cluster, 3TB storage): ~$2,500/month
    *   Object Storage (S3 Standard): ~$0.023/GB/month
    *   Object Storage (S3 IA): ~$0.0125/GB/month
    *   Object Storage (S3 Glacier): ~$0.004/GB/month

**Data Volume:**
*   **Monthly Raw Data**: 2,000,000,000 events * 1 KB/event = **2,000 GB/month (2 TB)**
*   **Monthly Parquet Data**: 2,000 GB * 20% = **400 GB/month**

---

#### **Option 1: Kafka -> S3 -> Mongo Costs**

*   **MongoDB Storage (Hot - 90 days)**: 400 GB/month * 3 months = **1.2 TB**
*   **Object Storage Costs (Cold - 84 months)**:
    *   Months 1-3 (Standard): 400 GB * 3 * $0.023 = $27.60
    *   Months 4-12 (IA): 400 GB * 9 * $0.0125 = $45.00
    *   Months 13-84 (Glacier): 400 GB * 72 * $0.004 = $115.20
    *   **Total Cold Storage Cost (per month of data, over 7 years)**: ~$187.80
*   **Total Monthly Storage Cost (for new data)**: (MongoDB cost for 1.2TB) + (S3 cost for 400GB) ≈ **$1,000 + $9.20 = ~$1,009/month**

---

#### **Option 2: Kafka -> Mongo -> S3 Costs**

*   **MongoDB Storage (Hot - 14 months)**: 400 GB/month * 14 months = **5.6 TB**
*   **Object Storage Costs (Cold - 70 months)**:
    *   Calculated similarly to Option 1, but starting after month 14.
    *   **Total Cold Storage Cost (per month of data, over 7 years)**: ~$156.60
*   **Total Monthly Storage Cost (for new data)**: (MongoDB cost for 5.6TB) + (S3 cost for 400GB) ≈ **$4,600 + $9.20 = ~$4,609/month**

---

### 4.3. Recommendation

We recommend **Option 1 (Kafka -> Object Storage -> MongoDB)**.

**Justification:**

1.  **Significant Cost Savings**: This option is projected to be **~$3,600 cheaper per month** in storage costs alone, resulting in over **$43,000 in savings annually**. This is due to keeping only a small fraction of data in the more expensive MongoDB.
2.  **Superior Resilience and Durability**: Landing data in Object Storage first creates a robust, immutable source of truth. This decouples ingestion from processing, making the entire system more resilient to failures in downstream components.
3.  **Reduced Operational Risk**: The archival process is automated via a highly reliable, declarative S3 Lifecycle Policy. This is far less risky and complex to manage than the custom, multi-step archival script required for Option 2.
4.  **Future-Proof Architecture**: The data lake in Object Storage provides a flexible foundation for future analytics, machine learning, or compliance reporting without impacting the performance of the primary billing system.

While Option 1 introduces a slight delay in data availability and one more component to manage (the ingestion job), its benefits in cost, resilience, and operational simplicity far outweigh these minor trade-offs for a system of this scale and criticality.

## 5. Handling Late-Arriving Events

A robust billing system must account for events that arrive out of order or with a significant delay. For example, a network blip might cause an API usage event to be published to Kafka minutes or even hours after it occurred. If our aggregation jobs have already run for that period, we must have a mechanism to correct the billing records.

Here is how each option would handle this scenario.

---

### 5.1. Handling Late Events in Option 1 (Kafka -> Object Storage -> MongoDB)

This architecture is inherently more resilient to late-arriving data due to the stateless nature of its periodic batch processing.

**Strategy: Reprocess with Backfilling**

The core idea is that the aggregation job is designed to be **re-runnable** and idempotent for a given time window.

**Implementation Steps:**

1.  **Ingestion**: The late event is landed in the Object Storage bucket in the correct date-partitioned folder based on its `timestamp` (e.g., `.../year=2023/month=10/day=27/...`), regardless of when it arrives.
2.  **Scheduled Aggregation Job Modification**:
    *   The job is configured to process data for a specific time window (e.g., "yesterday").
    *   Crucially, the job's logic for updating the `monthly_bill_summary` collection must be an **upsert** operation based on the `billingPeriod` and `customerId`.
    *   The aggregation pipeline will first **delete** any existing usage summary for that customer and period before inserting the newly calculated one. This ensures the data is always a full recalculation.
3.  **Backfill Process**:
    *   If a late event from yesterday is detected, a manual or automated trigger can re-run the aggregation job specifically for the previous day's date.
    *   The job will re-read all the data for that day from Object Storage (including the newly arrived late event), perform the full aggregation, and overwrite the previous day's billing summary in MongoDB with the corrected totals.

**Example Workflow:**
*   **Day 1, 2:00 AM**: The aggregation job runs for `billingPeriod = "2023-10-26"`. It processes all data in the S3 partition for that day and creates a billing summary for `customer-123` showing 100 API calls.
*   **Day 1, 9:00 AM**: A late event from `2023-10-26` for `customer-123` arrives in Kafka and is written to the correct S3 partition.
*   **Action**: The aggregation job for `2023-10-26` is manually re-triggered.
*   **Result**: The job now reads 101 events for `customer-123` from S3. It deletes the previous summary and inserts a new one showing 101 calls. The billing record is now correct.

**Pros & Cons for Option 1:**

| Pros | Cons |
| --- | --- |
| **Highly Accurate**: Reprocessing the entire time window guarantees that the final result is correct. | **Higher Compute Cost**: Reprocessing can be computationally expensive if you have to re-scan large amounts of data. |
| **Conceptually Simple**: The logic is straightforward: "delete and replace" for a given period. | **Correction Latency**: There is a delay between the late event arriving and the corrected bill being generated. |
| **Immutable Source of Truth**: The raw data in S3 is never modified, providing a clear audit trail. | |

---

### 5.2. Handling Late Events in Option 2 (Kafka -> MongoDB -> Archive to S3)

This architecture is more challenging because data is being updated in near real-time. A simple batch reprocess is not as straightforward.

**Strategy: Incremental Updates with a "Correction Window"**

The core idea is to treat the `monthly_bill_summary` as a mutable document that can be incrementally updated, but only for a limited time.

**Implementation Steps:**

1.  **Ingestion**: The late event is inserted into the `raw_events` collection in MongoDB. Its `timestamp` is in the past.
2.  **Triggering a Correction**:
    *   A separate "correction" process is needed. This could be a scheduled job that runs more frequently (e.g., hourly) and looks for events that arrived in the last hour but have timestamps from the previous day.
    *   Alternatively, a change stream on the `raw_events` collection could trigger a correction function whenever a document with a past timestamp is inserted.
3.  **Incremental Update Logic**:
    *   When a late event is detected, the correction process does **not** perform a full re-aggregation.
    *   Instead, it performs an **incremental update** on the `monthly_bill_summary` document in MongoDB.
    *   Using MongoDB's update operators, it finds the correct billing summary document and increments the `callCount` and `totalCharge` for the relevant product.

**Example MongoDB Update Operation:**
```javascript
// Find the summary for customer-123 for the 2023-10 period
// and increment the payments API call count and charge.
db.monthly_bill_summary.updateOne(
   {
     "billingPeriod": "2023-10",
     "customerId": "customer-123",
     "usage.product": "payments"
   },
   {
     $inc: {
       "usage.$.callCount": 1,
       "usage.$.totalCharge": 0.02, // Rate for payments API
       "totalAmountDue": 0.02
     }
   }
);
```

4.  **Correction Window**: To prevent the system from having to keep old data "hot" indefinitely, we define a **correction window** (e.g., 72 hours). The correction process will only handle events with a timestamp within this window. Events arriving later than this would require a manual backfill process, similar to Option 1, but would be much more complex to implement.

**Pros & Cons for Option 2:**

| Pros | Cons | --- | --- |
| **Fast Correction**: Corrections are applied quickly via targeted updates, not full re-aggregation. | **High Complexity**: The incremental update logic is significantly more complex to write, test, and maintain than a "delete and replace" strategy. |
| **Low Compute Cost**: Only the specific late event is processed, not the entire day's data. | **Risk of Inconsistency**: If the correction logic fails or has a bug, the billing summary can become out of sync with the raw data, leading to hard-to-detect errors. |
| | **Limited Correction Window**: Not suitable for handling very late events (e.g., days old) without significant additional engineering effort. |

---

### 5.3. Comparison and Recommendation for Late Events

| Aspect | Option 1 (Reprocess) | Option 2 (Incremental Update) |
| --- | --- | --- |
| **Accuracy** | **Highest**. Guarantees correctness by recalculating from source. | **High**. Dependent on the correctness of the complex update logic. |
| **Complexity** | **Lower**. The aggregation logic is simpler, and the backfill mechanism is straightforward. | **Higher**. Requires building and maintaining a complex, stateful incremental update system. |
| **Operational Overhead** | **Lower**. Debugging is easier as you can re-run a process to verify results. | **Higher**. Debugging inconsistencies between raw and aggregated data is very difficult. |
| **Cost** | Higher compute cost during a reprocess. | Lower compute cost per correction, but higher engineering cost. |

**Recommendation:**

The **reprocessing strategy in Option 1** is strongly recommended. While it can be more computationally expensive during a correction, its simplicity, accuracy, and robustness make it far superior for a financial system like billing. The risk of billing errors introduced by the complex incremental update logic in Option 2 is unacceptably high. The reprocessing approach provides a clear, auditable, and guaranteed path to correct billing records, which is paramount.

## 6. Proactive Reprocessing using Object Storage Event Notifications

This mechanism acts as a trigger for your backfill process, making it event-driven rather than purely scheduled.

### 6.1. How It Works (The General Pattern)

1.  **Event Notification**: You configure your Object Storage bucket to send a notification for every object created (`s3:ObjectCreated:*`).
2.  **Trigger**: This notification triggers a serverless function (e.g., AWS Lambda, Google Cloud Function).
3.  **Logic in the Function**: The function executes the core logic:
    *   It inspects the object key (e.g., `.../year=2023/month=10/day=27/...`) to extract the date of the data.
    *   It compares this data date to the current date.
    *   If the data date is older than a defined threshold (e.g., "older than 2 hours" or "not today"), it flags the partition for reprocessing.
4.  **Action**: The function then triggers the reprocessing workflow for that specific partition.

### 6.2. Implementation Details (Using AWS as an example)

1.  **S3 Bucket Notification**:
    *   In your S3 bucket properties, go to "Event notifications".
    *   Create a new notification with:
        *   **Event name**: `LateDataDetector`
        *   **Events**: `s3:ObjectCreated:*`
        *   **Destination**: Lambda function

2.  **AWS Lambda Function (`LateDataHandler`)**:
    *   **Runtime**: Python or Node.js are excellent choices.
    *   **Permissions**: The function needs `s3:GetObject` permissions to read the object metadata (it doesn't need to read the whole file) and permissions to invoke the next step (e.g., AWS Step Functions, another Lambda, or a message queue).
    *   **Code Logic (Pseudocode)**:

    ```python
    import datetime
    import os

    # Define your acceptable delay, e.g., 2 hours
    MAX_DELAY_HOURS = int(os.environ['MAX_DELAY_HOURS'])

    def lambda_handler(event, context):
        # Get the object key from the S3 event
        bucket = event['Records'][0]['s3']['bucket']['name']
        key = event['Records'][0]['s3']['object']['key']

        # Extract date from the partitioned path, e.g., ".../year=2023/month=10/day=27/file.parquet"
        try:
            parts = key.split('/')
            year_part = [p for p in parts if p.startswith('year=')][0]
            month_part = [p for p in parts if p.startswith('month=')][0]
            day_part = [p for p in parts if p.startswith('day=')][0]

            data_date = datetime.date(
                int(year_part.split('=')[1]),
                int(month_part.split('=')[1]),
                int(day_part.split('=')[1])
            )
        except (IndexError, ValueError):
            print(f"Could not parse date from key: {key}")
            return {'statusCode': 400, 'body': 'Invalid key format'}

        # Compare with today's date
        today = datetime.date.today()
        delta = today - data_date

        if delta.days > 0 or (delta.days == 0 and datetime.datetime.now().hour > MAX_DELAY_HOURS):
            print(f"LATE DATA DETECTED! Partition for {data_date} received on {today}. Triggering reprocess.")
            # Trigger the reprocessing job
            trigger_reprocess_for_partition(year=data_date.year, month=data_date.month, day=data_date.day)
            return {'statusCode': 200, 'body': 'Late data detected and reprocess triggered.'}
        else:
            print(f"Data is on-time. Partition for {data_date}. No action needed.")
            return {'statusCode': 200, 'body': 'Data is on-time.'}

    def trigger_reprocess_for_partition(year, month, day):
        # This is where you call your reprocessing mechanism.
        # Examples:
        # - Publish a message to an SQS queue with the partition details.
        # - Invoke another Lambda function that runs the aggregation.
        # - Start an AWS Step Functions execution.
        print(f"Triggering reprocess for {year}-{month:02d}-{day:02d}")
        # ... implementation of trigger
    ```

---

## 7. Integration with Architectural Options

This event-driven reprocessing mechanism fits perfectly with **Option 1** and provides a major advantage.

### 7.1. Integration with Option 1 (Kafka -> S3 -> MongoDB)

This is the ideal use case for this pattern.

*   **Flow**:
    1.  A late event for `2023-10-26` arrives and is written to the S3 partition `.../year=2023/month=10/day=26/`.
    2.  The `LateDataHandler` Lambda function is triggered.
    3.  It detects that the data is for a past day and immediately calls your **reprocessing job**, passing the parameters `year=2023, month=10, day=26`.
    4.  The reprocessing job runs its aggregation pipeline *only* for that specific day's partition, overwriting the previous day's summary in MongoDB with the corrected totals.

*   **Benefit**: This creates a highly efficient, automated, and low-latency correction system. You don't have to wait for a scheduled job to discover the discrepancy; you fix it as soon as the bad data arrives.

### 7.2. Integration with Option 2 (Kafka -> MongoDB -> S3)

This mechanism is **less useful** for Option 2, but can still provide value.

*   **Flow**:
    1.  A late event for `2023-10-26` arrives and is written to the S3 partition `.../year=2023/month=10/day=26/`.
    2.  The `LateDataHandler` Lambda function is triggered.
    3.  It detects the late data. However, the data has *already* been inserted into MongoDB via the Kafka connector.
    4.  The Lambda function's action would now be different. It could:
        *   **Trigger the Incremental Update Logic**: It could invoke the complex incremental update function, telling it to apply a correction for the specific event details (if it can get them).
        *   **Send an Alert**: More simply, it could just send a notification (e.g., to Slack or PagerDuty) so an operator knows a manual correction may be needed.

*   **Benefit**: It provides visibility into late data, but it doesn't solve the core problem elegantly. The primary correction mechanism in Option 2 remains the complex incremental update logic, which this event can only help trigger. It doesn't simplify the architecture.

---

## 8. Conclusion and Final Recommendation

Implementing an event-driven alerting mechanism on your Object Storage partitions is a powerful way to manage late-arriving data.

*   It **perfectly complements Option 1**, creating a robust, automated, and event-driven system for ensuring billing accuracy. The "reprocess" action is simple, reliable, and a natural fit for this trigger.
*   For Option 2, it provides **limited value**. It can alert you to a problem but doesn't simplify the inherently complex and risky incremental update logic required to fix it.

This further strengthens the recommendation for **Option 1 (Kafka -> Object Storage -> MongoDB)**. The combination of a simple, re-runnable aggregation job and an event-driven trigger to run it creates a system that is not only cost-effective and resilient but also highly intelligent and self-correcting.
