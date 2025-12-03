# System Architecture

> **Project:** Order Management Renewal Service  
> **Last Updated:** December 4, 2025

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Component Architecture](#component-architecture)
3. [Data Flow Diagrams](#data-flow-diagrams)
4. [Authentication Flow](#authentication-flow)
5. [Database Architecture](#database-architecture)
6. [Scheduler Architecture](#scheduler-architecture)
7. [External Integrations](#external-integrations)
8. [Deployment Architecture](#deployment-architecture)

---

## High-Level Architecture

The Order Management Renewal Service is a Spring Boot microservice that orchestrates motor insurance policy renewals for Liberty Insurance Hong Kong. It integrates with multiple internal and external systems.

```mermaid
flowchart TB
    subgraph External["External Clients"]
        WebPortal[Web Portal]
        MobileApp[Mobile App]
        InternalSystems[Internal Systems]
    end
   
    subgraph Gateway["API Gateway Layer"]
        Apigee[Apigee API Gateway]
        Auth[JWT Authentication]
    end
   
    subgraph Application["Order Management Renewal Service"]
        Controller[REST Controllers]
        Services[Business Services]
        Scheduler[Scheduled Jobs]
        Repositories[Repository Layer]
    end
   
    subgraph Data["Data Layer"]
        MSSQL[(MS SQL Server<br/>GRPPRODUCER)]
        Oracle[(Oracle<br/>Premia)]
    end
   
    subgraph External_Services["External Services"]
        Premia[Premia Insurance System]
        Nebula[Nebula Document Mgmt]
        OpenL[OpenL Rule Engine]
        Email[LC Email Service]
        AzureAD[Azure AD]
        AWS_KMS[AWS KMS]
    end
   
    External --> Gateway
    Gateway --> Application
    Controller --> Services
    Services --> Repositories
    Services --> Scheduler
    Repositories --> Data
    Services --> External_Services
```

---

## Component Architecture

### Application Layers

```mermaid
flowchart TB
    subgraph Presentation["Presentation Layer"]
        OrderMgmtController[OrderManagementController]
        KmsController[KmsController]
    end
   
    subgraph Business["Business Logic Layer"]
        subgraph Services["Service Interfaces"]
            DistributeSvc[DistributeRenewalService]
            SubmitSvc[SubmitRenewalService]
            UploadSvc[UploadRenewalService]
            SearchSvc[SearchRenewalService]
            MotorSvc[MotorUnderwrittenRenewalService]
            EmailSvc[EmailService]
            NebulaSvc[NebulaService]
            OrderSvc[OrderService]
            OfferSvc[OfferService]
            RuleEngineSvc[RenewalOfferRuleEngineService]
        end
       
        subgraph Implementations["Service Implementations"]
            DistributeSvcImpl[DistributeRenewalServiceImpl]
            SubmitSvcImpl[SubmitRenewalServiceImpl]
            UploadSvcImpl[UploadRenewalServiceImpl]
            SearchSvcImpl[SearchRenewalServiceImpl]
            MotorSvcImpl[MotorUnderwrittenRenewalServiceImpl]
            EmailSvcImpl[EmailServiceImpl]
            NebulaSvcImpl[NebulaServiceImpl]
            OrderSvcImpl[OrderServiceImpl]
            OfferSvcImpl[OfferServiceImpl]
        end
    end
   
    subgraph Repository["Data Access Layer"]
        DistributeRepo[DistributeRenewalServiceRepo]
        UploadRepo[UploadRenewalServiceRepo]
        SearchRepo[SearchRenewalServiceRepo]
        MotorRepo[MotorUnderwrittenRenewalRepo]
        OrderRepo[OrderRepo]
        OfferRepo[OfferRepo]
    end
   
    Presentation --> Services
    Services --> Implementations
    Implementations --> Repository
```

### Service Responsibilities

```mermaid
flowchart LR
    subgraph Upload["Upload Domain"]
        UploadService[Upload Service]
        UploadService --> |Save Offers| Database1[(Database)]
        UploadService --> |Validate| RuleEngine[Rule Engine]
    end
   
    subgraph Distribute["Distribution Domain"]
        DistributeService[Distribute Service]
        DistributeService --> |Send Email| EmailService[Email Service]
        DistributeService --> |Update Status| Database2[(Database)]
        DistributeService --> |Sync| PremiaAPI[Premia API]
    end
   
    subgraph Submit["Submit Domain"]
        SubmitService[Submit Service]
        SubmitService --> |ETL Process| Oracle[(Oracle DB)]
        SubmitService --> |Update Status| Database3[(Database)]
    end
   
    subgraph Order["Order Domain"]
        OrderService[Order Service]
        OrderService --> |Manage Docs| NebulaService[Nebula Service]
        OrderService --> |Update Status| Database4[(Database)]
    end
```

---

## Data Flow Diagrams

### Renewal Offer Upload Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Controller as OrderMgmtController
    participant Service as UploadRenewalService
    participant Repo as UploadRenewalServiceRepo
    participant DB as MS SQL Server
   
    Client->>Controller: POST /motor/renewal/offer
    Controller->>Controller: Validate Request
    Controller->>Service: saveUnderWrittenRenewalOffers()
   
    loop For Each Record
        Service->>Service: Validate Business Rules
        Service->>Repo: Save Record
        Repo->>DB: INSERT/UPDATE
        DB-->>Repo: Result
        Repo-->>Service: Status
    end
   
    Service-->>Controller: RenewalOfferListSaveResponse
    Controller-->>Client: HTTP 200 Response
```

### Renewal Offer Distribution Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Controller as OrderMgmtController
    participant DistSvc as DistributeRenewalService
    participant Repo as DistributeRenewalServiceRepo
    participant Premia as Premia API
    participant Email as Email Service
    participant DB as MS SQL Server
   
    Client->>Controller: POST /motor/renewal/offer/distribute
    Controller->>DistSvc: getStatusDetails()
    DistSvc->>Repo: getMotorRenewalStatus()
    Repo->>DB: Query Status
    DB-->>Repo: Status List
    Repo-->>DistSvc: StatusDetails
   
    alt Status = Ready for Distribution
        DistSvc->>Repo: updateMotorRenewalStatus(33)
        DistSvc->>Premia: Sync Renewal Data
        Premia-->>DistSvc: Sync Response
       
        alt Sync Success
            DistSvc->>Email: Send Notification Email
            Email-->>DistSvc: Email Status
            DistSvc->>Repo: updateMotorRenewalStatus(31)
        else Sync Failed
            DistSvc->>Repo: updateMotorRenewalStatus(ERROR)
        end
    end
   
    DistSvc-->>Controller: DistributeUnderwrittenRenewalOfferRes
    Controller-->>Client: HTTP 200 Response
```

### Submit to Premia Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Controller as OrderMgmtController
    participant SubmitSvc as SubmitRenewalService
    participant Repo as Repository
    participant Oracle as Oracle Database
    participant MSSQL as MS SQL Server
   
    Client->>Controller: POST /motor/renewal/offer/submit
    Controller->>SubmitSvc: submitRenewalV3()
    SubmitSvc->>Repo: Get Offer Details
    Repo->>MSSQL: Query Offers
    MSSQL-->>Repo: Offer Data
    Repo-->>SubmitSvc: Offer List
   
    loop For Each Offer
        SubmitSvc->>Oracle: Call P_PREMIA_RENEW_ETL
        Oracle-->>SubmitSvc: ETL Result
       
        alt ETL Success
            SubmitSvc->>Repo: Update Status to Submitted
            Repo->>MSSQL: UPDATE
        else ETL Failed
            SubmitSvc->>Repo: Update Status to Failed
            Repo->>MSSQL: UPDATE with Error
        end
    end
   
    SubmitSvc-->>Controller: SubmitRenewalOfferResponse
    Controller-->>Client: HTTP 200 Response
```

---

## Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Gateway as Apigee Gateway
    participant Filter as JwtAuthenticationFilter
    participant AzureAD as Azure AD JWKS
    participant CustomerJWKS as Customer JWKS
    participant Controller as REST Controller
   
    Client->>Gateway: Request + JWT Token
    Gateway->>Filter: Forward Request
    Filter->>Filter: Extract JWT from Header
   
    alt Azure AD Token
        Filter->>AzureAD: Validate Token
        AzureAD-->>Filter: Token Valid/Invalid
    else Customer Token
        Filter->>CustomerJWKS: Validate Token
        CustomerJWKS-->>Filter: Token Valid/Invalid
    end
   
    alt Token Valid
        Filter->>Filter: Set Security Context
        Filter->>Controller: Forward to Controller
        Controller-->>Client: Response
    else Token Invalid
        Filter-->>Client: 401 Unauthorized
    end
```

---

## Database Architecture

### Multi-Database Configuration

```mermaid
flowchart TB
    subgraph Application["Order Management Service"]
        DbConfig[DbConfig.java]
        MsSqlTemplate[MsSqlJdbcTemplate]
        OracleTemplate[OracleJdbcTemplate]
    end
   
    subgraph Primary["Primary Database - MS SQL Server"]
        HikariMS[HikariCP Pool<br/>Max: 20, Min: 5]
        GRPPRODUCER[(GRPPRODUCER<br/>Application Data)]
    end
   
    subgraph Secondary["Secondary Database - Oracle"]
        HikariOracle[HikariCP Pool<br/>Max: 20, Min: 5]
        Premia[(Premia<br/>Insurance Core)]
    end
   
    DbConfig --> MsSqlTemplate
    DbConfig --> OracleTemplate
    MsSqlTemplate --> HikariMS
    OracleTemplate --> HikariOracle
    HikariMS --> GRPPRODUCER
    HikariOracle --> Premia
```

### Entity Relationships

```mermaid
erDiagram
    MOTOR_RENEWAL_OFFER ||--o{ RENEWAL_STATUS_HISTORY : has
    MOTOR_RENEWAL_OFFER {
        int id PK
        string policy_no
        date pol_from_date
        date pol_to_date
        string producer_code
        string product_type
        string insured_name
        float sum_insured
        float gross_premium
        int status_code
        datetime create_date
        string create_by
        datetime modified_date
        string modified_by
    }
   
    RENEWAL_STATUS_HISTORY {
        int id PK
        int offer_id FK
        int status_code
        datetime status_date
        string modified_by
    }
   
    RENEWAL_NOTIFICATION ||--|| MOTOR_RENEWAL_OFFER : references
    RENEWAL_NOTIFICATION {
        int id PK
        int offer_id FK
        string email_type
        datetime sent_date
        string status
    }
   
    ORDER ||--|| MOTOR_RENEWAL_OFFER : references
    ORDER {
        int id PK
        int offer_id FK
        string order_status
        string nebula_doc_id
        datetime create_date
    }
```

---

## Scheduler Architecture

### Scheduled Jobs Overview

```mermaid
flowchart TB
    subgraph Scheduler["SchedulerJob Component"]
        Submit[offerSubmitSchedulerJob<br/>Cron: configurable]
        Distribute[offerDistributeSchedulerJob<br/>Cron: configurable]
        Retrieval[offerRetrivalSchedulerJob<br/>Cron: configurable]
        Notification[renewalNotificationSchedulerJob<br/>Cron: configurable]
        AutoUpdate[omcPatchAutoUpdateSchedulerJob<br/>Cron: configurable]
        OrderStatus[orderStatusSchedulerJob<br/>Cron: configurable]
        OfferStatus[offerStatusSchedulerJob<br/>Cron: configurable]
        Reindex[reIndexingOfPolicyDocuments<br/>Cron: configurable]
    end
   
    subgraph Services["Target Services"]
        SubmitSvc[SubmitRenewalServiceV3]
        DistSvc[DistributeRenewalService]
        MotorSvc[MotorUnderwrittenRenewalService]
        OrderSvc[OrderService]
        OfferSvc[OfferService]
    end
   
    subgraph Guards["Job Running Guards"]
        JobFlag1[jobRunning]
        JobFlag2[distributeJobRunning]
        JobFlag3[offerRetrivalJobRunning]
        JobFlag4[renewalNotificationJobRunning]
        JobFlag5[omcPatchAutoUpdateJobRunning]
        JobFlag6[orderStatusJobRunning]
        JobFlag7[offerStatusJobRunning]
        JobFlag8[reIndexingDocumentsRunning]
    end
   
    Submit --> |saveUnderWrittenRenewalOffers| SubmitSvc
    Distribute --> |distributeRenewalAPIScheduler| DistSvc
    Retrieval --> |retriveOfferFromPremia| MotorSvc
    Notification --> |renewalNotificationAPIScheduler| DistSvc
    AutoUpdate --> |omcPatchAutoUpdate| MotorSvc
    OrderStatus --> |updateOrderStatus| OrderSvc
    OfferStatus --> |updateOfferStatus| OfferSvc
    Reindex --> |reIndexPolicyDocuments| OrderSvc
```

### Job State Machine

```mermaid
stateDiagram-v2
    [*] --> Idle: Application Start
    Idle --> Running: Cron Trigger
    Running --> Processing: Guard Check Pass
    Processing --> Completed: Success
    Processing --> Failed: Exception
    Completed --> Idle: Reset Flag
    Failed --> Idle: Reset Flag
   
    note right of Running
        Check if another instance
        is already running
    end note
   
    note right of Processing
        Execute business logic
        with proper error handling
    end note
```

---

## External Integrations

### Integration Points

```mermaid
flowchart LR
    subgraph OMC["Order Management Service"]
        Services[Business Services]
    end
   
    subgraph Premia_System["Premia Insurance System"]
        SyncAPI[Motor Renewal Sync API]
        ApprovalAPI[Document Approval API]
        RevokeAPI[Revoke Distribution API]
        ETL[P_PREMIA_RENEW_ETL]
    end
   
    subgraph OpenL_System["OpenL Rule Engine"]
        PricingAPI[Motor Pricing API]
        AllRulesAPI[Get All Option List API]
    end
   
    subgraph Nebula_System["Nebula Document Management"]
        SearchMetadata[Search Metadata API]
        BulkUpdate[Bulk Update API]
        WrapperAPI[Nebula Wrapper API]
    end
   
    subgraph Email_System["Email Services"]
        LCEmail[LC Email Service]
        LC2Email[LC2 Email Service]
    end
   
    subgraph Auth_System["Authentication"]
        AzureAD[Azure AD OAuth]
        CustomerJWKS[Customer JWKS]
    end
   
    Services --> |REST| Premia_System
    Services --> |REST| OpenL_System
    Services --> |REST| Nebula_System
    Services --> |REST| Email_System
    Services --> |JWKS| Auth_System
```

### Integration Configuration

| Integration | Base URL Pattern | Authentication |
|-------------|------------------|----------------|
| Premia Sync | `motor.renewal.sync.url` | Internal Certificate |
| OpenL Pricing | `motor.pricing.renewal.url` | None |
| Nebula | `nebula.searchMetadataUrl` | Azure OAuth + API Key |
| LC Email | `lcemail.urlEmail` | API Key |
| LC2 Email | `lc2email.urlEmail` | API Key |

---

## Deployment Architecture

### Container Architecture

```mermaid
flowchart TB
    subgraph Container["Docker Container"]
        subgraph Base["Base Image"]
            LMSpringBoot[lm_springboot:IS-B85]
        end
       
        subgraph Application["Application Layer"]
            JAR[api-order-mgmt-renewal-service.jar]
            Templates[Email Templates]
            Certs[SSL Certificates]
        end
       
        subgraph Agents["Monitoring Agents"]
            Contrast[Contrast Security Agent]
            Datadog[Datadog APM Agent]
        end
       
        subgraph Config["Configuration"]
            EnvVars[Environment Variables]
            Profile[Spring Profile]
        end
    end
   
    subgraph External["External Dependencies"]
        MSSQL[(MS SQL Server)]
        Oracle[(Oracle DB)]
        APIs[External APIs]
    end
   
    Base --> Application
    Application --> Agents
    Config --> JAR
    Container --> External
```

### CI/CD Pipeline

```mermaid
flowchart LR
    subgraph Source["Source Control"]
        GitHub[GitHub Repository]
    end
   
    subgraph Build["Build Stage"]
        Jenkins[Jenkins Pipeline]
        Maven[Maven Build]
        Docker[Docker Build]
    end
   
    subgraph Registry["Artifact Registry"]
        Artifactory[Liberty Artifactory]
    end
   
    subgraph Deploy["Deployment"]
        NonProd[Non-Prod Environment]
        Prod[Production Environment]
    end
   
    GitHub --> |Checkout| Jenkins
    Jenkins --> Maven
    Maven --> Docker
    Docker --> |Push| Artifactory
    Artifactory --> |Deploy| NonProd
    NonProd --> |Promote| Prod
```

---

## Security Architecture

```mermaid
flowchart TB
    subgraph External["External Access"]
        Client[API Client]
    end
   
    subgraph Security["Security Layer"]
        TLS[TLS 1.2+]
        Apigee[Apigee Gateway]
        JWT[JWT Validation]
        RBAC[Role-Based Access]
    end
   
    subgraph Application["Application Security"]
        SpringSecurity[Spring Security]
        JWTFilter[JWT Authentication Filter]
        Jasypt[Jasypt Encryption]
    end
   
    subgraph Secrets["Secrets Management"]
        AWSKMS[AWS KMS]
        EnvSecrets[Environment Secrets]
        EncryptedConfig[Encrypted Properties]
    end
   
    subgraph Monitoring["Security Monitoring"]
        Contrast[Contrast Security]
        Logging[Audit Logging]
    end
   
    Client --> TLS
    TLS --> Apigee
    Apigee --> JWT
    JWT --> SpringSecurity
    SpringSecurity --> JWTFilter
    JWTFilter --> RBAC
   
    Application --> Jasypt
    Jasypt --> Secrets
    Application --> Monitoring
```

---

## Network Architecture

```mermaid
flowchart TB
    subgraph Internet["Internet"]
        ExtClients[External Clients]
    end
   
    subgraph DMZ["DMZ"]
        Apigee[Apigee API Gateway]
    end
   
    subgraph Internal["Internal Network"]
        subgraph AppTier["Application Tier"]
            OMC[OMC Service Container]
        end
       
        subgraph DataTier["Data Tier"]
            MSSQL[MS SQL Server]
            Oracle[Oracle Database]
        end
       
        subgraph ServiceTier["Service Tier"]
            Premia[Premia Services]
            Nebula[Nebula Services]
            Email[Email Services]
        end
    end
   
    subgraph Cloud["Cloud Services"]
        AzureAD[Azure AD]
        AWS[AWS Services]
    end
   
    ExtClients --> DMZ
    DMZ --> AppTier
    AppTier --> DataTier
    AppTier --> ServiceTier
    AppTier --> Cloud
```
