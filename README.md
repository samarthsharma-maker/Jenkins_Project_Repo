``` mermaid
%%{init: {'theme':'base', 'themeVariables': { 'fontSize':'14px'}}}%%
flowchart TB
    %% ============ CLIENTS ============
    subgraph CLIENTS["Clients"]
        BROWSER["Web Browser<br/>bookmyshow.com"]
        MOBILE["Mobile App<br/>API calls"]
        PARTNER["Partner / Event Manager<br/>(PVR, Inox)"]
    end

    %% ============ EDGE / ENTRY ============
    DNS["AWS Route 53<br/>DNS &middot; Global Routing<br/>Health-based Failover"]
    WAF["AWS WAF<br/>Web Application Firewall<br/>(SQLi / XSS filtering)"]
    CDN["CloudFront CDN<br/>Caches static content<br/>at edge locations"]
    APIGW{{"API Gateway<br/>AuthN / AuthZ &middot; Rate Limiting<br/>Routing &middot; Circuit Breaker<br/>Request Transform &middot; Caching"}}
    COGNITO["AWS Cognito<br/>User Pools &middot; Federation<br/>(Google / Facebook)"]

    %% ============ MICROSERVICES (EKS) ============
    subgraph EKS["AWS EKS / ECS Cluster (Microservices)"]
        direction TB
        SEARCH["Search Service<br/>/search"]
        EVENT["Event CRUD Service<br/>/event"]
        BOOKING["Booking Service<br/>/booking<br/>(strong consistency, TTL locks)"]
        PAYMENT["Payment Service<br/>(external gateway)"]
        NOTIF["Notification Service"]
        PARTNERSVC["Partner Mgmt Service"]
        SESSION["Session Service<br/>(internal)"]
        SYNC["Data Sync Service<br/>SQL &rarr; Search index"]
        EVENTSRC["Event Sourcing Service<br/>(S3-triggered)"]
    end

    %% ============ DATA STORES ============
    subgraph DATA["Data Layer"]
        direction TB
        SQLDB[("SQL DB - RDS / Aurora<br/>Primary + Read Replicas<br/>ACID &middot; Booking source of truth")]
        ES[("Elasticsearch<br/>Fast multi-field search")]
        REDIS[("Redis<br/>Cache &middot; TTL seat locks<br/>Session store")]
        EVENTDB[("Event Service DB<br/>independent")]
        S3[("S3 Bucket<br/>Event posters / docs")]
    end

    TEXTRACT["AWS Textract<br/>Extract event data<br/>from uploaded docs"]

    %% ============ OBSERVABILITY & GOVERNANCE ============
    subgraph OPS["Observability &amp; Governance"]
        direction LR
        CW["CloudWatch"]
        CT["CloudTrail"]
        PROM["Prometheus"]
        GRAF["Grafana"]
        CONFIG["AWS Config<br/>Compliance rules"]
    end

    %% ============ DR ============
    DR["Secondary Region<br/>(Multi-region DR)<br/>Route 53 failover"]

    %% ---------- REQUEST FLOW ----------
    BROWSER --> DNS
    MOBILE --> DNS
    DNS --> WAF
    WAF --> CDN
    CDN --> APIGW
    APIGW <-->|authenticate| COGNITO

    %% ---------- ROUTING TO SERVICES ----------
    APIGW -->|"/search"| SEARCH
    APIGW -->|"/event"| EVENT
    APIGW -->|"/booking"| BOOKING

    %% ---------- SERVICE INTERACTIONS ----------
    BOOKING --> PAYMENT
    BOOKING --> NOTIF
    EVENT --> PARTNERSVC
    BOOKING -.->|seat lock TTL| REDIS
    SEARCH --> SESSION

    %% ---------- DATA ACCESS ----------
    BOOKING -->|writes| SQLDB
    SEARCH -->|reads| ES
    SEARCH -.->|simple key lookups| REDIS
    EVENT --> EVENTDB

    %% ---------- SYNC PIPELINE ----------
    SQLDB -->|change stream| SYNC
    SYNC -->|index updates| ES

    %% ---------- EVENT SOURCING PIPELINE ----------
    PARTNER --> COGNITO
    PARTNER -->|upload poster/doc| S3
    S3 -->|trigger| EVENTSRC
    EVENTSRC --> TEXTRACT
    TEXTRACT -->|structured data| EVENTSRC
    EVENTSRC -->|populate| EVENTDB

    %% ---------- OBSERVABILITY HOOKS ----------
    EKS -.->|metrics / logs| OPS
    APIGW -.->|metrics| CW
    DATA -.->|metrics| OPS

    %% ---------- DR ----------
    DNS -.->|failover| DR

    %% ============ STYLES ============
    classDef edge fill:#e3f2fd,stroke:#1565c0,stroke-width:2px;
    classDef svc fill:#e8f5e9,stroke:#2e7d32,stroke-width:1.5px;
    classDef store fill:#fff3e0,stroke:#e65100,stroke-width:1.5px;
    classDef ops fill:#f3e5f5,stroke:#6a1b9a,stroke-width:1.5px;
    classDef client fill:#eceff1,stroke:#455a64,stroke-width:1.5px;
    classDef dr fill:#ffebee,stroke:#c62828,stroke-width:2px,stroke-dasharray:5 5;

    class BROWSER,MOBILE,PARTNER client;
    class DNS,WAF,CDN,APIGW,COGNITO,TEXTRACT edge;
    class SEARCH,EVENT,BOOKING,PAYMENT,NOTIF,PARTNERSVC,SESSION,SYNC,EVENTSRC svc;
    class SQLDB,ES,REDIS,EVENTDB,S3 store;
    class CW,CT,PROM,GRAF,CONFIG ops;
    class DR dr;
```
