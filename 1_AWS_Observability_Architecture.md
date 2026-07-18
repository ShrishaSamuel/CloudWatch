# Production-Grade Observability Architecture
## AWS E-Commerce Microservices Platform

> **Role**: Senior AWS Observability Architect
> **Date**: 2026-07-18
> **Scope**: Full-Stack Observability — Metrics, Logs, Traces, RUM

---

## Table of Contents

1. [Architecture Diagram](#1-architecture-diagram)
2. [Component Interaction Flow](#2-component-interaction-flow)
3. [Monitoring Data Flow](#3-monitoring-data-flow)
4. [Logging Architecture](#4-logging-architecture)
5. [Tracing Architecture](#5-tracing-architecture)
6. [Security Architecture](#6-security-architecture)
7. [Multi-AZ Design](#7-multi-az-design)
8. [High Availability Considerations](#8-high-availability-considerations)
9. [AWS Well-Architected Best Practices](#9-aws-well-architected-best-practices)

---

## 1. Architecture Diagram

### 1.1 High-Level Observability Architecture

```mermaid
graph TB
    subgraph USER["👤 End Users / Browsers"]
        Browser["React Frontend\n(CloudFront + S3)"]
    end

    subgraph INGRESS["Ingress Layer"]
        RUM["CloudWatch RUM\n(Real User Monitoring)"]
        APIGW["API Gateway\n(REST / HTTP API)"]
        ALB["Application Load Balancer"]
    end

    subgraph COMPUTE["Compute Layer — Amazon EKS (Multi-AZ)"]
        direction TB
        subgraph AZ1["Availability Zone 1"]
            SVC_A["Product Service\n(Pod)"]
            SVC_B["Order Service\n(Pod)"]
        end
        subgraph AZ2["Availability Zone 2"]
            SVC_C["Cart Service\n(Pod)"]
            SVC_D["Payment Service\n(Pod)"]
        end
        subgraph AZ3["Availability Zone 3"]
            SVC_E["Notification Service\n(Pod)"]
            SVC_F["Search Service\n(Pod)"]
        end
        ADOT_COL["ADOT Collector\n(DaemonSet)"]
    end

    subgraph SERVERLESS["Serverless Layer"]
        LAMBDA_A["Lambda: Order Processor"]
        LAMBDA_B["Lambda: Email Notifier"]
        LAMBDA_C["Lambda: Inventory Sync"]
    end

    subgraph DATA["Data Layer"]
        RDS_W["RDS Aurora (Writer)\nPostgreSQL"]
        RDS_R["RDS Aurora (Reader)\nRead Replica"]
        DDB["DynamoDB\n(Sessions / Cart)"]
        ELASTICACHE["ElastiCache Redis\n(Cache Layer)"]
    end

    subgraph OBSERVABILITY["Observability Platform"]
        direction TB
        subgraph COLLECTION["Signal Collection"]
            CW_AGENT["CloudWatch Agent\n(EKS Node-level)"]
            XRAY["AWS X-Ray\n(Distributed Tracing)"]
            CONT_INS["Container Insights\n(EKS Enhanced)"]
            APP_SIG["Application Signals\n(SLI/SLO Engine)"]
        end
        subgraph STORAGE["Observability Storage"]
            CW_LOGS["CloudWatch Logs\n(Log Groups)"]
            CW_METRICS["CloudWatch Metrics\n(Custom + Platform)"]
            CW_TRACES["X-Ray Traces\n(Service Map)"]
        end
        subgraph ALERTING["Alerting Engine"]
            CW_ALARMS["CloudWatch Alarms\n(Composite + Metric)"]
            SNS["Amazon SNS\n(Alert Routing)"]
            EVENTBRIDGE["EventBridge\n(Event Automation)"]
        end
        subgraph VISUALIZATION["Visualization"]
            GRAFANA["Amazon Managed Grafana\n(Dashboards)"]
            CW_DASH["CloudWatch Dashboards\n(Ops View)"]
        end
    end

    subgraph SECURITY["Security & Compliance"]
        CLOUDTRAIL["AWS CloudTrail\n(API Audit)"]
        CONFIG["AWS Config\n(Compliance)"]
        SECURITYHUB["Security Hub\n(Aggregation)"]
        GUARDDUTY["GuardDuty\n(Threat Detection)"]
    end

    Browser -->|"JS Beacon\n(performance, errors)"| RUM
    Browser --> APIGW
    APIGW --> ALB
    ALB --> SVC_A & SVC_B & SVC_C & SVC_D
    SVC_A & SVC_B & SVC_C & SVC_D --> LAMBDA_A & LAMBDA_B & LAMBDA_C
    SVC_A & SVC_B --> RDS_W
    RDS_W --> RDS_R
    SVC_C & SVC_D --> DDB
    SVC_A --> ELASTICACHE

    ADOT_COL -->|"OTLP Traces"| XRAY
    ADOT_COL -->|"OTLP Metrics"| CW_METRICS
    CW_AGENT --> CW_LOGS & CW_METRICS
    CONT_INS --> CW_METRICS & CW_LOGS
    APP_SIG --> CW_METRICS
    RUM --> CW_METRICS & CW_LOGS

    XRAY --> CW_TRACES
    CW_TRACES --> GRAFANA
    CW_METRICS --> CW_ALARMS
    CW_ALARMS --> SNS --> EVENTBRIDGE
    CW_METRICS & CW_LOGS & CW_TRACES --> GRAFANA
    CW_METRICS & CW_LOGS --> CW_DASH

    CLOUDTRAIL & CONFIG --> SECURITYHUB
    GUARDDUTY --> SECURITYHUB
```

### 1.2 ADOT Instrumentation Detail (EKS)

```mermaid
graph LR
    subgraph EKS_NODE["EKS Worker Node"]
        subgraph APP_POD["Application Pod"]
            APP["App Container"]
            SIDECAR["ADOT Sidecar\n(or SDK auto-instrumentation)"]
        end
        DAEMON["ADOT DaemonSet Collector\n(Node-level aggregator)"]
        CW_AGENT_N["CloudWatch Agent\n(Node metrics)"]
    end

    APP -->|"OTLP gRPC :4317"| SIDECAR
    SIDECAR -->|"OTLP gRPC"| DAEMON
    DAEMON -->|"Traces → OTLP"| XRAY_EP["X-Ray Endpoint\n(xray.region.amazonaws.com)"]
    DAEMON -->|"Metrics → EMF"| CW_EP["CloudWatch EMF\n(Embedded Metric Format)"]
    CW_AGENT_N -->|"Node/Container Metrics"| CW_CONT["Container Insights"]
```

---

## 2. Component Interaction Flow

### 2.1 Request Lifecycle with Observability Hooks

```mermaid
sequenceDiagram
    actor User
    participant RUM as CloudWatch RUM
    participant CF as CloudFront
    participant React as React App
    participant APIGW as API Gateway
    participant ALB as App Load Balancer
    participant OrderSvc as Order Service (EKS)
    participant ADOT as ADOT Collector
    participant Lambda as Lambda Processor
    participant RDS as Aurora RDS
    participant DDB as DynamoDB
    participant XRay as AWS X-Ray
    participant CW as CloudWatch

    User->>CF: HTTPS Request
    CF->>React: Serve React Bundle
    React-->>RUM: Page Load / Web Vitals beacon<br/>(LCP, FID, CLS, TTFB)

    User->>APIGW: POST /api/orders
    Note over APIGW: Access log → CW Logs<br/>X-Ray segment starts
    APIGW->>ALB: Forward with X-Amzn-Trace-Id header
    ALB->>OrderSvc: Route to EKS Pod
    Note over OrderSvc: ADOT SDK injects<br/>trace context (W3C TraceContext)

    OrderSvc->>ADOT: Export span (OTLP gRPC)
    ADOT->>XRay: Forward trace segments
    ADOT->>CW: EMF metrics (latency, errors)

    OrderSvc->>RDS: INSERT order record
    Note over RDS: Enhanced Monitoring<br/>Performance Insights active
    RDS-->>OrderSvc: ACK

    OrderSvc->>DDB: PUT session/cart item
    Note over DDB: CloudWatch contributor<br/>insights enabled

    OrderSvc->>Lambda: Invoke async (SQS trigger)
    Note over Lambda: Lambda Powertools<br/>auto-instrumentation

    Lambda->>XRay: Sub-segment
    Lambda->>CW: Custom metric (order_processed_total)

    OrderSvc-->>APIGW: 201 Created
    APIGW-->>User: Response

    XRay-->>CW: Service Map update
    CW-->>CW: Composite Alarm evaluation
```

### 2.2 Observability Signal Categories per Component

| Component | Metrics | Logs | Traces | RUM |
|---|---|---|---|---|
| React Frontend | Web Vitals, JS Errors, API Latency | Console errors, JS stack traces | — | ✅ CloudWatch RUM |
| API Gateway | Latency, 4xx/5xx, Request Count | Access Logs → CW Logs | ✅ X-Ray active tracing | — |
| ALB | ActiveConnections, TargetResponseTime, HTTP_Fixed_Response | Access Logs → S3 → CW Logs Insights | ✅ X-Ray pass-through | — |
| EKS Pods | CPU, Memory, Disk I/O, Custom OTLP metrics | stdout/stderr → Fluent Bit → CW Logs | ✅ ADOT → X-Ray | — |
| Lambda | Duration, ConcurrentExecutions, Errors, Throttles | Function logs → CW Logs | ✅ Active tracing | — |
| RDS Aurora | CPUUtilization, FreeableMemory, DBConnections | Slow query, error logs | ✅ RDS Performance Insights | — |
| DynamoDB | ConsumedReadCapacityUnits, SystemErrors, SuccessfulRequestLatency | CloudTrail data events | — | — |

---

## 3. Monitoring Data Flow

### 3.1 Metrics Pipeline

```mermaid
flowchart LR
    subgraph SOURCES["Metric Sources"]
        AWS_NATIVE["AWS Native Metrics\n(RDS, DDB, Lambda, ALB, APIGW)"]
        CONTAINER["Container Insights\n(EKS Pod/Node/Namespace)"]
        APP_SIG["Application Signals\n(SLI: Latency, Availability, Errors)"]
        CUSTOM_EMF["Custom EMF Metrics\n(Business KPIs via ADOT)"]
        RUM_M["RUM Metrics\n(Web Vitals, Page Views)"]
    end

    subgraph CW_NS["CloudWatch Namespaces"]
        NS1["AWS/ApplicationELB"]
        NS2["AWS/ApiGateway"]
        NS3["AWS/EKS\nContainerInsights"]
        NS4["AWS/Lambda"]
        NS5["AWS/RDS"]
        NS6["AWS/DynamoDB"]
        NS7["Custom/ECommerce/*"]
        NS8["AWS/ApplicationSignals"]
    end

    subgraph PROCESSING["Processing Layer"]
        METRIC_MATH["Metric Math\n(Error Rate = Errors/Requests)"]
        ANOMALY["Anomaly Detection\n(ML-based bands)"]
        COMPOSITE["Composite Alarms\n(multi-condition)"]
    end

    subgraph DESTINATIONS["Destinations"]
        CW_DASH2["CloudWatch Dashboards\n(Ops team)"]
        GRAFANA2["Managed Grafana\n(Engineering)"]
        SNS2["SNS → PagerDuty / Slack"]
        S3_ARCHIVE["S3 Archive\n(long-term retention)"]
    end

    SOURCES --> CW_NS
    CW_NS --> PROCESSING
    PROCESSING --> DESTINATIONS
```

### 3.2 SLO / SLI Definition via Application Signals

| Service | SLI | SLO Target | Error Budget (30d) |
|---|---|---|---|
| Order Service | p99 Latency < 500ms | 99.5% | 3.6 hours |
| Payment Service | Availability (success rate) | 99.9% | 43.2 minutes |
| Product Service | p95 Latency < 200ms | 99.0% | 7.2 hours |
| Cart Service (DDB) | Read Latency p99 < 10ms | 99.5% | 3.6 hours |
| React Frontend | LCP < 2.5s | 95.0% | 36 hours |

### 3.3 Key CloudWatch Alarms

```mermaid
graph TD
    subgraph COMPOSITE_ALARM["Composite Alarm: Payment Service Degraded"]
        CA["🔴 PaymentService-Degraded\n(OR logic)"]
        A1["Alarm: p99Latency > 1s\n(5min evaluation)"]
        A2["Alarm: ErrorRate > 1%\n(3 datapoints of 5)"]
        A3["Alarm: ThrottleCount > 10\n(1min evaluation)"]
        CA --> A1 & A2 & A3
    end

    subgraph ACTION["Alarm Actions"]
        SNS_CRITICAL["SNS: ecommerce-critical"]
        PAGE["PagerDuty Escalation"]
        RB["EventBridge → Lambda\n(Auto-rollback trigger)"]
        DASH_ANN["CloudWatch Dashboard\nAnnotation"]
    end

    CA -->|"ALARM state"| SNS_CRITICAL
    SNS_CRITICAL --> PAGE & RB & DASH_ANN
```

---

## 4. Logging Architecture

### 4.1 Log Pipeline

```mermaid
flowchart TB
    subgraph LOG_SOURCES["Log Sources"]
        APP_LOGS["Application Logs\n(structured JSON)\nEKS stdout/stderr"]
        APIGW_LOGS["API Gateway Access Logs\n(JSON format)"]
        ALB_LOGS["ALB Access Logs\n(S3 delivery)"]
        LAMBDA_LOGS["Lambda Logs\n(Powertools structured)"]
        RDS_LOGS["RDS Logs\n(slow query, error, audit)"]
        VPC_FLOW["VPC Flow Logs\n(network telemetry)"]
        CLOUDTRAIL_L["CloudTrail Logs\n(API activity)"]
    end

    subgraph COLLECTION["Log Collection"]
        FLUENTBIT["Fluent Bit DaemonSet\n(EKS — lightweight forwarder)"]
        CW_AGENT_L["CloudWatch Agent\n(EC2 / EKS nodes)"]
        LAMBDA_EXTN["Lambda Telemetry API\n(Extensions)"]
    end

    subgraph CW_LOGS_STORE["CloudWatch Logs"]
        LG_APP["/aws/eks/ecommerce/application\n(7-day hot retention)"]
        LG_APIGW["/aws/apigateway/ecommerce\n(30-day retention)"]
        LG_LAMBDA["/aws/lambda/*\n(14-day retention)"]
        LG_RDS["/aws/rds/cluster/ecommerce/*\n(30-day retention)"]
        LG_VPC["/aws/vpc/flowlogs\n(7-day retention)"]
        LG_AUDIT["/aws/cloudtrail/ecommerce\n(90-day retention)"]
    end

    subgraph LOG_PROCESSING["Log Processing"]
        CW_LI["CloudWatch Logs Insights\n(ad-hoc queries)"]
        METRIC_FILTER["Metric Filters\n(error count, latency extraction)"]
        LOG_ANOMALY["CloudWatch Log Anomaly Detection\n(ML-based pattern alerting)"]
        SUBSCR["Subscription Filter\n(real-time streaming)"]
    end

    subgraph LONG_TERM["Long-Term & SIEM"]
        S3_LOGS["S3 Glacier\n(1-year+ archive)"]
        OPENSEARCH["Amazon OpenSearch\n(optional SIEM / full-text)"]
    end

    APP_LOGS --> FLUENTBIT
    APIGW_LOGS & ALB_LOGS --> CW_AGENT_L
    LAMBDA_LOGS --> LAMBDA_EXTN
    RDS_LOGS --> CW_LOGS_STORE
    VPC_FLOW --> LG_VPC
    CLOUDTRAIL_L --> LG_AUDIT

    FLUENTBIT --> LG_APP
    CW_AGENT_L --> LG_APIGW
    LAMBDA_EXTN --> LG_LAMBDA

    LG_APP & LG_APIGW & LG_LAMBDA & LG_RDS --> LOG_PROCESSING
    LOG_PROCESSING --> S3_LOGS
    SUBSCR --> OPENSEARCH
```

### 4.2 Structured Log Format (JSON Standard)

All application services **must** emit structured logs in this schema:

```json
{
  "timestamp": "2026-07-18T10:23:45.123Z",
  "level": "INFO",
  "service": "order-service",
  "version": "2.3.1",
  "environment": "production",
  "region": "us-east-1",
  "traceId": "1-abc12345-xyz98765432109876543210",
  "spanId": "abc1234567890abc",
  "userId": "usr_hashed_9f3a",
  "orderId": "ord_20260718_0042",
  "message": "Order created successfully",
  "durationMs": 142,
  "httpStatusCode": 201,
  "component": "OrderController",
  "action": "createOrder"
}
```

**Key rules:**
- No PII in log fields — use hashed/tokenized identifiers
- Always include `traceId` / `spanId` for log-trace correlation
- Use `ERROR` level only for actionable failures (triggers metric filters)
- `durationMs` enables latency-from-logs extraction via Metric Filters

### 4.3 Fluent Bit Configuration (EKS DaemonSet)

```yaml
# fluent-bit-config.yaml (key sections)
[INPUT]
    Name              tail
    Path              /var/log/containers/*ecommerce*.log
    Parser            docker
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Merge_Log           On          # Merge JSON app logs into record
    Keep_Log            Off
    Annotations         Off
    Labels              On

[FILTER]
    Name    record_modifier
    Match   *
    Record  environment production
    Record  region      us-east-1

[OUTPUT]
    Name               cloudwatch_logs
    Match              kube.*
    region             us-east-1
    log_group_name     /aws/eks/ecommerce/application
    log_stream_prefix  pod-
    auto_create_group  true
    log_retention_days 7
```

---

## 5. Tracing Architecture

### 5.1 Distributed Trace Flow

```mermaid
flowchart LR
    subgraph TRACE_INIT["Trace Initiation"]
        USER_REQ["User Request"]
        APIGW_T["API Gateway\n(Root Segment)"]
    end

    subgraph PROPAGATION["Context Propagation (W3C TraceContext)"]
        ALB_T["ALB\n(Segment)"]
        ORDER_T["Order Service\n(Subsegment)"]
        PAYMENT_T["Payment Service\n(Subsegment)"]
        LAMBDA_T["Lambda\n(Subsegment)"]
    end

    subgraph DATA_T["Data Access Traces"]
        RDS_T["RDS\n(SQL subsegment\nvia X-Ray SDK)"]
        DDB_T["DynamoDB\n(HTTP subsegment\nauto-instrumented)"]
        HTTP_T["External HTTP\n(3rd party payment API)"]
    end

    subgraph XRAY_BACKEND["X-Ray Backend"]
        XRAY_SVC_MAP["Service Map\n(latency heatmap)"]
        XRAY_TRACE_LIST["Trace List\n(sampled traces)"]
        XRAY_ANALYTICS["Trace Analytics\n(filter expressions)"]
    end

    USER_REQ --> APIGW_T
    APIGW_T -->|"X-Amzn-Trace-Id header"| ALB_T
    ALB_T -->|"Trace context forwarded"| ORDER_T
    ORDER_T -->|"gRPC context"| PAYMENT_T
    ORDER_T -->|"Lambda invoke"| LAMBDA_T
    ORDER_T --> RDS_T & DDB_T
    PAYMENT_T --> HTTP_T

    ORDER_T & PAYMENT_T & LAMBDA_T -->|"OTLP → ADOT"| XRAY_BACKEND
    XRAY_BACKEND --> XRAY_SVC_MAP & XRAY_TRACE_LIST & XRAY_ANALYTICS
```

### 5.2 ADOT Collector Pipeline Configuration

```yaml
# adot-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 50
  
  memory_limiter:
    limit_mib: 512
    spike_limit_mib: 128
    check_interval: 5s

  resourcedetection:
    detectors: [env, eks, ec2]
    timeout: 5s

  # Redact sensitive attributes before export
  attributes:
    actions:
      - key: user.email
        action: delete
      - key: http.request.header.authorization
        action: delete
      - key: db.statement
        action: hash   # Hash SQL containing PII

  filter/drop_health_checks:
    spans:
      exclude:
        match_type: strict
        attributes:
          - key: http.url
            value: "/health"

exporters:
  awsxray:
    region: us-east-1
    index_all_attributes: true

  awsemf:
    region: us-east-1
    namespace: Custom/ECommerce
    log_group_name: /aws/otel/ecommerce/metrics
    dimension_rollup_option: NoDimensionRollup
    metric_declarations:
      - dimensions: [[service.name, environment]]
        metric_name_selectors:
          - "http.server.duration"
          - "http.server.request.count"
          - "order.processing.duration"

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, attributes, filter/drop_health_checks, batch]
      exporters:  [awsxray]
    
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters:  [awsemf]
```

### 5.3 Sampling Strategy

| Scenario | Sampling Rule | Rationale |
|---|---|---|
| Default (all services) | 5% reservoir + 5/s fixed | Baseline coverage |
| Payment Service | 100% (no sampling) | Compliance — all transactions traced |
| Error responses (5xx) | 100% forced | Every failure must be traceable |
| Health check endpoints | 0% (filtered in ADOT) | Noise reduction |
| Latency > 1s | 100% forced (X-Ray rule) | Capture all slow requests |
| Admin/internal APIs | 10% | Lower traffic value |

```json
// X-Ray Sampling Rules (via AWS Console / Terraform)
{
  "rules": [
    {
      "RuleName": "PaymentService-AllTraces",
      "Priority": 1,
      "ServiceName": "payment-service",
      "FixedRate": 1.0,
      "ReservoirSize": 100,
      "ResourceARN": "*",
      "HTTPMethod": "POST",
      "URLPath": "/api/payments/*"
    },
    {
      "RuleName": "SlowRequests",
      "Priority": 5,
      "FixedRate": 1.0,
      "ReservoirSize": 50,
      "Attributes": { "http.status_code": "5??" }
    },
    {
      "RuleName": "DefaultRule",
      "Priority": 10000,
      "FixedRate": 0.05,
      "ReservoirSize": 5
    }
  ]
}
```

---

## 6. Security Architecture

### 6.1 Observability Security Model

```mermaid
graph TB
    subgraph IAM["IAM Security Boundary"]
        IRSA_ORDER["IRSA: order-service-sa\nAllow: xray:PutTraceSegments\nAllow: cloudwatch:PutMetricData\nAllow: logs:CreateLogStream"]
        IRSA_ADOT["IRSA: adot-collector-sa\nAllow: xray:PutTraceSegments\nAllow: cloudwatch:PutMetricData\nAllow: logs:PutLogEvents\nAllow: xray:GetSamplingRules"]
        IRSA_LAMBDA["Lambda Execution Role\nAllow: xray:PutTraceSegments\nAllow: logs:CreateLogGroup\nAllow: cloudwatch:PutMetricData"]
    end

    subgraph ENCRYPTION["Encryption"]
        KMS_CW["KMS CMK: cloudwatch-logs-key\n(log group encryption)"]
        KMS_XRAY["KMS CMK: xray-traces-key\n(trace encryption at rest)"]
        TLS["TLS 1.3 in transit\n(OTLP gRPC, HTTPS)"]
    end

    subgraph NETWORK["Network Controls"]
        VPC_EP_CW["VPC Endpoint: com.amazonaws.us-east-1.logs\n(PrivateLink — no internet)"]
        VPC_EP_XRAY["VPC Endpoint: com.amazonaws.us-east-1.xray\n(PrivateLink)"]
        VPC_EP_MON["VPC Endpoint: com.amazonaws.us-east-1.monitoring\n(PrivateLink)"]
        SG_ADOT["Security Group: adot-collector\nInbound: 4317 from EKS nodes only\nOutbound: 443 to VPC endpoints only"]
    end

    subgraph AUDIT["Audit & Compliance"]
        CT_OBS["CloudTrail: Log all CloudWatch / X-Ray API calls"]
        CONFIG_RULES["AWS Config Rules:\n- cloudwatch-alarm-action-check\n- cloud-trail-encryption-enabled\n- xray-encryption-at-rest"]
        SEC_HUB["Security Hub: FSBP Standard\n(AWS Foundational Security Best Practices)"]
    end

    subgraph DATA_PROT["Data Protection"]
        REDACT["ADOT Processor: Redact PII attributes\n(email, CC, authorization headers)"]
        HASH["SQL statement hashing\n(db.statement → SHA-256 prefix)"]
        MASK["CloudWatch Logs: Data Protection Policy\n(SSN, credit card patterns auto-masked)"]
    end

    IAM --> ENCRYPTION
    NETWORK --> ENCRYPTION
    AUDIT --> SEC_HUB
    DATA_PROT --> AUDIT
```

### 6.2 IAM Least-Privilege Policy (Application Pod)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "XRayWrite",
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "xray:GetSamplingRules",
        "xray:GetSamplingTargets"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchMetrics",
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "cloudwatch:namespace": [
            "Custom/ECommerce/OrderService",
            "Custom/ECommerce/PaymentService"
          ]
        }
      }
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/eks/ecommerce/*:*"
    }
  ]
}
```

### 6.3 CloudWatch Logs Data Protection Policy

```json
{
  "Name": "ecommerce-pii-protection",
  "DataIdentifiers": [
    "arn:aws:dataprotection::aws:data-identifier/CreditCardNumber",
    "arn:aws:dataprotection::aws:data-identifier/EmailAddress",
    "arn:aws:dataprotection::aws:data-identifier/USSocialSecurityNumber",
    "arn:aws:dataprotection::aws:data-identifier/IpAddress"
  ],
  "Configuration": {
    "CustomDataIdentifiers": [],
    "ManagedDataIdentifiers": ["CreditCardNumber", "EmailAddress", "USSocialSecurityNumber"]
  },
  "Statement": [
    {
      "Sid": "audit-policy",
      "DataIdentifier": ["CreditCardNumber", "EmailAddress", "USSocialSecurityNumber"],
      "Operation": {
        "Audit": {
          "FindingsDestination": {
            "S3": { "Bucket": "ecommerce-compliance-findings" }
          }
        }
      }
    },
    {
      "Sid": "redact-policy",
      "DataIdentifier": ["CreditCardNumber", "USSocialSecurityNumber"],
      "Operation": { "Deidentify": { "MaskConfig": {} } }
    }
  ]
}
```

---

## 7. Multi-AZ Design

### 7.1 Multi-AZ Observability Topology

```mermaid
graph TB
    subgraph REGION["AWS Region: us-east-1"]
        subgraph AZ_A["Availability Zone: us-east-1a"]
            EKS_A["EKS Node Group A\n(2x m6i.xlarge)"]
            ADOT_A["ADOT DaemonSet\n(node a)"]
            RDS_PRIMARY["RDS Aurora Writer\n(Primary)"]
        end

        subgraph AZ_B["Availability Zone: us-east-1b"]
            EKS_B["EKS Node Group B\n(2x m6i.xlarge)"]
            ADOT_B["ADOT DaemonSet\n(node b)"]
            RDS_REPLICA_B["RDS Aurora Reader\n(Replica B)"]
        end

        subgraph AZ_C["Availability Zone: us-east-1c"]
            EKS_C["EKS Node Group C\n(2x m6i.xlarge)"]
            ADOT_C["ADOT DaemonSet\n(node c)"]
            RDS_REPLICA_C["RDS Aurora Reader\n(Replica C)"]
        end

        subgraph REGIONAL_SERVICES["Regional Observability Services (inherently HA)"]
            CW_REGIONAL["CloudWatch\n(Regional — 99.9% SLA)"]
            XRAY_REGIONAL["AWS X-Ray\n(Regional — 99.9% SLA)"]
            GRAFANA_REGIONAL["Managed Grafana\n(Multi-AZ workspace)"]
        end

        subgraph SHARED["Shared Infrastructure"]
            NLB_ADOT["NLB → ADOT Gateway\n(cross-AZ routing failover)"]
            DDB_GLOBAL["DynamoDB\n(inherently Multi-AZ)"]
        end
    end

    ADOT_A & ADOT_B & ADOT_C -->|"VPC PrivateLink"| CW_REGIONAL
    ADOT_A & ADOT_B & ADOT_C -->|"VPC PrivateLink"| XRAY_REGIONAL
    CW_REGIONAL --> GRAFANA_REGIONAL
    EKS_A --> RDS_PRIMARY
    EKS_B & EKS_C --> RDS_REPLICA_B & RDS_REPLICA_C
    EKS_A & EKS_B & EKS_C --> DDB_GLOBAL
```

### 7.2 ADOT Collector High Availability Configuration

```yaml
# adot-daemonset.yaml — ensures one collector per node
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: adot-collector
  namespace: amazon-metrics
spec:
  selector:
    matchLabels:
      app: adot-collector
  template:
    metadata:
      labels:
        app: adot-collector
    spec:
      serviceAccountName: adot-collector-sa
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: adot-collector
          image: public.ecr.aws/aws-observability/aws-otel-collector:latest
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 10
```

---

## 8. High Availability Considerations

### 8.1 Observability Resilience Design

```mermaid
graph LR
    subgraph RESILIENCE["HA Resilience Patterns"]
        direction TB

        subgraph BUFFER["Buffering & Backpressure"]
            B1["ADOT memory_limiter\n(512MB cap — prevents OOM)"]
            B2["Fluent Bit mem_buf_limit\n(50MB per tail input)"]
            B3["CloudWatch PutLogEvents\nbatch (max 1MB / 10k events)"]
        end

        subgraph RETRY["Retry & Circuit Breaking"]
            R1["ADOT exporter retry\n(max_elapsed_time: 300s)"]
            R2["Exponential backoff\n(initial: 5s, multiplier: 1.5)"]
            R3["Dead-letter: SQS DLQ\nfor failed metric deliveries"]
        end

        subgraph FALLBACK["Graceful Degradation"]
            F1["App continues if\nADOT unavailable\n(non-blocking SDK)"]
            F2["Local disk buffer\n(Fluent Bit tail position)"]
            F3["CloudWatch EMF fallback\n(if X-Ray endpoint fails)"]
        end

        subgraph CAPACITY["Capacity Planning"]
            C1["CloudWatch Logs:\n5GB/s ingestion limit\n→ partition by service"]
            C2["X-Ray: 100k segments/s\nper region (soft limit)"]
            C3["Managed Grafana:\nAuto-scaling workspace"]
        end
    end
```

### 8.2 Observability Component SLA Matrix

| Component | AWS SLA | Recovery Approach | Data Loss Tolerance |
|---|---|---|---|
| CloudWatch Logs | 99.9% | Regional service — auto-HA | None (persistent) |
| CloudWatch Metrics | 99.9% | Regional service — auto-HA | 1-minute granularity gap |
| AWS X-Ray | 99.9% | Regional service — auto-HA | Sampled traces (acceptable) |
| ADOT DaemonSet | Per-node | DaemonSet restart policy + liveness | < 60s buffer during restart |
| Managed Grafana | 99.9% | Multi-AZ workspace | Dashboards are stateless config |
| Fluent Bit | Per-pod | DaemonSet + tail position file | Logs from crash window |
| RUM | CloudFront-backed | Global CDN delivery | Beacon loss < 1% |

### 8.3 Chaos Engineering Scenarios

```mermaid
mindmap
  root((Chaos Scenarios))
    AZ Failure
      EKS nodes evicted from AZ-B
      ADOT DaemonSet recreated in surviving AZs
      CloudWatch continues receiving from AZ-A, AZ-C
      Alerts: NodeNotReady, PodEviction
    ADOT Collector Crash
      Apps buffer in-process with OTLP retry
      Fluent Bit continues log delivery independently
      X-Ray gap appears in service map — alarm fires
    CloudWatch Throttle
      Metric filter for ThrottleCount > 100
      ADOT exporter backs off exponentially
      EMF log-based fallback activated
    RDS Failover (60-90s)
      Performance Insights shows connection spike
      Enhanced Monitoring shows replica promotion
      Application Signals: Availability SLO breach auto-detected
```

---

## 9. AWS Well-Architected Best Practices

### 9.1 Operational Excellence Pillar

```mermaid
graph LR
    subgraph OE["Operational Excellence"]
        OE1["Runbooks as Code\n(SSM Automation Documents\nfor incident response)"]
        OE2["GitOps for Dashboards\n(Grafana dashboards\nin Git — Terraform)"]
        OE3["Alarm Runbook Links\n(Every alarm → wiki URL\nin AlarmDescription)"]
        OE4["Game Days\n(Quarterly AZ failover tests\nusing AWS FIS)"]
        OE5["SLO Review\n(Weekly error budget\nburn rate review)"]
    end
```

| Practice | Implementation |
|---|---|
| Define operations as code | ADOT config, Grafana dashboards, alarms via Terraform IaC |
| Annotate deployments | EventBridge → CloudWatch Annotations on every EKS deploy |
| Incremental change | Blue/green EKS deployments with automatic SLO rollback |
| Anticipate failure | Composite alarms + runbook automation (SSM OpsItems) |
| Learn from all operational events | Post-incident reviews → CloudWatch Insights queries saved |

### 9.2 Reliability Pillar

| Practice | Implementation |
|---|---|
| Monitor all layers | React (RUM) → API GW → ALB → EKS → Lambda → RDS — full coverage |
| Multi-AZ collector deployment | ADOT DaemonSet across all 3 AZs |
| Dead-letter queue for alerts | SNS DLQ for undelivered alarm notifications |
| Health checks | EKS liveness/readiness probes → ADOT health endpoint monitored |
| Tested disaster recovery | Quarterly GameDays with AWS FIS — inject ADOT pod failures |

### 9.3 Security Pillar

| Practice | Implementation |
|---|---|
| Least-privilege IAM | IRSA per service — scoped to specific CW namespaces |
| No credentials in code | All SDK auth via IRSA (EKS) / Lambda execution role |
| Encrypt data at rest | KMS CMKs for CloudWatch Logs + X-Ray traces |
| Encrypt data in transit | TLS 1.3 for all OTLP traffic + HTTPS for CloudWatch APIs |
| PII protection | CW Logs Data Protection Policy + ADOT attribute redaction |
| Network isolation | VPC PrivateLink endpoints — observability data never traverses internet |
| Audit all API calls | CloudTrail + Security Hub FSBP standard |

### 9.4 Performance Efficiency Pillar

| Practice | Implementation |
|---|---|
| Right-size collectors | ADOT: 100m CPU / 256Mi memory requests — tuned per node capacity |
| Efficient log filtering | Fluent Bit `grep` filter — drop debug logs in production |
| Sampling at the edge | X-Ray rules: 5% default, 100% for payments/errors |
| Metric cardinality control | ADOT dimension_rollup — avoid high-cardinality label explosions |
| CloudWatch Logs Insights | Use query caching + scoped log group queries |
| EMF over API calls | CloudWatch Embedded Metric Format — batch metrics via logs |

### 9.5 Cost Optimization Pillar

```mermaid
graph TB
    subgraph COST["Cost Optimization Strategy"]
        C1["Log Tiering\n7-day hot (CW Logs)\n90-day warm (S3 IA)\n365-day cold (S3 Glacier)"]
        C2["Metric Resolution\n60s standard for ops metrics\n1s high-res only for payments SLO"]
        C3["X-Ray Sampling\n5% default sampling\n= ~95% cost reduction vs 100%"]
        C4["CloudWatch Alarms\nComposite alarms\n= fewer individual alarm evaluations"]
        C5["Grafana Data Sources\nQuery CW directly\n= no duplicate storage"]
        C6["Container Insights\nEnhanced mode only on\ncritical namespaces"]
    end
```

| Cost Lever | Estimated Saving |
|---|---|
| Log retention lifecycle (S3 tiering) | 60–75% vs indefinite CW retention |
| X-Ray 5% sampling (vs 100%) | ~95% trace cost reduction |
| Standard vs high-res metrics (60s default) | 70% metric cost reduction |
| Fluent Bit log filtering (drop DEBUG) | 30–50% log volume reduction |
| EMF batched metric delivery | Eliminates separate PutMetricData API calls |

### 9.6 Sustainability Pillar

| Practice | Implementation |
|---|---|
| Right-size ADOT collectors | DaemonSet resource limits prevent idle CPU waste |
| Aggregate metrics over time | Use CloudWatch metric math instead of raw high-frequency data |
| Delete unused dashboards | Monthly Grafana dashboard audit — auto-delete stale (90d no view) |
| Compress log data | Fluent Bit gzip compression before S3 archive |
| Spot instances for non-critical nodes | EKS node group with Spot for ADOT/Grafana workloads |

---

## Architecture Decision Records (ADRs)

### ADR-001: ADOT DaemonSet vs Sidecar
- **Decision**: DaemonSet collector per node
- **Rationale**: Lower resource overhead (1 collector per node vs 1 per pod). Sidecar used only for services requiring isolated sampling (Payment Service)
- **Trade-off**: Shared collector = single point of failure per node (mitigated by DaemonSet restart policy)

### ADR-002: Fluent Bit over CloudWatch Agent for Log Collection
- **Decision**: Fluent Bit DaemonSet as primary log forwarder
- **Rationale**: 10x lower memory footprint than CW Agent. Native Kubernetes metadata enrichment. CloudWatch Agent retained for node-level host metrics
- **Trade-off**: Two agents per node — mitigated by strict resource limits

### ADR-003: Application Signals for SLO Management
- **Decision**: CloudWatch Application Signals as SLO engine (not Prometheus Rules)
- **Rationale**: Native AWS integration, no additional infrastructure. Auto-discovers EKS services via ADOT. Integrated error budget burn alerting
- **Trade-off**: AWS-proprietary SLO format — lower portability than OpenSLO

### ADR-004: Managed Grafana over Self-Hosted
- **Decision**: Amazon Managed Grafana
- **Rationale**: IAM-native authentication (no Grafana user management), Multi-AZ by default, CloudWatch datasource pre-configured, SSO via IAM Identity Center
- **Trade-off**: Higher cost than self-hosted — justified by zero-ops overhead

---

## Operational Runbook: Critical Alert Response

```mermaid
flowchart TD
    A["🔴 Composite Alarm FIRES\n(Payment Service Degraded)"] --> B["SNS → PagerDuty\nOn-call engineer paged"]
    B --> C["Open X-Ray Service Map\n(identify degraded node)"]
    C --> D{Latency or Error?}
    D -->|"High Latency"| E["Check RDS Performance Insights\n(slow queries, connections)"]
    D -->|"High Errors"| F["CloudWatch Logs Insights:\nfilter @logStream like /payment/\n| filter level = 'ERROR'\n| stats count by errorCode"]
    E --> G["Scale RDS Read Replica\nor tune query (index)"]
    F --> H["Check recent deployment\n(EventBridge annotation on dashboard)"]
    H --> I{Recent deploy?}
    I -->|"Yes"| J["Trigger rollback:\naws eks update-nodegroup\n(SSM Automation)"]
    I -->|"No"| K["Escalate to Tier 2\n+ open AWS Support case"]
    G & J & K --> L["Monitor Application Signals SLO\nConfirm error budget recovery"]
    L --> M["Post-Incident Review\n(within 48h)"]
```

---

*Architecture designed following AWS Well-Architected Framework 2026 edition. All resource limits and SLAs subject to AWS regional service quotas.*
