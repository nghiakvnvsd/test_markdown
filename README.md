# System Architecture Documentation

> **Document Version:** 1.0  
> **Last Updated:** December 4, 2025  
> **Project:** Order Management Renewal Service

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [High-Level Architecture](#high-level-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Architecture](#data-architecture)
5. [Integration Architecture](#integration-architecture)
6. [Security Architecture](#security-architecture)
7. [Deployment Architecture](#deployment-architecture)
8. [Data Flow Diagrams](#data-flow-diagrams)
9. [State Diagrams](#state-diagrams)
10. [Sequence Diagrams](#sequence-diagrams)

---

## Architecture Overview

The Order Management Renewal Service is a Spring Boot microservice that manages the motor insurance policy renewal lifecycle. It follows a layered architecture pattern with clear separation of concerns between presentation, business logic, and data access layers.

### Key Architecture Principles

| Principle | Implementation |
|-----------|----------------|
| **Separation of Concerns** | Distinct layers for controllers, services, repositories |
| **Dependency Injection** | Spring IoC container for loose coupling |
| **RESTful Design** | HTTP-based API following REST conventions |
| **Database Abstraction** | Repository pattern with JDBC templates |
| **Externalized Configuration** | Environment-specific property files |
| **Scheduled Operations** | Spring Scheduler for automated tasks |

---

## High-Level Architecture

```mermaid
flowchart TB
    subgraph Clients["🖥️ External Clients"]
        OS[OutSystems Portal]
        Mobile[Customer Mobile App]
        Admin[Admin Dashboard]
        External[External Systems]
    end
   
    subgraph Gateway["🔐 API Gateway Layer"]
        APIGEE[APIGEE Gateway<br/>OAuth 2.0 / JWT / API Key]
    end
   
    subgraph Application["⚙️ Order Management Renewal Service"]
        subgraph Controllers["Presentation Layer"]
            OMCtrl[OrderManagementController<br/>/motor/renewal/*]
            KmsCtrl[KmsController<br/>/motor/renewal/kms/*]
        end
       
        subgraph Services["Service Layer"]
            DistSvc[DistributeRenewalService]
            UploadSvc[UploadRenewalService]
            SubmitSvc[SubmitRenewalService]
            SearchSvc[SearchRenewalService]
            EmailSvc[EmailService]
            NebulaSvc[NebulaService]
            OrderSvc[OrderService]
            OfferSvc[OfferService]
            RuleSvc[RuleEngineService]
        end
       
        subgraph Repos["Repository Layer"]
            DistRepo[DistributeRenewalRepo]
            UploadRepo[UploadRenewalRepo]
            SearchRepo[SearchRenewalRepo]
            MotorRepo[MotorRenewalRepo]
            OrderRepo[OrderRepo]
            OfferRepo[OfferRepo]
        end
       
        subgraph Scheduler["Scheduler Component"]
            Jobs[SchedulerJob]
        end
    end
   
    subgraph DataLayer["💾 Data Layer"]
        MSSQL[(SQL Server<br/>GRPPRODUCER)]
        Oracle[(Oracle<br/>Premia DB)]
    end
   
    subgraph ExternalAPIs["🌐 External Systems"]
        POSWS[POS-WS<br/>Policy Admin]
        OpenL[OpenL<br/>Pricing Engine]
        NebulaDMS[Nebula<br/>Document Mgmt]
        LC2[Liberty Connect<br/>Email Service]
        AWS[AWS KMS<br/>Encryption]
    end
   
    Clients --> APIGEE
    APIGEE --> Controllers
    Controllers --> Services
    Services --> Repos
    Repos --> MSSQL
    Repos --> Oracle
    Services --> ExternalAPIs
    Jobs --> Services
```

---

## Component Architecture

### Application Layers

```mermaid
flowchart TB
    subgraph PresentationLayer["📱 Presentation Layer"]
        direction LR
        C1[OrderManagementController]
        C2[KmsController]
        EH[Exception Handler]
        V[Input Validation]
    end
   
    subgraph BusinessLayer["⚙️ Business Layer"]
        direction TB
        subgraph CoreServices["Core Services"]
            DS[DistributeRenewalService]
            US[UploadRenewalService]
            SS[SubmitRenewalService]
            SRS[SearchRenewalService]
            MRS[MotorRenewalService]
        end
       
        subgraph SupportServices["Support Services"]
            ES[EmailService]
            NS[NebulaService]
            NWS[NebulaWrapperService]
            OS[OrderService]
            OFS[OfferService]
        end
       
        subgraph IntegrationServices["Integration Services"]
            RES[RuleEngineService]
            MRH[MotorRenewalHandler]
        end
    end
   
    subgraph DataLayer["💾 Data Access Layer"]
        direction LR
        DR[DistributeRenewalRepo]
        UR[UploadRenewalRepo]
        SR[SearchRenewalRepo]
        MR[MotorRenewalRepo]
        OR[OrderRepo]
        OFR[OfferRepo]
    end
   
    subgraph Infrastructure["🔧 Infrastructure"]
        direction LR
        DB1[(SQL Server)]
        DB2[(Oracle)]
        Cache[Spring Cache]
    end
   
    PresentationLayer --> BusinessLayer
    BusinessLayer --> DataLayer
    DataLayer --> Infrastructure
```

### Controller Responsibilities

```mermaid
flowchart LR
    subgraph OrderManagementController["OrderManagementController"]
        direction TB
       
        subgraph OfferOps["Offer Operations"]
            Upload[POST /offer<br/>Upload Offers]
            Get[GET /offer<br/>Get Offers]
            Update[PUT /offer<br/>Update Offers]
            Delete[DELETE /offer<br/>Delete Offers]
            Search[POST /offer/search<br/>Search Offers]
        end
       
        subgraph DistOps["Distribution Operations"]
            Distribute[POST /offer/distribute<br/>Distribute]
            Revoke[POST /offer/revokeDistribute<br/>Revoke]
        end
       
        subgraph SubmitOps["Submission Operations"]
            Submit[POST /offer/submit<br/>Submit to Premia]
            Decline[POST /offer/decline<br/>Decline Offer]
        end
       
        subgraph ReportOps["Reporting Operations"]
            Summary[GET /offer/summary<br/>Metrics]
            Print[POST /offer/print<br/>Print]
        end
       
        subgraph SchedulerOps["Scheduler Triggers"]
            SchedSubmit[GET /scheduler/offer/submit]
            SchedDist[GET /scheduler/offer/distribute]
            SchedRetrieval[GET /scheduler/offer/retrival]
            SchedNotif[GET /scheduler/renewal/notification]
        end
    end
```

### Service Dependencies

```mermaid
flowchart TB
    subgraph Services["Service Layer Dependencies"]
        DSI[DistributeRenewalServiceImpl]
        USI[UploadRenewalServiceImpl]
        SSI[SubmitRenewalServiceImpl]
        SSV3[SubmitRenewalServiceV3Impl]
        SRSI[SearchRenewalServiceImpl]
        ESI[EmailServiceImpl]
        NSI[NebulaServiceImpl]
        NWSI[NebulaWrapperServiceImpl]
        OSI[OrderServiceImpl]
        OFSI[OfferServiceImpl]
    end
   
    subgraph Repositories["Repositories"]
        DRepo[DistributeRenewalRepo]
        URepo[UploadRenewalRepo]
        SRepo[SearchRenewalRepo]
        MRepo[MotorRenewalRepo]
        ORepo[OrderRepo]
        OFRepo[OfferRepo]
    end
   
    subgraph ExternalClients["External Clients"]
        RT[RestTemplate]
        JC[Jersey Client]
        JT[JdbcTemplate]
    end
   
    DSI --> DRepo
    DSI --> ESI
    DSI --> NWSI
    DSI --> RT
   
    USI --> URepo
    USI --> JT
   
    SSI --> MRepo
    SSI --> JT
   
    SSV3 --> MRepo
    SSV3 --> JT
   
    SRSI --> SRepo
   
    NSI --> RT
    NWSI --> RT
   
    OSI --> ORepo
    OSI --> NWSI
   
    OFSI --> OFRepo
```

---

## Data Architecture

### Database Schema Overview

```mermaid
erDiagram
    MOTOR_UNDERWRITTEN_RENEWAL_OFFER ||--o{ OFFER_STATUS_HISTORY : tracks
    MOTOR_UNDERWRITTEN_RENEWAL_OFFER ||--o| RENEWAL_ORDER : creates
    MOTOR_UNDERWRITTEN_RENEWAL_OFFER ||--o{ RENEWAL_NOTIFICATION : triggers
   
    MOTOR_UNDERWRITTEN_RENEWAL_OFFER {
        int id PK "Auto-increment ID"
        string policy_no UK "Policy Number"
        date pol_from_date "Policy Start Date"
        date pol_to_date "Policy End Date"
        string producer_code "Producer Code"
        string producer_name "Producer Name"
        string product_type "Product Type"
        string cover_type "Coverage Type"
        string insured_name "Insured Name"
        string insured_email "Customer Email"
        decimal new_sum_insured "Sum Insured"
        decimal new_gross_premium "Gross Premium"
        int status_code "Current Status"
        string failure_reason "Failure Reason"
        string renewal_notice_id "Document ID"
        int email_count "Emails Sent"
        datetime created_date "Created Date"
        string created_by "Created By"
        datetime modified_date "Modified Date"
        string modified_by "Modified By"
    }
   
    OFFER_STATUS_HISTORY {
        int id PK
        int offer_id FK
        int status_code
        string status_description
        string failure_reason
        datetime status_date
        string updated_by
    }
   
    RENEWAL_ORDER {
        int order_id PK
        int offer_id FK
        string order_status
        string cover_note_number
        datetime submitted_date
        datetime completed_date
        string nebula_doc_id
    }
   
    RENEWAL_NOTIFICATION {
        int notification_id PK
        int offer_id FK
        string notification_type
        string recipient_email
        datetime sent_date
        string send_status
    }
```

### Database Connections

```mermaid
flowchart LR
    subgraph App["Application"]
        DS1[Primary DataSource<br/>HikariCP Pool]
        DS2[Oracle DataSource<br/>HikariCP Pool]
    end
   
    subgraph Databases["Databases"]
        subgraph MSSQL["SQL Server"]
            GRP[(GRPPRODUCER)]
        end
       
        subgraph ORA["Oracle"]
            PREM[(Premia)]
        end
    end
   
    DS1 -->|"Max Pool: 20<br/>Idle: 5<br/>Timeout: 30s"| GRP
    DS2 -->|"Max Pool: 20<br/>Idle: 5<br/>Timeout: 30s"| PREM
```

### Data Flow Between Systems

```mermaid
flowchart LR
    subgraph Source["📥 Source Systems"]
        Premia1[(Premia<br/>Policy Data)]
        Upload[Batch Upload<br/>CSV/JSON]
    end
   
    subgraph Processing["⚙️ Processing"]
        OMRS[Order Mgmt<br/>Renewal Service]
    end
   
    subgraph Storage["💾 Storage"]
        MSSQL[(SQL Server<br/>Staging)]
    end
   
    subgraph Target["📤 Target Systems"]
        Premia2[(Premia<br/>ETL Target)]
        Nebula[Nebula<br/>Documents]
        Email[Email<br/>Notifications]
    end
   
    Premia1 -->|Offer Retrieval| OMRS
    Upload -->|Bulk Upload| OMRS
    OMRS -->|Store| MSSQL
    OMRS -->|Submit| Premia2
    OMRS -->|Update Metadata| Nebula
    OMRS -->|Send| Email
```

---

## Integration Architecture

### External System Integration Map

```mermaid
flowchart TB
    subgraph OMRS["Order Management Renewal Service"]
        Core[Core Application]
    end
   
    subgraph POS["POS-WS Integration"]
        direction TB
        POSWS[POS-WS API]
        SyncAPI[Motor Sync API<br/>/api/apc/motor/rn/sync]
        ApprovalAPI[Document Approval API<br/>/api/apc/document/change/approve]
        RevokeAPI[Revoke Distribution API<br/>/api/apc/motor/rn/revokeDistribute]
    end
   
    subgraph OpenLInt["OpenL Integration"]
        direction TB
        OpenL[OpenL Rules Engine]
        CalcAPI[Calculate Premium<br/>/calculate]
        RulesAPI[Get All Rules<br/>/getAllOptionList]
    end
   
    subgraph NebulaInt["Nebula Integration"]
        direction TB
        NebulaAPI[Nebula DMS API]
        MetaAPI[Content Metadata<br/>/content-metadata]
        BulkAPI[Bulk Update<br/>/bulkupdate]
        WrapperAPI[Nebula Wrapper<br/>/file]
    end
   
    subgraph EmailInt["Email Integration"]
        direction TB
        LC2[Liberty Connect v2]
        EmailAPI[Email API<br/>/v1/email]
    end
   
    subgraph AWSInt["AWS Integration"]
        direction TB
        KMS[AWS KMS]
        EncAPI[Encrypt/Decrypt]
    end
   
    Core -->|HTTPS + x-auth-token| POSWS
    Core -->|HTTPS| OpenL
    Core -->|HTTPS + OAuth| NebulaAPI
    Core -->|HTTPS + API Key| LC2
    Core -->|AWS SDK| KMS
```

### Integration Protocols

| System | Protocol | Auth Method | Data Format |
|--------|----------|-------------|-------------|
| POS-WS | HTTPS/REST | x-auth-token | JSON |
| OpenL | HTTPS/REST | API Key | JSON |
| Nebula DMS | HTTPS/REST | OAuth 2.0 | JSON |
| Liberty Connect | HTTPS/REST | API Key | JSON |
| AWS KMS | HTTPS | IAM/SDK | Binary |
| Premia | JDBC | DB Auth | SQL |

### Integration Sequence - Distribution Flow

```mermaid
sequenceDiagram
    autonumber
    participant C as 🖥️ Client
    participant OM as ⚙️ Order Mgmt Service
    participant DB as 💾 SQL Server
    participant POS as 📋 POS-WS
    participant NEB as 📁 Nebula
    participant LC as 📧 Liberty Connect
   
    C->>OM: POST /offer/distribute
    OM->>DB: Get offers by ID
    DB-->>OM: Offer details
   
    OM->>DB: Update status to 33 (Distributing)
   
    OM->>POS: Motor Sync API
    POS-->>OM: Sync response
   
    alt Sync Success
        OM->>POS: Document Approval API
        POS-->>OM: Approval response
       
        OM->>NEB: Update metadata
        NEB-->>OM: Metadata updated
       
        OM->>DB: Create order record
       
        OM->>LC: Send notification email
        LC-->>OM: Email sent
       
        OM->>DB: Update status to 31 (Distributed)
        OM->>DB: Update email count
       
        OM-->>C: Success response
    else Sync Failed
        OM->>DB: Update status to 32 (Failed)
        OM-->>C: Error response with details
    end
```

---

## Security Architecture

### Security Layers

```mermaid
flowchart TB
    subgraph External["🌐 External"]
        Client[API Client]
    end
   
    subgraph Gateway["🔐 API Gateway"]
        APIGEE[APIGEE]
        RateLimit[Rate Limiting]
        APIKey[API Key Validation]
    end
   
    subgraph AppSecurity["🔒 Application Security"]
        JWTFilter[JWT Authentication Filter]
        SecConfig[Spring Security Config]
        JWKS[JWKS Validation]
    end
   
    subgraph DataSecurity["🔐 Data Security"]
        Jasypt[Jasypt Encryption]
        KMS[AWS KMS]
        TLS[TLS 1.2+]
    end
   
    Client -->|HTTPS| Gateway
    Gateway -->|Validated Request| AppSecurity
    AppSecurity -->|Authenticated| DataSecurity
```

### Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant C as 🖥️ Client
    participant GW as 🔐 API Gateway
    participant JF as 🔑 JWT Filter
    participant JWKS as 📋 JWKS Endpoint
    participant S as ⚙️ Service
   
    C->>GW: Request + x-api-key + Bearer Token
    GW->>GW: Validate API Key
   
    alt API Key Valid
        GW->>JF: Forward Request
...
