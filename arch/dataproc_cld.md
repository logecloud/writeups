# Open Banking API Billing System - Design Document

## 1. Executive Summary

This document outlines the design for a data aggregation and billing system for the bank's open banking APIs. The system will collect billable events from API Gateway (Apigee) and the Consent Management System, aggregate them, and produce monthly billing statements for third-party applications and aggregators.

---

## 2. Business Requirements

### 2.1 Functional Requirements

| Requirement | Description |
|------------|-------------|
| **Event Capture** | Capture all API calls from Apigee and consent events from the Consent System |
| **Multi-party Tracking** | Track all parties in the transaction chain: Bank → Customer → Aggregator → Application |
| **Differential Pricing** | Support different pricing for different APIs (e.g., Accounts API: $0.01/call, Payments API: $0.02/call) |
| **Billable/Non-billable** | Some APIs are free, some are chargeable |
| **Monthly Billing** | Generate monthly billing statements with detailed breakdowns |
| **Audit Trail** | Maintain detailed records for dispute resolution |
| **Data Retention** | Keep data for 14 months (12 months + 2 months), then archive to cold storage |

### 2.2 Non-Functional Requirements

| Requirement | Specification |
|------------|---------------|
| **Volume** | 2 billion API calls per month |
| **Throughput** | ~770 calls/second average, ~3,000 calls/second peak |
| **Latency** | Event processing within 5 minutes |
| **Availability** | 99.9% uptime |
| **Data Accuracy** | 100% - no missed or duplicate billing events |
| **Query Performance** | Monthly billing generation < 1 hour |

### 2.3 Technical Constraints

- **Message Broker**: Kafka (mandatory)
- **Database**: MongoDB (mandatory)
- **Processing**: Avoid Apache Flink (complexity concerns)
- **Storage**: Object storage (S3/equivalent) for archives

---

## 3. System Context

### 3.1 Event Sources

**Apigee API Gateway**
- Posts message to Kafka on every API call
- Contains: product, API name, endpoint, customer, application, aggregator, response code

**Consent Management System**
- Posts message to Kafka for consent lifecycle events
- Events: consent creation, renewal, revocation
- Each event is billable

### 3.2 Party Model

```
┌──────────────┐
│     Bank     │ (1st Party - Us)
└──────┬───────┘
       │
┌──────▼───────┐
│   Customer   │ (2nd Party - End User)
└──────┬───────┘
       │
       ├─────────────────┐
       │                 │
┌──────▼───────┐  ┌──────▼───────┐
│  Aggregator  │  │ Application  │ (3rd/4th Party)
│   (Plaid)    │  │ (Robinhood)  │
└──────────────┘  └──────────────┘
                         │
                  (Billing Entity)
```

**Billing Logic**: 
- Application (e.g., Robinhood) is typically the billing entity
- May use an aggregator (e.g., Plaid) internally
- System tracks all parties for analytics and flexible billing models

---

## 4. Late-Arriving Events Handling

### 4.1 Problem Statement

Late-arriving events occur when:
- Network delays cause events to arrive out of order
- System failures lead to message replay from Kafka
- Clock skew between Apigee nodes causes timestamp inconsistencies
- Batch processing delays in upstream systems

**Example Scenario:**
- Event with timestamp `2025-11-14 23:59:50` arrives on `2025-11-15 00:05:00`
- This event should be billed in November, but arrives after November processing

### 4.2 Business Impact

| Scenario | Impact | Acceptable Lateness |
|----------|--------|---------------------|
| Within same day | Low - daily aggregates not finalized | Up to 24 hours |
| Within same month | Medium - affects monthly billing | Up to 72 hours |
| After month closed | High - billing already finalized | Up to 7 days (grace period) |

### 4.3 Handling Strategy

**Detection Window**: Events can arrive up to 7 days late
**Grace Period**: Monthly billing remains "open" for 7 days after month end
**Billing Finalization**: Bills are finalized on the 8th of the following month

---

## 5. Architecture Options

### Option 1: Kafka → Object Storage → MongoDB

#### 4.1.1 Architecture Diagram

```
┌─────────────┐
│   Apigee    │──┐
└─────────────┘  │
                 ├──► Kafka Topics
┌─────────────┐  │    (api-calls-raw, consent-events-raw)
│  Consent    │──┘
│  System     │
└─────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Kafka Connect  │
        │   S3 Sink      │
        └────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │ Object Storage │ (Raw Events)
        │   (S3/Minio)   │
        │  - Partitioned │
        │    by date     │
        └────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │  Batch ETL Job │
        │  (Hourly/Daily)│
        │  - Read S3     │
        │  - Transform   │
        │  - Aggregate   │
        └────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │    MongoDB     │
        │  - Daily aggs  │
        │  - Monthly bill│
        └────────────────┘
```

#### 4.1.2 Data Flow

1. **Ingestion Layer**
   - Kafka receives events from Apigee and Consent System
   - Kafka Connect S3 Sink writes raw events to object storage
   - Partitioned by date: `s3://bucket/year=2025/month=11/day=14/`
   - Format: JSON or Parquet

2. **Processing Layer**
   - Scheduled batch job (hourly or daily)
   - Reads raw events from S3
   - Enriches with pricing configuration
   - Aggregates by day/customer/application/product/API
   - Writes to MongoDB

3. **Storage Layer**
   - MongoDB stores only aggregated data
   - Much smaller footprint
   - Fast query performance

4. **Archival**
   - Raw S3 data moves to Glacier/cold storage after 14 months

#### 4.1.3 MongoDB Schema (Simplified)

```javascript
// Collection: api_calls_daily_agg
{
  _id: ObjectId("..."),
  date: ISODate("2025-11-14"),
  product_code: "accounts",
  api_name: "account_balance",
  application_id: "robinhood_abc",
  customer_id: "cust_12345",
  aggregator_id: "plaid_xyz",
  call_count: 1523,
  billable_count: 1523,
  total_cents: 1523
}

// Collection: monthly_billing
{
  _id: ObjectId("..."),
  billing_month: "2025-11",
  billing_entity_id: "robinhood_abc",
  api_breakdown: [...],
  total_cents: 2110000
}
```

#### 4.1.4 Storage Calculations

**Object Storage (S3)**
| Component | Calculation | Size |
|-----------|-------------|------|
| Raw events per month | 2B events × 500 bytes | 1 TB |
| 14 months retention | 1 TB × 14 | 14 TB |
| Compression (gzip/parquet) | 14 TB × 0.3 | **4.2 TB** |

**MongoDB**
| Component | Calculation | Size |
|-----------|-------------|------|
| Daily aggregates | 60M docs/month × 200 bytes × 14 months | 168 GB |
| Monthly billing | 10K entities × 50KB × 14 months | 7 GB |
| Indexes (30% overhead) | 175 GB × 1.3 | 52 GB |
| **Total MongoDB** | | **227 GB** |

**Total Storage: 4.2 TB (S3) + 227 GB (MongoDB) = ~4.5 TB**

#### 4.1.5 Cost Analysis (AWS Example)

| Component | Specification | Monthly Cost |
|-----------|---------------|--------------|
| **Kafka** | 3 brokers (m5.large) | $292 |
| **S3 Standard** | 300 GB active (current month) | $7 |
| **S3 Intelligent Tiering** | 3.9 TB (older data) | $53 |
| **MongoDB** | 3-node replica set (r5.xlarge) | $876 |
| **ETL Compute** | Lambda/Fargate for batch jobs | $150 |
| **Data Transfer** | S3→MongoDB, minimal | $20 |
| **Total** | | **~$1,400/month** |

#### 4.1.6 Pros & Cons

**Pros:**
- ✅ **Durable landing zone**: Raw events safely stored in S3 before processing
- ✅ **Reprocessing capability**: Can reprocess from S3 if needed
- ✅ **Small MongoDB footprint**: Only aggregated data
- ✅ **Cost-effective storage**: S3 cheaper than database storage
- ✅ **Schema flexibility**: Can change aggregation logic without losing raw data

**Cons:**
- ❌ **Processing delay**: Hourly/daily batch processing = longer time to query
- ❌ **Complexity**: Additional component (S3) and ETL pipeline
- ❌ **Limited real-time**: Cannot query recent events until batch processes
- ❌ **ETL maintenance**: Need to maintain batch processing jobs
- ❌ **Two-step failure**: Can fail at Kafka→S3 or S3→MongoDB

---

### Option 2: Kafka → MongoDB → Archive to Object Storage (RECOMMENDED)

#### 4.2.1 Architecture Diagram

```
┌─────────────┐
│   Apigee    │──┐
└─────────────┘  │
                 ├──► Kafka Topics
┌─────────────┐  │    (3-5 brokers, 3+ partitions)
│  Consent    │──┘
│  System     │
└─────────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │ Kafka Consumer Group   │
        │ (5-10 instances)       │
        │ - Read from Kafka      │
        │ - Enrich with pricing  │
        │ - Bulk write to Mongo  │
        │ - Real-time aggregation│
        └────────────────────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │  MongoDB Cluster       │
        │  (Sharded/Replica Set) │
        │                        │
        │ - api_calls_raw (TTL)  │
        │ - api_calls_daily      │
        │ - api_calls_detail     │
        │ - consent_events       │
        │ - monthly_billing      │
        └────────────────────────┘
                 │
                 ▼ (after 14 months)
        ┌────────────────────────┐
        │  Archival Process      │
        │  (Scheduled Job)       │
        │  - Export to S3        │
        │  - Mark as archived    │
        │  - Delete after grace  │
        └────────────────────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │   Object Storage       │
        │   (Cold Archive)       │
        │   - Parquet format     │
        │   - Glacier/Deep Arch  │
        └────────────────────────┘
```

#### 4.2.2 Data Flow

1. **Ingestion Layer**
   - Kafka receives events (770 avg, 3000 peak msg/sec)
   - Consumer group with 5-10 parallel consumers
   - Batch size: 1000 messages, max wait: 5 seconds

2. **Real-time Processing**
   - Consumer enriches events with pricing from MongoDB config collection
   - Determines billable status and rate
   - Writes to MongoDB in three stages:
     - **Landing zone**: Raw events (TTL 7 days)
     - **Daily aggregates**: Pre-aggregated for billing queries
     - **Detail buckets**: Hourly buckets for audit trail

3. **Storage Layer - MongoDB Collections**
   - `api_calls_raw`: Landing zone with 7-day TTL
   - `api_calls_daily`: Daily aggregates (main billing source)
   - `api_calls_detail`: Hourly event buckets (10K events/bucket)
   - `consent_events`: Individual consent lifecycle events
   - `monthly_billing`: Final monthly billing aggregates
   - `api_configuration`: Pricing rules

4. **Monthly Aggregation**
   - Scheduled job runs on 1st of month
   - Reads from daily aggregates
   - Generates monthly billing documents
   - Fast processing (< 1 hour)

5. **Archival Process**
   - Daily job checks for data > 14 months old
   - Exports to S3 in Parquet format
   - Marks records as archived in MongoDB
   - Deletes archived data after 30-day grace period
   - S3 lifecycle policy moves to Glacier after 90 days

#### 4.2.3 MongoDB Schema (Detailed)

```javascript
// Collection 1: api_calls_raw (Landing zone - 7 day TTL)
{
  _id: ObjectId("..."),
  event_id: "uuid-string",
  received_at: ISODate("2025-11-14T10:30:00Z"),
  event_data: {
    timestamp: ISODate("2025-11-14T10:30:00Z"),
    product_code: "accounts",
    api_name: "account_balance",
    endpoint: "/v1/accounts/123/balance",
    method: "GET",
    response_code: 200,
    bank_id: "bank_001",
    customer_id: "cust_12345",
    aggregator_id: "plaid_xyz",
    application_id: "robinhood_abc",
    consent_id: "consent_789",
    billable: true,
    rate_cents: 1
  },
  processed: false
}
// TTL Index: { "received_at": 1 }, expireAfterSeconds: 604800

// Collection 2: api_calls_daily (Main billing source)
{
  _id: ObjectId("..."),
  date: ISODate("2025-11-14T00:00:00Z"),
  product_code: "accounts",
  api_name: "account_balance",
  customer_id: "cust_12345",
  application_id: "robinhood_abc",
  aggregator_id: "plaid_xyz",
  call_count: 1523,
  billable_count: 1523,
  total_cents: 1523,
  sample_event_ids: ["uuid1", "uuid2", "uuid3"], // First 10
  last_updated: ISODate("2025-11-14T23:59:00Z"),
  archived: false
}
// Index: { "date": 1, "application_id": 1, "product_code": 1 }

// Collection 3: api_calls_detail (Audit trail)
{
  _id: ObjectId("..."),
  hour_bucket: ISODate("2025-11-14T10:00:00Z"),
  application_id: "robinhood_abc",
  events: [
    {
      event_id: "uuid1",
      timestamp: ISODate("2025-11-14T10:30:15Z"),
      product_code: "accounts",
      api_name: "account_balance",
      endpoint: "/v1/accounts/123/balance",
      customer_id: "cust_12345",
      aggregator_id: "plaid_xyz",
      billable: true,
      rate_cents: 1,
      response_code: 200
    }
    // ... up to 10,000 events per bucket
  ],
  event_count: 8523,
  created_at: ISODate("2025-11-14T10:00:00Z"),
  archived: false
}
// Index: { "hour_bucket": 1, "application_id": 1 }

// Collection 4: consent_events
{
  _id: ObjectId("..."),
  event_id: "uuid",
  timestamp: ISODate("2025-11-14T10:30:00Z"),
  consent_id: "consent_789",
  event_type: "created",
  customer_id: "cust_12345",
  application_id: "robinhood_abc",
  aggregator_id: "plaid_xyz",
  products: ["accounts", "payments"],
  billable: true,
  rate_cents: 50,
  validity_period_days: 90,
  archived: false
}

// Collection 5: monthly_billing
{
  _id: ObjectId("..."),
  billing_month: "2025-11",
  billing_entity_id: "robinhood_abc",
  entity_type: "application",
  api_breakdown: [
    {
      product_code: "accounts",
      api_calls: {
        "account_balance": { 
          count: 1500000, 
          rate_cents: 1, 
          total_cents: 1500000 
        },
        "account_details": { 
          count: 500000, 
          rate_cents: 1, 
          total_cents: 500000 
        }
      },
      product_total_cents: 2000000
    }
  ],
  api_call_total_count: 2050000,
  api_call_total_cents: 2100000,
  consent_events: {
    created: { count: 150, rate_cents: 50, total_cents: 7500 }
  },
  consent_total_cents: 10000,
  total_cents: 2110000,
  generated_at: ISODate("2025-12-01T00:00:00Z"),
  finalized: false
}

// Collection 6: api_configuration (Pricing)
{
  _id: ObjectId("..."),
  product_code: "accounts",
  api_name: "account_balance",
  endpoint_pattern: "/v1/accounts/{id}/balance",
  billable: true,
  rate_cents: 1,
  effective_from: ISODate("2025-01-01T00:00:00Z"),
  effective_to: null
}
```

#### 4.2.4 Storage Calculations

**MongoDB (Hot Data - 14 months)**

| Collection | Calculation | Size |
|-----------|-------------|------|
| **api_calls_raw** | 2B/month × 500 bytes × 7 days retention | 230 GB (rotating) |
| **api_calls_daily** | 60M docs/month × 200 bytes × 14 months | 168 GB |
| **api_calls_detail** | 145K buckets/month × 50KB avg × 14 months | 100 GB |
| **consent_events** | 10M events/month × 300 bytes × 14 months | 42 GB |
| **monthly_billing** | 10K entities × 50KB × 14 months | 7 GB |
| **api_configuration** | Master data | 1 GB |
| **Subtotal (data)** | | **548 GB** |
| **Indexes (30%)** | | **164 GB** |
| **Working set overhead** | | **88 GB** |
| **Total MongoDB** | | **800 GB** |

**Object Storage (Cold Archive - After 14 months)**

| Component | Calculation | Annual Growth |
|-----------|-------------|---------------|
| Daily aggregates | 60M × 200 bytes × 12 months | 144 GB/year |
| Detail data | 145K × 50KB × 12 months | 87 GB/year |
| Consent events | 10M × 300 bytes × 12 months | 36 GB/year |
| Compression (Parquet) | 267 GB × 0.3 | **80 GB/year** |

After 5 years: 80 GB × 5 years = 400 GB in cold storage

**Total Storage:**
- **Active (MongoDB)**: 800 GB
- **Cold Archive (S3 Glacier)**: Grows 80 GB/year

#### 4.2.5 Infrastructure Specifications

**Kafka Cluster**
- 3-5 brokers (m5.large or equivalent)
- 3+ partitions per topic for parallelism
- Retention: 24-48 hours (raw events)
- Replication factor: 3

**MongoDB Cluster**
- **For < 1TB**: 3-node replica set
  - Primary: r5.2xlarge (8 vCPU, 64GB RAM)
  - Secondaries: r5.xlarge (4 vCPU, 32GB RAM)
  - Storage: 2TB NVMe SSD per node
  
- **For > 1TB**: Sharded cluster
  - 2 shards, 3 nodes per shard
  - Config servers: 3 nodes
  - Mongos routers: 2-3 instances
  - Shard key: `{date: 1, application_id: 1}`

**Kafka Consumers**
- 5-10 instances (t3.medium or equivalent)
- Auto-scaling based on Kafka lag
- Deployment: Kubernetes pods or ECS tasks

**Batch Jobs (Monthly aggregation, Archival)**
- Serverless (Lambda/Cloud Functions) or
- Scheduled container tasks (ECS/Cloud Run)
- Low cost as they run infrequently

#### 4.2.6 Cost Analysis (AWS Example)

| Component | Specification | Monthly Cost |
|-----------|---------------|--------------|
| **Kafka** | 3 brokers (m5.large) | $292 |
| **MongoDB** | 3-node replica set (r5.2xlarge + 2×r5.xlarge) | $1,314 |
| **MongoDB Storage** | 6TB NVMe (2TB × 3 nodes) | $600 |
| **Kafka Consumers** | 8 instances (t3.medium) | $237 |
| **Batch Jobs** | Lambda/ECS for monthly/archival | $50 |
| **S3 Glacier** | 400 GB (after 5 years) | $1.60 |
| **Data Transfer** | Minimal (within VPC) | $20 |
| **CloudWatch/Monitoring** | Logs and metrics | $100 |
| **Total** | | **~$2,615/month** |

**Cost Notes:**
- Year 1: ~$2,615/month (minimal cold storage)
- Year 5: ~$2,620/month (400 GB cold storage = +$5)
- MongoDB is the largest cost component (~73%)
- Scales linearly with data volume

#### 4.2.7 Pros & Cons

**Pros:**
- ✅ **Real-time processing**: Events processed within seconds
- ✅ **Immediate queryability**: Data available for queries immediately
- ✅ **Simpler architecture**: Direct Kafka→MongoDB flow
- ✅ **Pre-aggregated data**: Fast billing queries (aggregation at write time)
- ✅ **Single source of truth**: MongoDB is the primary data store
- ✅ **Operational simplicity**: Fewer components to manage
- ✅ **Better failure handling**: Kafka offset management ensures no data loss

**Cons:**
- ❌ **Higher MongoDB costs**: More storage in database vs. object storage
- ❌ **Limited reprocessing**: Must replay from Kafka (48hr retention) or backups
- ❌ **Database scaling**: May need sharding as data grows
- ❌ **Initial complexity**: Multi-level storage strategy in MongoDB

---

## 6. Recommendation

**Recommended: Option 2 (Kafka → MongoDB → Archive to Object Storage)**

### Rationale:

1. **Business Requirements Alignment**
   - Real-time processing meets the need for timely billing queries
   - Pre-aggregated daily data enables fast monthly billing generation
   - Detailed event storage supports dispute resolution

2. **Operational Excellence**
   - Simpler architecture = easier to operate and maintain
   - Fewer failure points = higher reliability
   - Standard Kafka consumer pattern = well-understood operations

3. **Cost-Benefit Analysis**
   - Extra $1,200/month (~86% more) for MongoDB storage
   - Significantly simpler operations (estimated 40% less engineering time)
   - Real-time capabilities enable future use cases (dashboards, alerts)
   - Cost is predictable and scales linearly

4. **Late-Arriving Events**
   - Real-time detection and handling
   - 7-day grace period for monthly billing
   - Automatic adjustments within grace period
   - Clear audit trail for post-finalization events
   - Monitoring dashboard for lateness patterns
   - MongoDB's document model is perfect for nested billing structures
   - Sharding provides clear horizontal scaling path
   - No complex ETL pipeline to maintain
   - Immediate data availability for customer queries

5. **Technical Advantages**
   - Kafka provides replay capability (48 hours)
   - MongoDB backups enable disaster recovery
   - Archival to S3 provides long-term audit trail
   - Proven technology stack (Kafka + MongoDB)

6. **Risk Mitigation**

- If cost is the primary concern and real-time processing is not needed
- If you anticipate frequent reprocessing with changing business logic
- If you have existing S3-based data lake infrastructure
- If you have strong ETL engineering capabilities

---

## 7. Implementation Roadmap

### Phase 1: MVP (Months 1-2)
- ✅ Setup Kafka cluster and topics
- ✅ Setup MongoDB replica set
- ✅ Implement basic Kafka consumer with late-arrival detection
- ✅ Create core collections (daily aggregates, config)
- ✅ Basic monthly billing aggregation with grace period
- ✅ Late arrival monitoring collection

### Phase 2: Production (Months 3-4)
- ✅ Add detail collection for audit trail
- ✅ Implement archival process
- ✅ Add monitoring and alerting (including late-arrival alerts)
- ✅ Performance tuning and optimization
- ✅ Load testing (2B events/month simulation)
- ✅ Implement billing_adjustments and credit_notes collections
- ✅ Daily reconciliation process

### Phase 3: Optimization (Months 5-6)
- ✅ Implement sharding (if needed)
- ✅ Build billing dashboard/API
- ✅ Automated reconciliation
- ✅ Anomaly detection

---

## 8. Key Metrics and Monitoring

### 8.1 SLIs (Service Level Indicators)

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Kafka Consumer Lag | < 10,000 messages | > 50,000 messages |
| Event Processing Time | < 5 minutes | > 15 minutes |
| MongoDB Write Latency | < 100ms (p99) | > 500ms (p99) |
| Daily Aggregate Accuracy | 100% | < 99.99% |
| Monthly Billing Generation | < 1 hour | > 2 hours |
| Data Loss | 0 events | Any data loss |

### 8.2 Operational Dashboards

**Real-time Monitoring:**
- Kafka topics throughput and lag
- Consumer group status and lag
- MongoDB write throughput and latency
- Error rates and dead letter queue size

**Business Monitoring:**
- Daily API call volume by product
- Daily revenue by application
- Top consumers (by volume and revenue)
- Pricing configuration changes
- **Late arrival statistics (by time bucket)**
- **Grace period utilization**
- **Billing adjustment trends**

---

## 9. Risk Assessment

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| MongoDB storage growth exceeds capacity | Medium | High | Implement sharding, monitor growth, archival automation |
| Kafka consumer falls behind | Medium | High | Auto-scaling, alerting, multiple consumer instances |
| Pricing configuration errors | Low | High | Approval workflow, audit log, version control |
| Duplicate billing events | Low | Critical | Event ID deduplication, idempotent writes |
| Archive process failure | Low | Medium | Monitoring, alerts, manual retry capability |
| MongoDB primary failure | Low | High | Auto-failover replica set, backups |
| Late events exceeding grace period | Medium | Medium | Clear policy, adjustment process, monitoring |
| Clock skew in source systems | Low | High | NTP synchronization, timestamp validation |

---

## 10. Appendix

### A. Sample Event Payloads

**API Call Event (from Apigee)**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2025-11-14T10:30:15.123Z",
  "product_code": "accounts",
  "api_name": "account_balance",
  "endpoint": "/v1/accounts/ACC123/balance",
  "method": "GET",
  "response_code": 200,
  "response_time_ms": 45,
  "bank_id": "bank_001",
  "customer_id": "cust_12345",
  "aggregator_id": "plaid_xyz",
  "application_id": "robinhood_abc",
  "consent_id": "consent_789",
  "request_id": "req_abc123"
}
```

**Consent Event**
```json
{
  "event_id": "660e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2025-11-14T09:15:30.456Z",
  "consent_id": "consent_789",
  "event_type": "created",
  "customer_id": "cust_12345",
  "application_id": "robinhood_abc",
  "aggregator_id": "plaid_xyz",
  "products": ["accounts", "payments"],
  "scopes": ["read_accounts", "read_balances", "initiate_payments"],
  "validity_period_days": 90,
  "expiry_date": "2026-02-12T09:15:30.456Z"
}
```

### B. Query Examples

**Get monthly billing for an application:**
```javascript
db.monthly_billing.findOne({
  billing_month: "2025-11",
  billing_entity_id: "robinhood_abc"
})
```

**Get daily API usage for a customer:**
```javascript
db.api_calls_daily.aggregate([
  {
    $match: {
      date: { 
        $gte: ISODate("2025-11-01"), 
        $lt: ISODate("2025-12-01") 
      },
      customer_id: "cust_12345"
    }
  },
  {
    $group: {
      _id: "$api_name",
      total_calls: { $sum: "$call_count" },
      total_cost_cents: { $sum: "$total_cents" }
    }
  }
])
```

**Get late arrivals summary:**
```javascript
db.late_arrival_monitoring.aggregate([
  {
    $match: {
      date: { 
        $gte: ISODate("2025-11-01"), 
        $lt: ISODate("2025-12-01") 
      }
    }
  },
  {
    $project: {
      date: 1,
      total_late: {
        $add: [
          "$late_arrivals.5-60min",
          "$late_arrivals.1-24hr",
          "$late_arrivals.1-7days",
          "$late_arrivals.>7days"
        ]
      },
      critical_late: "$late_arrivals.>7days"
    }
  },
  {
    $group: {
      _id: null,
      total_late_events: { $sum: "$total_late" },
      critical_late_events: { $sum: "$critical_late" }
    }
  }
])
```

**Get billing adjustments for a month:**
```javascript
db.billing_adjustments.find({
  affected_month: "2025-11",
  applied_to_billing: false
}).sort({ processing_timestamp: -1 })
```
