# Apache Fineract - Close of Business (COB) & Batch Processing

## 1. Close of Business (COB) Main Orchestration

```mermaid
flowchart TB
    Start([Scheduled Trigger<br/>End of Business Day]) --> CheckStatus{COB Already<br/>Running?}

    CheckStatus -->|Yes| Skip([Skip - Already Running])
    CheckStatus -->|No| LockCOB[Acquire Distributed Lock<br/>COB_LOCK]

    LockCOB --> InitCOB[Initialize COB Context<br/>Set Business Date<br/>Load Configuration]

    InitCOB --> PreCOB[Pre-COB Validation]
    PreCOB --> ValidateGL{GL Accounts<br/>Balanced?}
    ValidateGL -->|No| Alert1[Send Alert to Accountant]
    ValidateGL -->|Yes| StartPartitions

    Alert1 --> StartPartitions[Partition Loan Accounts<br/>by Office/Batch Size]

    StartPartitions --> ParallelCOB{Execute Parallel<br/>COB Workers}

    ParallelCOB --> Worker1[Worker 1:<br/>Office A Loans]
    ParallelCOB --> Worker2[Worker 2:<br/>Office B Loans]
    ParallelCOB --> WorkerN[Worker N:<br/>Office N Loans]

    Worker1 --> ProcessLoan1[Process Each Loan<br/>in Partition]
    Worker2 --> ProcessLoan2[Process Each Loan<br/>in Partition]
    WorkerN --> ProcessLoanN[Process Each Loan<br/>in Partition]

    ProcessLoan1 --> Sync1[Synchronization Point]
    ProcessLoan2 --> Sync1
    ProcessLoanN --> Sync1

    Sync1 --> PostCOB[Post-COB Processing]

    PostCOB --> AccrualPost[Interest Accrual<br/>Posting]
    AccrualPost --> ProvCalc[Provisioning<br/>Calculation]
    ProvCalc --> Aging[Portfolio Aging<br/>Update]
    Aging --> Reports[Generate COB<br/>Reports]
    Reports --> Notifications[Send Notifications<br/>to Stakeholders]

    Notifications --> AdvanceDate[Advance Business Date<br/>to Next Day]
    AdvanceDate --> ReleaseLock[Release Distributed Lock]
    ReleaseLock --> End([COB Complete])

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style ParallelCOB fill:#fff3e0
    style ValidateGL fill:#ffccbc
    style Alert1 fill:#ffcdd2
```

## 2. Individual Loan COB Processing (Per Account)

```mermaid
flowchart LR
    Start([Loan Account]) --> LoadLoan[Load Loan<br/>with Schedule]

    LoadLoan --> CheckActive{Loan<br/>Active?}
    CheckActive -->|No| Skip([Skip])
    CheckActive -->|Yes| CheckOverdue{Has Overdue<br/>Installments?}

    CheckOverdue -->|No| AccrueInt
    CheckOverdue -->|Yes| ApplyPenalty[Apply Late Payment<br/>Penalty Charges]

    ApplyPenalty --> AccrueInt[Calculate Daily<br/>Interest Accrual]

    AccrueInt --> UpdateDerived[Update Derived Fields:<br/>- Days Overdue<br/>- Arrears Amount<br/>- Outstanding Balance]

    UpdateDerived --> ClassifyNPA{Days Overdue<br/>&gt; NPA Days?}

    ClassifyNPA -->|Yes| MarkNPA[Mark as NPA<br/>Update Classification]
    ClassifyNPA -->|No| ClassifyDelinq

    MarkNPA --> ClassifyDelinq{Delinquency<br/>Classification}

    ClassifyDelinq --> UpdateBucket[Update Delinquency Bucket<br/>0-30, 31-60, 61-90, 90+]

    UpdateBucket --> RecalcProvision[Recalculate Provisioning<br/>Based on Classification]

    RecalcProvision --> EventPublish[Publish Business Events:<br/>- LoanAccrualProcessed<br/>- LoanDelinquencyChanged<br/>- LoanNPAMarked]

    EventPublish --> SaveState[Save Updated State<br/>to Database]

    SaveState --> End([Complete])

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style ClassifyNPA fill:#ffccbc
    style ClassifyDelinq fill:#fff3e0
```

## 3. Savings Account COB Processing

```mermaid
flowchart TB
    Start([Savings Account]) --> CheckActive{Account<br/>Active?}

    CheckActive -->|No| Skip([Skip])
    CheckActive -->|Yes| CheckMinBalance{Balance Below<br/>Minimum?}

    CheckMinBalance -->|Yes| ApplyMinBalFee[Apply Minimum<br/>Balance Fee]
    CheckMinBalance -->|No| CheckDormant

    ApplyMinBalFee --> CheckDormant{Days Inactive<br/>&gt; Dormancy Days?}

    CheckDormant -->|Yes| MarkDormant[Mark Account<br/>as Dormant]
    CheckDormant -->|No| CheckWithholdTax

    MarkDormant --> CheckWithholdTax{Withholding Tax<br/>Applicable?}

    CheckWithholdTax -->|Yes| CalcWithholdTax[Calculate Withholding Tax<br/>on Interest Earned]
    CheckWithholdTax -->|No| CalcInterest

    CalcWithholdTax --> CalcInterest[Calculate Daily<br/>Interest Accrual]

    CalcInterest --> CheckPostingFreq{Interest Posting<br/>Frequency Met?}

    CheckPostingFreq -->|Daily| PostInterest
    CheckPostingFreq -->|Monthly| CheckMonth
    CheckPostingFreq -->|Quarterly| CheckQuarter
    CheckPostingFreq -->|Annually| CheckYear

    CheckMonth{End of<br/>Month?}
    CheckQuarter{End of<br/>Quarter?}
    CheckYear{End of<br/>Year?}

    CheckMonth -->|Yes| PostInterest[Post Interest to Account<br/>Generate GL Entries]
    CheckMonth -->|No| SaveAccrual
    CheckQuarter -->|Yes| PostInterest
    CheckQuarter -->|No| SaveAccrual
    CheckYear -->|Yes| PostInterest
    CheckYear -->|No| SaveAccrual

    PostInterest --> DeductTax{Tax Amount<br/>&gt; 0?}

    DeductTax -->|Yes| PostTaxDeduction[Post Tax Deduction<br/>to Tax Authority GL]
    DeductTax -->|No| EventPublish

    PostTaxDeduction --> EventPublish[Publish Events:<br/>- InterestPosted<br/>- WithholdingTaxApplied]

    SaveAccrual[Save Accrual State] --> EventPublish

    EventPublish --> End([Complete])

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style CheckPostingFreq fill:#fff3e0
```

## 4. Batch Job Scheduler Architecture

```mermaid
graph TB
    subgraph "Job Scheduler (Quartz)"
        Scheduler[Quartz Scheduler<br/>Cron-based Triggers]
        JobStore[(Job Store<br/>Persistent)]
        TriggerStore[(Trigger Store<br/>Cron Expressions)]
    end

    subgraph "Fineract Batch Jobs"
        COB[Close of Business<br/>Daily 11:59 PM]
        InterestPost[Savings Interest Posting<br/>Based on Frequency]
        LoanRepayDue[Loan Repayment Due<br/>Daily Morning]
        Provisioning[Loan Provisioning<br/>Monthly]
        Reports[Scheduled Reports<br/>Various Frequencies]
        Notifications[SMS/Email Notifications<br/>Event-driven]
        Aging[Portfolio Aging<br/>Daily]
        FeesCharges[Apply Recurring Fees<br/>Monthly/Weekly]
        DataArchival[Archive Old Data<br/>Monthly]
        AccrualPosting[Accrual GL Posting<br/>Daily]
    end

    subgraph "Job Execution Context"
        TenantContext[Tenant Context<br/>Multi-tenant Isolation]
        JobParams[Job Parameters<br/>Configuration]
        RetryPolicy[Retry Policy<br/>3 attempts, backoff]
        ErrorHandler[Error Handler<br/>Email Alerts]
    end

    subgraph "Execution Infrastructure"
        ThreadPool[Thread Pool<br/>10 Workers]
        DBConnection[DB Connection Pool<br/>Per Tenant]
        LockManager[Distributed Lock<br/>Prevent Duplicate Runs]
    end

    subgraph "Monitoring & Logging"
        JobHistory[(Job Execution History)]
        AuditLog[(Audit Log)]
        AlertSystem[Alert System<br/>Failed Jobs]
    end

    Scheduler --> JobStore
    Scheduler --> TriggerStore

    Scheduler --> COB
    Scheduler --> InterestPost
    Scheduler --> LoanRepayDue
    Scheduler --> Provisioning
    Scheduler --> Reports
    Scheduler --> Notifications
    Scheduler --> Aging
    Scheduler --> FeesCharges
    Scheduler --> DataArchival
    Scheduler --> AccrualPosting

    COB --> TenantContext
    InterestPost --> TenantContext
    LoanRepayDue --> TenantContext
    Provisioning --> TenantContext

    TenantContext --> JobParams
    JobParams --> RetryPolicy
    RetryPolicy --> ErrorHandler

    TenantContext --> ThreadPool
    ThreadPool --> DBConnection
    DBConnection --> LockManager

    LockManager --> JobHistory
    ErrorHandler --> AuditLog
    ErrorHandler --> AlertSystem

    style Scheduler fill:#e1f5ff,stroke:#01579b,stroke-width:2px
    style COB fill:#ffccbc,stroke:#bf360c,stroke-width:2px
    style TenantContext fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style LockManager fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

## 5. COB Error Handling & Recovery

```mermaid
sequenceDiagram
    participant Scheduler
    participant COBJob
    participant LockManager
    participant AccountProcessor
    participant ErrorHandler
    participant Admin

    Scheduler->>COBJob: Trigger COB Job
    COBJob->>LockManager: Acquire Lock "COB_LOCK"

    alt Lock Already Held
        LockManager-->>COBJob: Lock Unavailable
        COBJob->>ErrorHandler: Log Warning
        ErrorHandler->>Admin: Alert: COB Already Running
        COBJob-->>Scheduler: Exit Gracefully
    else Lock Acquired
        LockManager-->>COBJob: Lock Granted
        COBJob->>AccountProcessor: Process Accounts in Batches

        loop Each Batch
            AccountProcessor->>AccountProcessor: Process 1000 Accounts

            alt Processing Error
                AccountProcessor->>ErrorHandler: Log Error with Account ID
                ErrorHandler->>ErrorHandler: Store Failed Account
                ErrorHandler->>Admin: Email Alert if Critical
                AccountProcessor->>AccountProcessor: Continue with Next Account
            else Processing Success
                AccountProcessor->>AccountProcessor: Mark Batch Complete
            end
        end

        AccountProcessor-->>COBJob: All Batches Complete
        COBJob->>COBJob: Generate Failure Report

        alt Has Failures
            COBJob->>ErrorHandler: Create Failure Summary
            ErrorHandler->>Admin: Email: COB Completed with N Failures
            COBJob->>COBJob: Mark COB as "Partial Success"
        else No Failures
            COBJob->>COBJob: Mark COB as "Success"
        end

        COBJob->>LockManager: Release Lock
        LockManager-->>COBJob: Lock Released
        COBJob-->>Scheduler: Job Complete
    end

    Admin->>COBJob: Retry Failed Accounts (Manual)
    COBJob->>AccountProcessor: Reprocess Failed Accounts
    AccountProcessor-->>COBJob: Reprocessing Complete
    COBJob->>Admin: Final Report
```

## 6. Business Date Management

```mermaid
stateDiagram-v2
    [*] --> CurrentBusinessDate

    CurrentBusinessDate: Business Date: 2025-11-18
    CurrentBusinessDate: System Date: 2025-11-18 14:30:00
    CurrentBusinessDate: COB Status: Not Run

    CurrentBusinessDate --> COBTriggered: Scheduled Trigger<br/>11:59 PM

    COBTriggered: COB Status: Running
    COBTriggered: Processing: Loans, Savings, GL

    COBTriggered --> COBValidation: Validation Check

    COBValidation: Verify All Accounts Processed
    COBValidation: Check GL Balance
    COBValidation: Generate Reports

    COBValidation --> COBFailed: Validation Failed
    COBValidation --> COBSuccess: All Checks Pass

    COBFailed: COB Status: Failed
    COBFailed: Business Date: 2025-11-18 (Unchanged)
    COBFailed: Manual intervention required

    COBFailed --> ManualFix: Admin Fixes Issues
    ManualFix --> COBTriggered: Retry COB

    COBSuccess: COB Status: Success
    COBSuccess: Advance Business Date

    COBSuccess --> NextBusinessDate: Date Advanced

    NextBusinessDate: Business Date: 2025-11-19
    NextBusinessDate: System Date: 2025-11-19 00:00:05
    NextBusinessDate: COB Status: Not Run
    NextBusinessDate: Ready for Operations

    NextBusinessDate --> [*]

    note right of CurrentBusinessDate
        All transactions during the day
        use Business Date 2025-11-18
        regardless of system time
    end note

    note right of COBSuccess
        Business date advances ONLY
        after successful COB completion
    end note
```

## 7. Parallel COB Processing with Partitioning

```mermaid
flowchart TB
    Start([COB Initiated]) --> LoadConfig[Load COB Configuration:<br/>- Partition Size<br/>- Thread Pool Size<br/>- Timeout Settings]

    LoadConfig --> QueryAccounts[Query All Active Accounts<br/>Requiring COB Processing]

    QueryAccounts --> PartitionStrategy{Partitioning<br/>Strategy}

    PartitionStrategy -->|By Office| OfficePartition[Partition by Office ID<br/>Each Office = 1 Partition]
    PartitionStrategy -->|By Size| SizePartition[Partition by Account Count<br/>1000 Accounts per Partition]
    PartitionStrategy -->|Hybrid| HybridPartition[Office + Size Hybrid<br/>Max 1000 per Office]

    OfficePartition --> CreatePartitions
    SizePartition --> CreatePartitions
    HybridPartition --> CreatePartitions

    CreatePartitions[Create N Partitions] --> SubmitToPool[Submit to Thread Pool<br/>Max 10 Concurrent Workers]

    SubmitToPool --> P1[Partition 1<br/>Worker Thread]
    SubmitToPool --> P2[Partition 2<br/>Worker Thread]
    SubmitToPool --> P3[Partition 3<br/>Worker Thread]
    SubmitToPool --> PN[Partition N<br/>Worker Thread]

    P1 --> Process1[Process Accounts<br/>in Partition 1]
    P2 --> Process2[Process Accounts<br/>in Partition 2]
    P3 --> Process3[Process Accounts<br/>in Partition 3]
    PN --> ProcessN[Process Accounts<br/>in Partition N]

    Process1 --> Result1[Store Results<br/>Success/Failure Count]
    Process2 --> Result2[Store Results<br/>Success/Failure Count]
    Process3 --> Result3[Store Results<br/>Success/Failure Count]
    ProcessN --> ResultN[Store Results<br/>Success/Failure Count]

    Result1 --> Barrier[Synchronization Barrier<br/>Wait for All Partitions]
    Result2 --> Barrier
    Result3 --> Barrier
    ResultN --> Barrier

    Barrier --> Aggregate[Aggregate Results:<br/>Total Processed: XXXXX<br/>Success: XXXXX<br/>Failed: XX]

    Aggregate --> CheckFailures{Any<br/>Failures?}

    CheckFailures -->|Yes| GenerateFailureReport[Generate Failure Report<br/>with Account IDs]
    CheckFailures -->|No| Success

    GenerateFailureReport --> AlertAdmin[Alert Admin via Email]
    AlertAdmin --> PartialSuccess[Mark COB as<br/>Partial Success]

    Success[Mark COB as Success] --> End([Complete])
    PartialSuccess --> End

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style Barrier fill:#fff3e0
    style CheckFailures fill:#ffccbc
```

## Key COB Processing Details

### COB Execution Sequence

1. **Pre-COB Phase**:
   - Acquire distributed lock to prevent concurrent COB runs
   - Validate GL accounts are balanced
   - Load configuration (partition size, parallelism level)
   - Query all active accounts requiring processing

2. **Parallel Processing Phase**:
   - Partition accounts by office or size
   - Submit partitions to thread pool (default 10 workers)
   - Each worker processes accounts independently
   - Per-account processing:
     - Load loan/savings with schedule
     - Calculate interest accrual
     - Apply penalties and fees
     - Update derived fields (days overdue, balances)
     - Classify delinquency/NPA status
     - Recalculate provisioning
     - Publish business events
   - Synchronization barrier waits for all partitions

3. **Post-COB Phase**:
   - Aggregate results from all partitions
   - Generate COB summary report
   - Post accrued interest to GL
   - Update provisioning entries
   - Refresh portfolio aging reports
   - Send notifications to stakeholders
   - Advance business date to next day
   - Release distributed lock

### Error Handling

- **Account-level errors**: Logged and reported, COB continues
- **Critical errors**: Entire COB is rolled back
- **Failed accounts**: Tracked in failure report for manual reprocessing
- **Lock timeout**: COB exits gracefully, alerts admin
- **GL imbalance**: COB is blocked until resolved

### Performance Optimizations

- **Partitioning**: Distributes load across multiple threads
- **Batch loading**: Loads accounts in batches to reduce DB queries
- **Derived fields**: Pre-calculated to avoid runtime computation
- **Event batching**: Business events are batched before publishing
- **Index usage**: Queries optimized with indexes on account status, office ID

### Business Date Management

- Business date is independent of system date
- Transactions always use current business date
- Business date advances ONLY after successful COB
- Failed COB keeps business date unchanged
- Manual date override available for admins (holidays, system downtime)

### Monitoring & Alerts

- **Job execution history**: Stored for auditing
- **Performance metrics**: Average processing time per account
- **Failure rate**: Percentage of failed accounts
- **Email alerts**: Sent for failures, partial success, or critical errors
- **Dashboard**: Real-time COB progress tracking
