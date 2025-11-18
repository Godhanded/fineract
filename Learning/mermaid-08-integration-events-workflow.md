# Apache Fineract - Integration & Event-Driven Architecture

## 1. Event Bus Architecture

```mermaid
graph TB
    subgraph "Domain Services"
        LoanService[Loan Service]
        SavingsService[Savings Service]
        ClientService[Client Service]
        AccountingService[Accounting Service]
    end

    subgraph "Event Publishing Infrastructure"
        EventBus[Business Event Bus<br/>Spring ApplicationEventPublisher]
        EventSerializer[Event Serializer<br/>Apache Avro]
        EventStore[(Event Store<br/>m_business_event)]
    end

    subgraph "Internal Listeners"
        AccountingListener[Accounting Listener<br/>Auto GL Posting]
        NotificationListener[Notification Listener<br/>SMS/Email]
        AuditListener[Audit Listener<br/>Compliance Logging]
        ProvisioningListener[Provisioning Listener<br/>Auto-provisioning]
    end

    subgraph "External Integration"
        WebhookPublisher[Webhook Publisher<br/>HTTP POST]
        MessageQueue[Message Queue<br/>Kafka/ActiveMQ]
        WebhookRegistry[(Webhook Registry<br/>m_hook)]
    end

    subgraph "External Systems"
        PaymentGateway[Payment Gateway]
        CreditBureau[Credit Bureau]
        SMSGateway[SMS Gateway]
        EmailServer[Email Server]
        ThirdPartyApp[Third-party Apps]
        DataWarehouse[Data Warehouse]
    end

    LoanService -->|Publishes| EventBus
    SavingsService -->|Publishes| EventBus
    ClientService -->|Publishes| EventBus
    AccountingService -->|Publishes| EventBus

    EventBus --> EventSerializer
    EventSerializer --> EventStore

    EventBus --> AccountingListener
    EventBus --> NotificationListener
    EventBus --> AuditListener
    EventBus --> ProvisioningListener

    EventBus --> WebhookPublisher
    EventBus --> MessageQueue

    WebhookPublisher --> WebhookRegistry
    WebhookRegistry --> WebhookPublisher

    WebhookPublisher --> ThirdPartyApp
    MessageQueue --> DataWarehouse

    NotificationListener --> SMSGateway
    NotificationListener --> EmailServer

    AccountingListener --> PaymentGateway
    AuditListener --> CreditBureau

    style EventBus fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style EventStore fill:#e0f2f1,stroke:#004d40,stroke-width:2px
    style WebhookPublisher fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
```

## 2. Event Publishing Flow

```mermaid
sequenceDiagram
    participant Service as Domain Service<br/>(Loan/Savings)
    participant EventBus as Event Bus
    participant Serializer as Avro Serializer
    participant EventStore as Event Store DB
    participant SyncListener as Synchronous Listeners<br/>(Accounting, Audit)
    participant AsyncPublisher as Async Publisher<br/>(Webhooks, Queue)
    participant External as External Systems

    Service->>Service: Business Operation<br/>(e.g., Approve Loan)

    Service->>EventBus: publishEvent(LoanApprovedEvent)

    Note over EventBus: Event published in<br/>same transaction as<br/>business operation

    par Synchronous Processing
        EventBus->>SyncListener: onApplicationEvent(event)
        SyncListener->>SyncListener: Generate GL Entries
        SyncListener->>SyncListener: Log Audit Trail
        SyncListener-->>EventBus: Complete
    end

    EventBus->>Serializer: Serialize Event to Avro
    Serializer->>Serializer: Convert to Binary Format
    Serializer-->>EventBus: Avro Bytes

    EventBus->>EventStore: INSERT into m_business_event
    EventStore-->>EventBus: Event Stored

    EventBus-->>Service: Event Published

    Note over Service: Business operation<br/>transaction commits

    par Asynchronous Processing (Post-Commit)
        EventBus->>AsyncPublisher: publishAsync(event)
        AsyncPublisher->>AsyncPublisher: Load Webhook Configs
        AsyncPublisher->>External: HTTP POST to webhook URLs
        External-->>AsyncPublisher: 200 OK
        AsyncPublisher->>AsyncPublisher: Publish to Kafka
    end
```

## 3. Event Types & Schema

```mermaid
classDiagram
    class BusinessEvent {
        <<abstract>>
        +Long id
        +String type
        +Date createdDate
        +String tenantId
        +Map~String,Object~ data
    }

    class LoanEvent {
        +Long loanId
        +String accountNo
        +Long clientId
        +BigDecimal amount
    }

    class SavingsEvent {
        +Long savingsId
        +String accountNo
        +Long clientId
        +BigDecimal balance
    }

    class ClientEvent {
        +Long clientId
        +String clientName
        +Long officeId
    }

    class TransactionEvent {
        +Long transactionId
        +String transactionType
        +BigDecimal amount
        +Date transactionDate
    }

    BusinessEvent <|-- LoanEvent
    BusinessEvent <|-- SavingsEvent
    BusinessEvent <|-- ClientEvent
    BusinessEvent <|-- TransactionEvent

    LoanEvent <|-- LoanApprovedEvent
    LoanEvent <|-- LoanDisbursedEvent
    LoanEvent <|-- LoanRepaymentEvent
    LoanEvent <|-- LoanWrittenOffEvent
    LoanEvent <|-- LoanClosedEvent

    SavingsEvent <|-- SavingsActivatedEvent
    SavingsEvent <|-- SavingsDepositEvent
    SavingsEvent <|-- SavingsWithdrawalEvent
    SavingsEvent <|-- InterestPostedEvent

    ClientEvent <|-- ClientCreatedEvent
    ClientEvent <|-- ClientActivatedEvent
    ClientEvent <|-- ClientRejectedEvent
    ClientEvent <|-- ClientClosedEvent

    TransactionEvent <|-- GLJournalEntryCreated
    TransactionEvent <|-- ChargeAppliedEvent
    TransactionEvent <|-- FeeWaivedEvent

    note for BusinessEvent "All events extend BusinessEvent\nSerialized using Apache Avro\nStored in m_business_event table"
```

## 4. Webhook Configuration & Delivery

```mermaid
sequenceDiagram
    participant Admin
    participant API as Webhook API
    participant DB as Database
    participant EventBus as Event Bus
    participant WebhookSvc as Webhook Service
    participant RetryQueue as Retry Queue
    participant External as External System

    Admin->>API: POST /hooks<br/>Register Webhook

    Note over Admin,API: Configuration:<br/>- URL: https://example.com/webhook<br/>- Events: [LoanApproved, LoanDisbursed]<br/>- Auth: Bearer token<br/>- Retry: 3 attempts

    API->>DB: INSERT into m_hook<br/>- name<br/>- url<br/>- events (JSON array)<br/>- is_active

    API-->>Admin: Webhook Registered

    Note over EventBus: Business event occurs<br/>(Loan Approved)

    EventBus->>WebhookSvc: Event: LoanApprovedEvent

    WebhookSvc->>DB: Query Active Webhooks<br/>WHERE 'LoanApproved' IN events

    DB-->>WebhookSvc: Matching Webhooks (2 found)

    loop For Each Webhook
        WebhookSvc->>WebhookSvc: Build Payload:<br/>- event type<br/>- entity ID<br/>- tenant ID<br/>- timestamp<br/>- data

        WebhookSvc->>External: HTTP POST to webhook URL<br/>Headers: Authorization, Content-Type<br/>Body: JSON payload

        alt Success (2xx response)
            External-->>WebhookSvc: 200 OK
            WebhookSvc->>DB: Log Success in m_hook_history
        else Failure (4xx/5xx or timeout)
            External-->>WebhookSvc: 500 Internal Server Error
            WebhookSvc->>RetryQueue: Enqueue for Retry<br/>Attempt 1 of 3
            WebhookSvc->>DB: Log Failure in m_hook_history
        end
    end

    Note over RetryQueue: Wait 5 minutes

    RetryQueue->>WebhookSvc: Retry Webhook Delivery

    WebhookSvc->>External: HTTP POST (Retry 1)

    alt Success
        External-->>WebhookSvc: 200 OK
        WebhookSvc->>DB: Log Retry Success
    else Failure
        External-->>WebhookSvc: 503 Service Unavailable
        WebhookSvc->>RetryQueue: Enqueue for Retry<br/>Attempt 2 of 3
    end

    Note over RetryQueue: Exponential backoff<br/>Wait 15 minutes

    RetryQueue->>WebhookSvc: Retry Webhook Delivery

    WebhookSvc->>External: HTTP POST (Retry 2)

    alt Success
        External-->>WebhookSvc: 200 OK
    else Max Retries Exceeded
        External-->>WebhookSvc: Timeout
        WebhookSvc->>DB: Log Final Failure
        WebhookSvc->>Admin: Email Alert:<br/>Webhook delivery failed after 3 attempts
    end
```

## 5. Message Queue Integration (Kafka)

```mermaid
flowchart LR
    subgraph "Fineract Instance"
        EventBus[Event Bus]
        KafkaProducer[Kafka Producer]
        EventFilter[Event Filter<br/>Configured Events Only]
    end

    subgraph "Kafka Cluster"
        Topic1[Topic: fineract.loans]
        Topic2[Topic: fineract.savings]
        Topic3[Topic: fineract.clients]
        Topic4[Topic: fineract.transactions]
    end

    subgraph "External Consumers"
        Analytics[Analytics Service<br/>Real-time Dashboards]
        DataLake[Data Lake<br/>Historical Analysis]
        RiskEngine[Risk Engine<br/>Portfolio Monitoring]
        ReportingTool[Reporting Tool<br/>BI Reports]
        MobileNotif[Mobile App<br/>Push Notifications]
    end

    EventBus --> EventFilter
    EventFilter --> KafkaProducer

    KafkaProducer -->|LoanEvents| Topic1
    KafkaProducer -->|SavingsEvents| Topic2
    KafkaProducer -->|ClientEvents| Topic3
    KafkaProducer -->|TransactionEvents| Topic4

    Topic1 --> Analytics
    Topic1 --> RiskEngine

    Topic2 --> Analytics
    Topic2 --> DataLake

    Topic3 --> DataLake
    Topic3 --> MobileNotif

    Topic4 --> ReportingTool
    Topic4 --> DataLake

    style EventBus fill:#fff3e0,stroke:#e65100
    style KafkaProducer fill:#e1f5ff,stroke:#01579b
    style Topic1 fill:#f3e5f5,stroke:#4a148c
    style Topic2 fill:#f3e5f5,stroke:#4a148c
    style Topic3 fill:#f3e5f5,stroke:#4a148c
    style Topic4 fill:#f3e5f5,stroke:#4a148c
```

## 6. Payment Gateway Integration

```mermaid
sequenceDiagram
    participant Client
    participant API as Fineract API
    participant LoanService as Loan Service
    participant PaymentGW as Payment Gateway<br/>Integration Service
    participant ExternalPG as External Payment<br/>Gateway (Stripe/PayPal)
    participant EventBus as Event Bus
    participant AccountingSvc as Accounting Service

    Client->>API: POST /loans/{id}/repayment<br/>Amount: $500<br/>Payment Method: Online

    API->>LoanService: Process Repayment

    LoanService->>PaymentGW: Initiate Payment<br/>- Amount: $500<br/>- Customer ID<br/>- Payment Method

    PaymentGW->>ExternalPG: POST /charges<br/>Create Payment Intent

    ExternalPG-->>PaymentGW: Payment Intent Created<br/>Status: pending<br/>Intent ID: pi_123456

    PaymentGW-->>LoanService: Payment Initiated<br/>External Ref: pi_123456

    LoanService->>LoanService: Create Transaction Record<br/>Status: PENDING_PAYMENT<br/>External ID: pi_123456

    LoanService-->>API: Payment Initiated
    API-->>Client: Payment Pending<br/>Complete on Gateway

    Note over Client,ExternalPG: Customer completes<br/>payment on gateway UI

    ExternalPG->>ExternalPG: Process Payment

    alt Payment Success
        ExternalPG->>PaymentGW: Webhook: payment_intent.succeeded<br/>Intent ID: pi_123456

        PaymentGW->>PaymentGW: Verify Webhook Signature

        PaymentGW->>LoanService: Payment Confirmed<br/>Amount: $500<br/>External ID: pi_123456

        LoanService->>LoanService: Find Transaction by External ID
        LoanService->>LoanService: Update Transaction Status: SUCCESS
        LoanService->>LoanService: Apply Repayment to Loan
        LoanService->>LoanService: Update Loan Balances

        LoanService->>EventBus: Publish LoanRepaymentEvent

        EventBus->>AccountingSvc: Generate GL Entries:<br/>DR Bank<br/>CR Loan Portfolio

        AccountingSvc->>AccountingSvc: Post Journal Entries

        LoanService->>PaymentGW: Acknowledge Webhook
        PaymentGW-->>ExternalPG: 200 OK

    else Payment Failed
        ExternalPG->>PaymentGW: Webhook: payment_intent.failed<br/>Reason: insufficient_funds

        PaymentGW->>LoanService: Payment Failed<br/>External ID: pi_123456

        LoanService->>LoanService: Update Transaction Status: FAILED
        LoanService->>EventBus: Publish PaymentFailedEvent

        EventBus->>Client: Notification: Payment Failed
    end
```

## 7. Credit Bureau Integration

```mermaid
flowchart TB
    Start([Loan Application<br/>Submitted]) --> CheckConfig{Credit Bureau<br/>Integration<br/>Enabled?}

    CheckConfig -->|No| SkipCB[Skip Credit Check]
    CheckConfig -->|Yes| LoadConfig[Load Bureau Config:<br/>- API Endpoint<br/>- Credentials<br/>- Bureau Type]

    LoadConfig --> BuildRequest[Build Credit Report Request:<br/>- Client National ID<br/>- Client Name<br/>- Date of Birth]

    BuildRequest --> SendRequest[HTTP POST to Bureau API]

    SendRequest --> WaitResponse{Response<br/>Received?}

    WaitResponse -->|Timeout| LogError[Log Error:<br/>Bureau Unavailable]
    WaitResponse -->|Success| ParseResponse[Parse Bureau Response:<br/>- Credit Score<br/>- Active Loans<br/>- Default History<br/>- Payment Behavior]

    LogError --> ManualReview[Flag for Manual Review]

    ParseResponse --> StoreCreditReport[Store Credit Report<br/>in m_credit_bureau_report]

    StoreCreditReport --> EvaluateScore{Credit Score<br/>&gt;= Threshold?}

    EvaluateScore -->|Yes| AutoApprove[Auto-Approve Eligible<br/>Set Flag: credit_check_passed]
    EvaluateScore -->|No| RejectOrReview{Auto-Reject<br/>Enabled?}

    RejectOrReview -->|Yes| AutoReject[Auto-Reject Application]
    RejectOrReview -->|No| ManualReview

    AutoApprove --> PublishEvent[Publish Event:<br/>CreditCheckCompleted]
    AutoReject --> PublishEvent
    ManualReview --> PublishEvent

    PublishEvent --> End([Continue Loan Workflow])

    SkipCB --> End

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style EvaluateScore fill:#fff9c4
    style AutoReject fill:#ffcdd2
```

## 8. SMS & Email Notification Flow

```mermaid
sequenceDiagram
    participant EventBus as Event Bus
    participant NotifListener as Notification Listener
    participant TemplateEngine as Template Engine
    participant DB as Database
    participant SMSGateway as SMS Gateway
    participant EmailServer as Email Server
    participant Client
    participant LoanOfficer

    EventBus->>NotifListener: Event: LoanApprovedEvent<br/>Loan ID: 12345

    NotifListener->>DB: Query Notification Config<br/>Event: LoanApproved

    DB-->>NotifListener: Notification Rules:<br/>- Send SMS to Client<br/>- Send Email to Loan Officer

    par SMS Notification
        NotifListener->>DB: Load Client Phone Number
        DB-->>NotifListener: Phone: +1234567890

        NotifListener->>TemplateEngine: Render SMS Template<br/>Template: "Your loan #{{accountNo}} has been approved"
        TemplateEngine-->>NotifListener: "Your loan #LAA00012345 has been approved"

        NotifListener->>SMSGateway: Send SMS<br/>To: +1234567890<br/>Message: "Your loan..."

        alt SMS Success
            SMSGateway-->>NotifListener: Delivered
            NotifListener->>DB: Log SMS History: SUCCESS
        else SMS Failure
            SMSGateway-->>NotifListener: Failed: Invalid number
            NotifListener->>DB: Log SMS History: FAILED
        end
    and Email Notification
        NotifListener->>DB: Load Loan Officer Email
        DB-->>NotifListener: Email: officer@example.com

        NotifListener->>TemplateEngine: Render Email Template<br/>Template: email_loan_approved.html
        TemplateEngine-->>NotifListener: Rendered HTML

        NotifListener->>EmailServer: Send Email<br/>To: officer@example.com<br/>Subject: "Loan Approved: LAA00012345"<br/>Body: [HTML]

        alt Email Success
            EmailServer-->>NotifListener: Sent
            NotifListener->>DB: Log Email History: SUCCESS
        else Email Failure
            EmailServer-->>NotifListener: Failed: SMTP error
            NotifListener->>DB: Log Email History: FAILED
            NotifListener->>NotifListener: Retry after 5 min
        end
    end

    NotifListener->>SMSGateway: Deliver to Client
    SMSGateway->>Client: SMS Received

    NotifListener->>EmailServer: Deliver to Loan Officer
    EmailServer->>LoanOfficer: Email Received
```

## 9. Event Store & Audit Trail

```mermaid
erDiagram
    M_BUSINESS_EVENT {
        BIGINT id PK
        VARCHAR type "LoanApproved, LoanDisbursed, etc"
        VARCHAR category "LOAN, SAVINGS, CLIENT"
        BIGINT entity_id "Loan/Savings/Client ID"
        TEXT schema_version "Avro schema version"
        BLOB serialized_data "Avro binary format"
        DATETIME created_date
        BIGINT tenant_id FK
        VARCHAR created_by "User who triggered event"
        BIGINT office_id FK
    }

    M_HOOK {
        BIGINT id PK
        VARCHAR name "Third-party webhook name"
        VARCHAR url "https://example.com/webhook"
        TEXT events "JSON array of event types"
        TINYINT is_active
        VARCHAR content_type "application/json"
        TEXT template "Optional custom template"
    }

    M_HOOK_REGISTERED_EVENTS {
        BIGINT id PK
        BIGINT hook_id FK
        VARCHAR entity_name "Loan, Client"
        VARCHAR action_name "CREATE, APPROVE, DISBURSE"
    }

    M_HOOK_HISTORY {
        BIGINT id PK
        BIGINT hook_id FK
        TEXT request "HTTP request sent"
        TEXT response "HTTP response received"
        INT status_code "200, 500, etc"
        DATETIME created_date
        TINYINT is_success
    }

    M_NOTIFICATION_HISTORY {
        BIGINT id PK
        VARCHAR notification_type "SMS, EMAIL"
        BIGINT recipient_id FK "Client/User ID"
        VARCHAR recipient_contact "Phone/Email"
        TEXT message_content
        VARCHAR status "SENT, FAILED, PENDING"
        DATETIME sent_date
        BIGINT business_event_id FK
    }

    M_HOOK ||--o{ M_HOOK_REGISTERED_EVENTS : "registers"
    M_HOOK ||--o{ M_HOOK_HISTORY : "delivery logs"
    M_BUSINESS_EVENT ||--o{ M_NOTIFICATION_HISTORY : "triggers"
```

## 10. External API Integration Patterns

```mermaid
graph TB
    subgraph "Integration Patterns"
        Webhooks[Webhooks<br/>Push-based<br/>Real-time]
        Polling[API Polling<br/>Pull-based<br/>Periodic]
        MessageQueue[Message Queue<br/>Pub/Sub<br/>Decoupled]
        DirectAPI[Direct API Calls<br/>Synchronous<br/>Request/Response]
    end

    subgraph "Use Cases"
        PaymentUpdate[Payment Status Updates]
        DataSync[Data Synchronization]
        EventStream[Event Streaming]
        OnDemand[On-Demand Queries]
    end

    subgraph "External Systems"
        PaymentGW[Payment Gateway]
        AccountingSystem[Accounting System]
        CRM[CRM System]
        MobileApp[Mobile App Backend]
        Analytics[Analytics Platform]
    end

    PaymentUpdate --> Webhooks
    DataSync --> Polling
    EventStream --> MessageQueue
    OnDemand --> DirectAPI

    Webhooks --> PaymentGW
    Webhooks --> MobileApp

    Polling --> AccountingSystem
    Polling --> CRM

    MessageQueue --> Analytics

    DirectAPI --> PaymentGW
    DirectAPI --> AccountingSystem

    style Webhooks fill:#e3f2fd,stroke:#1976d2
    style Polling fill:#f3e5f5,stroke:#7b1fa2
    style MessageQueue fill:#fff3e0,stroke:#f57c00
    style DirectAPI fill:#e8f5e9,stroke:#388e3c
```

## Key Integration Concepts

### Event-Driven Architecture

Fineract implements event-driven architecture using:
- **Spring ApplicationEventPublisher**: For internal event bus
- **Apache Avro**: For event serialization (schema evolution support)
- **Synchronous listeners**: Execute in same transaction (accounting, audit)
- **Asynchronous publishers**: Execute post-commit (webhooks, message queue)

### Event Types

Over 100 business events across domains:
- **Loan events**: Application, approval, disbursement, repayment, write-off, closure
- **Savings events**: Activation, deposit, withdrawal, interest posting
- **Client events**: Creation, activation, rejection, transfer, closure
- **Transaction events**: Journal entries, fee application, charge waiver
- **System events**: COB completion, batch job execution

### Webhook Delivery

- **Configuration**: Webhooks registered with URL, events, auth headers
- **Retry logic**: 3 attempts with exponential backoff (5min, 15min, 45min)
- **Delivery history**: All webhook deliveries logged with response status
- **Security**: Support for Bearer token, API key, Basic auth
- **Payload**: JSON format with event type, tenant ID, entity ID, timestamp, data

### Message Queue Integration

- **Kafka topics**: Separate topics per domain (loans, savings, clients)
- **Event filter**: Only configured events are published to queue
- **Consumer groups**: Multiple consumers can process same events
- **Ordering**: Events for same entity maintain order within partition
- **Retention**: Configurable retention period (default 7 days)

### Payment Gateway Integration

- **Supported gateways**: Stripe, PayPal, Razorpay, custom implementations
- **Payment flow**:
  1. Fineract initiates payment intent
  2. Customer completes payment on gateway
  3. Gateway sends webhook on completion
  4. Fineract processes webhook, updates transaction
  5. Accounting entries generated automatically
- **Reconciliation**: Scheduled job matches gateway transactions with Fineract records

### Credit Bureau Integration

- **Supported bureaus**: Equifax, Experian, TransUnion, local bureaus
- **Credit check triggers**: Loan application, client activation
- **Data retrieved**: Credit score, active loans, default history, payment behavior
- **Auto-decisioning**: Configure score thresholds for auto-approval/rejection
- **Compliance**: Stores client consent before pulling credit report

### Notification Channels

- **SMS**: Integration with Twilio, AWS SNS, local SMS gateways
- **Email**: SMTP server configuration for transactional emails
- **Push notifications**: Mobile app integration via Firebase Cloud Messaging
- **Templates**: Configurable templates with placeholders
- **Delivery tracking**: All notifications logged with delivery status
