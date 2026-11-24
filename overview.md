# Credit Score Service: overview

```mermaid
flowchart LR
    CONS["Consumers"]

    subgraph EXT["External providers"]
        S1["<b>Data Source 1</b><br/>SOAP · monthly"]
        S2["<b>Data Source 2</b><br/>REST · weekly"]
    end

    subgraph AWS["AWS"]
        S3[("<b>Data Source 3</b><br/>MySQL · real-time")]
        API["<b>Credit Score API</b><br/>GET /companies/{vat}/<br/>credit-score"]
        R[("<b>Redis</b>")]
        PG[("<b>PostgreSQL</b>")]
        DEF["<b>Default score</b>"]
        SCH["<b>Scheduler</b>"]
        subgraph RQ["SQS refresh queues"]
            Q1[["ds1"]]
            Q2[["ds2"]]
            Q3[["ds3"]]
        end
        subgraph CON["Connectors"]
            C1["<b>DS1 connector</b><br/>lookup by VAT"]
            C2["<b>DS2 connector</b><br/>search → id → data"]
            C3["<b>DS3 connector</b><br/>polls changed rows"]
        end
        QS[["<b>SQS</b><br/>recalculate"]]
        SC["<b>Scoring Service</b>"]
    end

    CONS <-->|"HTTPS<br/>VAT → score"| API
    API -->|"1 · read"| R
    API -->|"2 · on miss"| PG
    API -->|"3 · nothing found"| DEF
    API -.->|"unknown or stale:<br/>request refresh"| RQ
    SCH -->|"due VATs"| RQ
    SCH -->|"due recalculations"| QS
    Q1 --> C1
    Q2 --> C2
    Q3 --> C3
    S1 -->|"SOAP / HTTPS"| C1
    S2 -->|"REST / HTTPS"| C2
    S3 -->|"SQL read-only"| C3
    C1 & C3 -.->|"name + address"| PG
    PG -.->|"name + address"| C2
    CON -->|"source data"| PG
    CON -->|"source updated"| QS
    QS --> SC
    SC -->|"write score"| PG
    SC -->|"write score"| R

    classDef req fill:#dae8fc,stroke:#6c8ebf,color:#000
    classDef bg fill:#f5f5f5,stroke:#999,color:#000
    classDef src fill:#ffe6cc,stroke:#d79b00,color:#000
    class CONS,API,R,PG,DEF req
    class SCH,Q1,Q2,Q3,QS,SC,C1,C2,C3 bg
    class S1,S2,S3 src
```

## GET /companies/{vat}/credit-score

### Known VAT

```json
{
  "vatNumber": "DE123456789",
  "score": 72,
  "band": "B",
  "status": "PARTIAL",
  "confidence": 0.92,
  "calculatedAt": "2026-11-24T09:15:02Z",
  "sources": [
    { "source": "ds1_rating_soap", "state": "OK",    "dataAsOf": "2026-11-03T00:00:00Z" },
    { "source": "ds2_credit_rest", "state": "STALE", "dataAsOf": "2026-11-10T00:00:00Z" },
    { "source": "ds3_internal",    "state": "OK",    "dataAsOf": "2026-11-24T09:14:40Z" }
  ]
}
```

### Unknown VAT

```json
{
  "vatNumber": "DE999999999",
  "score": 30,
  "band": "D",
  "status": "PROVISIONAL",
  "confidence": 0,
  "calculatedAt": null,
  "sources": []
}
```

## Processes

### Read a score

```mermaid
%%{init: {"sequence": {"mirrorActors": false, "actorMargin": 90}}}%%
sequenceDiagram
    participant C as Consumer
    participant A as API
    participant R as Redis
    participant P as PostgreSQL
    participant Q as SQS
    C->>A: GET /companies/{vat}/credit-score
    A->>R: get score
    alt hit
        R-->>A: score
    else miss
        A->>P: get score
        P-->>A: score or<br/>nothing
        A->>R: set score if found
        Note right of A: nothing found → default score
    end
    opt unknown or stale
        A->>Q: request refresh
    end
    A-->>C: 200 score
```

### Refresh

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant Q as SQS source queue
    participant K as Connector
    participant D as Data source
    participant P as PostgreSQL
    participant QS as SQS recalculate
    participant SC as Scoring Service
    participant R as Redis
    S->>Q: due VATs
    S->>QS: due recalculations
    Q->>K: refresh VAT
    K->>P: get name + address (DS2)
    K->>D: fetch data
    D-->>K: data
    K->>P: save source data
    K->>QS: source updated
    QS->>SC: recalculate VAT
    SC->>P: read source data
    SC->>P: save score
    SC->>R: set score
```

## Infrastructure

```mermaid
flowchart LR
    CONS["Consumers"]
    SEN["Sentry"]

    subgraph AWS["AWS"]
        GW["API Gateway"]
        SM["Secrets Manager"]
        EB["EventBridge Scheduler"]
        SQS[["SQS queues + DLQs"]]
        subgraph VPC["VPC · private subnets"]
            subgraph ECS["ECS Fargate"]
                API["Credit Score API"]
                CON["DS1 · DS2 · DS3 connectors"]
                SC["Scoring Service"]
                JOB["Scheduler job"]
            end
            R[("ElastiCache Redis")]
            PG[("RDS PostgreSQL")]
            DS3[("Data Source 3<br/>MySQL")]
            NAT["NAT Gateway"]
        end
    end

    subgraph EXT["External providers"]
        DS1["Data Source 1<br/>SOAP"]
        DS2["Data Source 2<br/>REST"]
    end

    CONS -->|"HTTPS · REST"| GW
    GW -->|"VPC Link"| API
    API --> R
    API --> PG
    API -->|"refresh request"| SQS
    EB -->|"tick"| JOB
    JOB -->|"due VATs"| SQS
    SQS --> CON
    SQS --> SC
    CON -->|"source updated"| SQS
    CON --> PG
    SC --> PG
    SC --> R
    CON -->|"SQL · read-only"| DS3
    CON --> NAT
    NAT -->|"SOAP · HTTPS"| DS1
    NAT -->|"REST · HTTPS"| DS2
    ECS -.->|"credentials"| SM
    ECS -.->|"errors · traces"| SEN

    classDef req fill:#dae8fc,stroke:#6c8ebf,color:#000
    classDef bg fill:#f5f5f5,stroke:#999,color:#000
    classDef src fill:#ffe6cc,stroke:#d79b00,color:#000
    class CONS,GW,API,R,PG req
    class EB,SQS,CON,SC,JOB,NAT,SM,SEN bg
    class DS1,DS2,DS3 src
```

## Database

```mermaid
erDiagram
    direction LR
    company {
        varchar     vat_number      PK
        varchar     name
        varchar     address
        timestamptz created_at
    }
    data_source {
        varchar     code             PK
        varchar     type
        interval    refresh_interval
        numeric     weight
        boolean     enabled
    }
    source_data {
        bigint      id              PK
        varchar     vat_number      FK
        varchar     source          FK
        varchar     external_id
        jsonb       data
        timestamptz data_as_of
        timestamptz next_refresh_at
    }
    credit_score {
        varchar     vat_number      PK
        smallint    score
        char        band
        varchar     status
        numeric     confidence
        jsonb       sources
        timestamptz calculated_at
    }
    company ||--o{ source_data : "has"
    data_source ||--o{ source_data : "provides"
    company ||--o| credit_score : "has"
```

## Assumptions

- DS3 is in the same AWS private cloud.
- DS1 or DS3 provides the company's name and address.
