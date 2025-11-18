# Apache Fineract - Maker-Checker Approval Workflow

## 1. Maker-Checker State Machine

```mermaid
stateDiagram-v2
    [*] --> Pending: Maker Submits<br/>Command

    Pending: Status: Awaiting Approval
    Pending: Maker: john.doe
    Pending: Action: CREATE_LOAN
    Pending: Can be approved by Checker

    Pending --> Approved: Checker Approves
    Pending --> Rejected: Checker Rejects
    Pending --> Deleted: Maker Withdraws

    Approved: Status: Approved
    Approved: Checker: jane.smith
    Approved: Auto-execute: Yes

    Approved --> Executed: System Executes<br/>Command

    Executed: Status: Complete
    Executed: Business Action Performed
    Executed: Audit Trail Recorded

    Rejected: Status: Rejected
    Rejected: Reason: Insufficient Documentation
    Rejected: Maker can resubmit

    Rejected --> [*]: Final State

    Deleted: Status: Withdrawn
    Deleted: Maker cancelled request
    Deleted: No action taken

    Deleted --> [*]: Final State

    Executed --> [*]: Final State

    note right of Pending
        Commands requiring approval:
        - Create/Update/Delete Clients
        - Loan approvals above limit
        - Disbursements
        - Write-offs
        - Product changes
        - GL account modifications
    end note

    note right of Approved
        Auto-execution happens
        immediately after approval
        in separate transaction
    end note
```

## 2. Loan Approval Sequence (Maker-Checker)

```mermaid
sequenceDiagram
    participant LoanOfficer as Loan Officer<br/>(Maker)
    participant API as REST API
    participant CommandRouter as Command Router
    participant AuditService as Audit Service
    participant DB as Database
    participant Manager as Branch Manager<br/>(Checker)
    participant LoanService as Loan Service
    participant EventBus as Event Bus

    LoanOfficer->>API: POST /loans/{id}/approve<br/>Amount: $10,000

    API->>API: Check Maker Permission<br/>"APPROVE_LOAN"

    alt Maker Has Direct Permission
        API->>LoanService: Approve Loan Directly
        LoanService->>DB: Update Loan Status = APPROVED
        LoanService->>EventBus: Publish LoanApproved Event
        LoanService-->>API: Success
        API-->>LoanOfficer: Loan Approved (No Checker Required)
    else Requires Checker Approval
        API->>CommandRouter: Create Approval Command
        CommandRouter->>DB: Insert into m_portfolio_command_source<br/>- maker_id<br/>- action: APPROVE_LOAN<br/>- entity_id: loan_id<br/>- command_json: approval_details<br/>- status: PENDING
        CommandRouter->>AuditService: Log Maker Action
        AuditService->>DB: Insert Audit Record
        CommandRouter-->>API: Command Created (Pending Approval)
        API-->>LoanOfficer: Request Submitted<br/>Awaiting Manager Approval

        Note over Manager: Manager Reviews<br/>Pending Approvals

        Manager->>API: GET /makercheckers<br/>Filter: Pending

        API->>DB: Query Pending Commands
        DB-->>API: List of Pending Commands
        API-->>Manager: Show Pending:<br/>Loan #12345 Approval

        Manager->>API: POST /makercheckers/{id}/approve<br/>Checker Decision

        API->>API: Check Checker Permission<br/>"CHECKER_APPROVE_LOAN"

        alt Checker is Same as Maker
            API-->>Manager: Error: Checker cannot be Maker
        else Valid Checker
            API->>CommandRouter: Process Checker Approval
            CommandRouter->>DB: Update m_portfolio_command_source<br/>- checker_id<br/>- status: APPROVED<br/>- approved_on_date

            CommandRouter->>AuditService: Log Checker Action
            AuditService->>DB: Insert Audit Record

            CommandRouter->>CommandRouter: Execute Command (Auto)
            CommandRouter->>LoanService: Execute Approve Loan
            LoanService->>DB: Update Loan Status = APPROVED
            LoanService->>EventBus: Publish LoanApproved Event
            EventBus->>EventBus: Notify Loan Officer via Email

            LoanService-->>CommandRouter: Execution Success
            CommandRouter->>DB: Update command_source<br/>status: EXECUTED

            CommandRouter-->>API: Success
            API-->>Manager: Approval Complete
        end
    end
```

## 3. Maker-Checker Configuration Matrix

```mermaid
graph TB
    subgraph "Permission Configuration"
        MakerPerm[Maker Permissions]
        CheckerPerm[Checker Permissions]
        DirectPerm[Direct Execute Permissions]
    end

    subgraph "Loan Operations"
        CreateLoan[Create Loan]
        ApproveLoan[Approve Loan]
        DisburseLoan[Disburse Loan]
        WriteOffLoan[Write-Off Loan]
        RepayLoan[Repayment]
    end

    subgraph "Client Operations"
        CreateClient[Create Client]
        UpdateClient[Update Client]
        ActivateClient[Activate Client]
        CloseClient[Close Client]
    end

    subgraph "Product Operations"
        CreateProduct[Create Loan Product]
        UpdateProduct[Update Product]
        DeleteProduct[Delete Product]
    end

    subgraph "Accounting Operations"
        CreateGL[Create GL Account]
        PostJournal[Post Manual Journal]
        CloseGL[Close GL Period]
    end

    MakerPerm --> CreateLoan
    MakerPerm --> CreateClient
    MakerPerm --> CreateProduct
    MakerPerm --> CreateGL

    CheckerPerm --> ApproveLoan
    CheckerPerm --> DisburseLoan
    CheckerPerm --> WriteOffLoan
    CheckerPerm --> ActivateClient
    CheckerPerm --> CloseClient
    CheckerPerm --> UpdateProduct
    CheckerPerm --> PostJournal

    DirectPerm --> RepayLoan

    CreateLoan -.->|Requires Checker| ApproveLoan
    ApproveLoan -.->|Requires Checker| DisburseLoan
    CreateClient -.->|Requires Checker| ActivateClient
    CreateProduct -.->|Requires Checker| UpdateProduct
    CreateGL -.->|Requires Checker| PostJournal

    style MakerPerm fill:#e3f2fd,stroke:#1976d2
    style CheckerPerm fill:#f3e5f5,stroke:#7b1fa2
    style DirectPerm fill:#e8f5e9,stroke:#388e3c
    style WriteOffLoan fill:#ffebee,stroke:#c62828
    style CloseGL fill:#ffebee,stroke:#c62828
```

## 4. Rejection Workflow

```mermaid
sequenceDiagram
    participant Maker
    participant API
    participant CommandRouter
    participant DB
    participant Checker
    participant NotificationSvc as Notification Service

    Maker->>API: POST /loans/{id}/approve
    API->>CommandRouter: Create Command
    CommandRouter->>DB: Insert Command (PENDING)
    CommandRouter-->>API: Command ID: 789
    API-->>Maker: Request Submitted

    Checker->>API: GET /makercheckers
    API->>DB: Query Pending
    DB-->>API: Command 789
    API-->>Checker: Show Pending Request

    Checker->>Checker: Reviews Loan Details

    alt Reject Decision
        Checker->>API: POST /makercheckers/789/reject<br/>Reason: "Missing income verification"

        API->>CommandRouter: Process Rejection
        CommandRouter->>DB: Update Command:<br/>- status: REJECTED<br/>- checker_id<br/>- rejection_reason

        CommandRouter->>NotificationSvc: Notify Maker
        NotificationSvc->>Maker: Email: Request Rejected<br/>Reason: Missing income verification

        CommandRouter-->>API: Rejection Recorded
        API-->>Checker: Rejection Complete

        Note over Maker: Maker can resubmit<br/>with corrections
    end
```

## 5. Maker Withdrawal Workflow

```mermaid
flowchart LR
    Start([Maker Submits Command]) --> Pending[Command Status:<br/>PENDING]

    Pending --> MakerReview{Maker Realizes<br/>Mistake?}

    MakerReview -->|No| WaitChecker[Wait for Checker]
    MakerReview -->|Yes| Withdraw

    Withdraw[Maker Withdraws Request] --> CheckStatus{Already<br/>Approved?}

    CheckStatus -->|Yes| CannotDelete[Error: Cannot Delete<br/>Already Approved]
    CheckStatus -->|No| DeleteCmd[DELETE /makercheckers/{id}]

    DeleteCmd --> UpdateDB[Update DB:<br/>status = DELETED]
    UpdateDB --> AuditLog[Log Withdrawal<br/>in Audit Trail]
    AuditLog --> NotifyChecker[Notify Checker:<br/>Request Withdrawn]
    NotifyChecker --> End([Command Deleted])

    WaitChecker --> CheckerAction{Checker<br/>Action}
    CheckerAction -->|Approve| Approved
    CheckerAction -->|Reject| Rejected

    Approved[Status: APPROVED] --> Execute[Auto-Execute]
    Execute --> Complete([Complete])

    Rejected[Status: REJECTED] --> Final([Final])

    CannotDelete --> ErrorEnd([Error])

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style Complete fill:#c8e6c9
    style Final fill:#ffccbc
    style ErrorEnd fill:#ffcdd2
```

## 6. Bulk Approval Workflow

```mermaid
sequenceDiagram
    participant Manager as Branch Manager<br/>(Checker)
    participant UI as Web UI
    participant API as REST API
    participant CommandRouter as Command Router
    participant DB as Database
    participant LoanService as Loan Service

    Manager->>UI: View Pending Approvals Dashboard
    UI->>API: GET /makercheckers?status=PENDING
    API->>DB: Query Pending Commands
    DB-->>API: Return 25 Pending Items
    API-->>UI: List of Pending Approvals

    UI-->>Manager: Display:<br/>- 10 Loan Approvals<br/>- 8 Loan Disbursements<br/>- 5 Client Activations<br/>- 2 Write-offs

    Manager->>Manager: Review Items

    Manager->>UI: Select Items to Approve<br/>(Checkbox Selection)
    UI->>Manager: Selected: 10 Items

    Manager->>UI: Click "Bulk Approve"

    loop For Each Selected Command
        UI->>API: POST /makercheckers/{id}/approve
        API->>CommandRouter: Process Approval

        CommandRouter->>DB: Update Command Status
        CommandRouter->>CommandRouter: Execute Command

        alt Loan Approval
            CommandRouter->>LoanService: Approve Loan
            LoanService->>DB: Update Loan
        else Loan Disbursement
            CommandRouter->>LoanService: Disburse Loan
            LoanService->>DB: Create Transactions
        else Client Activation
            CommandRouter->>LoanService: Activate Client
            LoanService->>DB: Update Client Status
        end

        CommandRouter-->>API: Success
        API-->>UI: Item {id} Approved
    end

    UI-->>Manager: Bulk Approval Complete<br/>10 of 10 Successful

    Note over Manager,DB: All approvals are atomic<br/>Each command is independent transaction
```

## 7. Maker-Checker Database Schema

```mermaid
erDiagram
    M_PORTFOLIO_COMMAND_SOURCE {
        BIGINT id PK
        VARCHAR action_name "CREATE_LOAN, APPROVE_LOAN, etc"
        VARCHAR entity_name "Loan, Client, SavingsAccount"
        BIGINT resource_id "Entity ID"
        BIGINT maker_id FK
        DATE made_on_date
        BIGINT checker_id FK
        DATE checked_on_date
        VARCHAR processing_result "SUCCESS, FAILURE"
        VARCHAR status "PENDING, APPROVED, REJECTED, DELETED"
        TEXT command_as_json "Complete command payload"
        TEXT rejection_reason
        BIGINT office_id FK
    }

    M_APPUSER {
        BIGINT id PK
        VARCHAR username
        VARCHAR firstname
        VARCHAR lastname
        VARCHAR email
        TINYINT is_deleted
    }

    M_ROLE {
        BIGINT id PK
        VARCHAR name
        VARCHAR description
    }

    M_ROLE_PERMISSION {
        BIGINT id PK
        BIGINT role_id FK
        VARCHAR permission_code "APPROVE_LOAN_MAKER, APPROVE_LOAN_CHECKER"
    }

    M_APPUSER_ROLE {
        BIGINT appuser_id FK
        BIGINT role_id FK
    }

    M_AUDIT {
        BIGINT id PK
        VARCHAR action_name
        VARCHAR entity_name
        BIGINT resource_id
        BIGINT maker_id FK
        DATE made_on_date
        VARCHAR checker_id
        DATE checked_on_date
        VARCHAR processing_result
        TEXT command_as_json
        BIGINT office_id FK
    }

    M_PORTFOLIO_COMMAND_SOURCE }o--|| M_APPUSER : "maker"
    M_PORTFOLIO_COMMAND_SOURCE }o--|| M_APPUSER : "checker"
    M_APPUSER_ROLE }o--|| M_APPUSER : "user"
    M_APPUSER_ROLE }o--|| M_ROLE : "role"
    M_ROLE_PERMISSION }o--|| M_ROLE : "role"
    M_AUDIT }o--|| M_APPUSER : "maker"
```

## 8. Permission-Based Routing

```mermaid
flowchart TB
    Start([User Action:<br/>POST /loans/{id}/approve]) --> ExtractUser[Extract User from<br/>Security Context]

    ExtractUser --> LoadPerms[Load User Permissions<br/>from Roles]

    LoadPerms --> CheckDirect{Has Direct<br/>Permission?<br/>APPROVE_LOAN}

    CheckDirect -->|Yes| DirectExec[Execute Directly<br/>No Maker-Checker]

    CheckDirect -->|No| CheckMaker{Has Maker<br/>Permission?<br/>APPROVE_LOAN_MAKER}

    CheckMaker -->|Yes| CreateCommand[Create Pending Command<br/>Route to Checker Queue]

    CheckMaker -->|No| Unauthorized[HTTP 403<br/>Unauthorized]

    CreateCommand --> NotifyChecker[Notify Checker via:<br/>- Email<br/>- Dashboard Alert<br/>- Mobile Push]

    NotifyChecker --> WaitApproval[Wait for Checker<br/>Approval]

    WaitApproval --> CheckerReview{Checker<br/>Reviews}

    CheckerReview -->|Approve| CheckSameUser{Checker ==<br/>Maker?}
    CheckerReview -->|Reject| Rejected[Mark REJECTED<br/>Notify Maker]

    CheckSameUser -->|Yes| ErrorSelf[Error: Self-Approval<br/>Not Allowed]
    CheckSameUser -->|No| AutoExec[Auto-Execute Command]

    AutoExec --> Success[Command Executed<br/>Business Action Complete]

    DirectExec --> Success
    Rejected --> End([End])
    ErrorSelf --> End
    Unauthorized --> End
    Success --> End

    style Start fill:#e1f5ff
    style Success fill:#c8e6c9
    style Unauthorized fill:#ffcdd2
    style ErrorSelf fill:#ffcdd2
    style Rejected fill:#ffccbc
```

## Key Maker-Checker Concepts

### Permission Model

Fineract uses a dual permission model for maker-checker operations:

1. **Direct Execute Permission**: User can execute action immediately without approval
   - Example: `APPROVE_LOAN`
   - Typically for senior staff or automated processes

2. **Maker Permission**: User can initiate action, requires checker approval
   - Example: `APPROVE_LOAN_MAKER`
   - Typically for loan officers, junior staff

3. **Checker Permission**: User can approve actions initiated by makers
   - Example: `APPROVE_LOAN_CHECKER`
   - Typically for managers, senior staff

### Approval Rules

- **Self-approval prevention**: A user cannot approve their own request
- **Office-level scoping**: Checkers can only approve requests from their office hierarchy
- **Role-based**: Both maker and checker must have appropriate role permissions
- **Audit trail**: All maker-checker actions are logged in `m_audit` table

### Command Storage

All pending maker-checker commands are stored in `m_portfolio_command_source` table:
- **command_as_json**: Complete JSON payload of the original request
- **action_name**: Type of action (CREATE_LOAN, APPROVE_LOAN, DISBURSE_LOAN, etc.)
- **entity_name**: Entity type (Loan, Client, SavingsAccount, etc.)
- **resource_id**: ID of the specific entity
- **maker_id**: User who created the request
- **checker_id**: User who approved/rejected (NULL if pending)
- **status**: PENDING, APPROVED, REJECTED, DELETED

### Auto-Execution

After checker approval, commands are automatically executed:
1. Command status updated to APPROVED
2. Original command JSON is deserialized
3. Business service method is invoked
4. Database transaction is committed
5. Events are published
6. Command status updated to EXECUTED

### Use Cases Requiring Maker-Checker

**High-risk operations**:
- Loan write-offs
- Large disbursements (above threshold)
- Client data modifications
- Product configuration changes
- GL account modifications
- Manual journal entries

**Configurable by permission**:
- Organization can decide which operations require maker-checker
- Permissions are assigned to roles
- Users get permissions through role assignments

### Rejection Workflow

When checker rejects:
1. Command status → REJECTED
2. Rejection reason stored
3. Maker notified via email
4. Original request remains in history (audit trail)
5. Maker can submit new request with corrections

### Dashboard & Reporting

- **Maker Dashboard**: Shows status of submitted requests
- **Checker Dashboard**: Shows pending approvals by office
- **Audit Reports**: Complete history of maker-checker actions
- **Aging Reports**: How long requests have been pending
- **Performance Metrics**: Approval rates, average approval time
