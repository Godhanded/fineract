# Client & Group Management - Complete Workflow

## 1. Client Lifecycle State Diagram

```mermaid
stateDiagram-v2
    [*] --> PENDING: Register Client

    PENDING --> ACTIVE: Activate
    PENDING --> REJECTED: Reject Application
    PENDING --> WITHDRAWN: Client Withdraws

    ACTIVE --> CLOSED: Close Client
    ACTIVE --> TRANSFER_IN_PROGRESS: Initiate Office Transfer
    ACTIVE --> TRANSFER_ON_HOLD: Transfer Disputed

    TRANSFER_IN_PROGRESS --> ACTIVE: Complete Transfer
    TRANSFER_IN_PROGRESS --> TRANSFER_ON_HOLD: Reject Transfer
    TRANSFER_ON_HOLD --> ACTIVE: Resolve & Accept
    TRANSFER_ON_HOLD --> ACTIVE: Cancel Transfer

    CLOSED --> PENDING: Reactivate

    REJECTED --> PENDING: Undo Rejection
    WITHDRAWN --> PENDING: Undo Withdrawal

    CLOSED --> [*]
    REJECTED --> [*]
    WITHDRAWN --> [*]

    note right of PENDING
        Status: 100
        KYC Collection
        Document Verification
    end note

    note right of ACTIVE
        Status: 300
        Can Open Accounts
        Can Apply for Loans
    end note

    note right of CLOSED
        Status: 600
        All Accounts Settled
        No Active Products
    end note

    note right of TRANSFER_IN_PROGRESS
        Status: 303
        Moving Between Branches
        Temporary State
    end note
```

## 2. Client Onboarding Workflow

```mermaid
sequenceDiagram
    actor Client as New Client
    actor LO as Loan Officer
    participant API as REST API
    participant Service as Client Service
    participant Validation as Data Validator
    participant Domain as Client Domain
    participant DB as Database
    participant Events as Event Bus
    participant ExtSys as External Systems

    %% Step 1: Registration
    rect rgb(230, 240, 255)
        Note over Client,ExtSys: STEP 1: CLIENT REGISTRATION
        Client->>LO: Provide Personal Info<br/>ID Documents, Photo
        LO->>API: POST /clients<br/>{firstname, lastname, dob,<br/>mobile, office, photo}
        API->>Validation: validateClientData()

        Validation->>Validation: checkMandatoryFields()
        Validation->>Validation: validateMobileUnique()
        Validation->>Validation: validateEmailUnique()
        Validation->>Validation: validateDateOfBirth()

        alt Validation Failed
            Validation-->>API: ValidationError
            API-->>LO: Error: Invalid Data
        else Validation Passed
            Validation-->>Service: Valid
            Service->>Domain: new Client(PENDING)

            Domain->>Domain: generateAccountNumber()
            Domain->>Domain: generateDisplayName()

            Domain->>DB: INSERT m_client<br/>(status=100, account_no)
            Service->>Events: ClientCreatedBusinessEvent

            Service-->>API: {clientId: 456}
            API-->>LO: Client #456 Created
        end
    end

    %% Step 2: KYC & Document Upload
    rect rgb(240, 250, 240)
        Note over Client,ExtSys: STEP 2: KYC DOCUMENTS
        LO->>API: POST /clients/456/identifiers<br/>{documentType: PASSPORT,<br/>documentKey: AB123456}
        API->>Service: addIdentifier(clientId, data)

        Service->>DB: INSERT m_client_identifier<br/>(client_id, doc_type, doc_key)
        Service-->>API: Identifier Added

        LO->>API: POST /clients/456/images<br/>(Upload Photo)
        API->>Service: uploadImage(clientId, file)
        Service->>DB: INSERT m_image<br/>(client_id, image_blob)
        Service->>DB: UPDATE m_client<br/>(image_id)
        Service-->>API: Photo Uploaded

        LO->>API: POST /clients/456/addresses<br/>{addressType, line1, city, state}
        API->>Service: addAddress(clientId, data)
        Service->>DB: INSERT m_client_address
        Service-->>API: Address Added
    end

    %% Step 3: Credit Bureau Check (Optional)
    rect rgb(255, 250, 235)
        Note over Client,ExtSys: STEP 3: CREDIT CHECK
        LO->>API: GET /clients/456/creditcheck
        API->>ExtSys: queryCreditBureau(client_data)
        ExtSys-->>API: {credit_score: 720,<br/>defaults: 0}
        API->>Service: storeCreditData()
        Service->>DB: INSERT credit_bureau_data
        Service-->>API: Credit Check Complete
        API-->>LO: Score: 720 - Good
    end

    %% Step 4: Activation
    rect rgb(240, 255, 240)
        Note over Client,ExtSys: STEP 4: CLIENT ACTIVATION
        LO->>API: POST /clients/456?command=activate<br/>{activationDate}
        API->>Service: activateClient(clientId, date)

        Service->>Domain: activate(activationDate)
        Domain->>Domain: validateActivation()

        Domain->>DB: UPDATE m_client<br/>(status=300,<br/>activation_date,<br/>office_joining_date)

        Service->>Events: ClientActivatedBusinessEvent

        %% External Integrations
        Events->>ExtSys: Send to CRM<br/>(New Active Client)
        Events->>ExtSys: SMS: Welcome to Bank<br/>Account #456 Active

        Service-->>API: Client Activated
        API-->>LO: Client #456 Active
        LO->>Client: Welcome! You can now<br/>open accounts & apply for loans
    end
```

## 3. Client-to-Accounts Relationship

```mermaid
graph TB
    Client[Client #456<br/>John Doe<br/>Status: ACTIVE]

    subgraph "Loan Accounts"
        Loan1[Loan #1001<br/>Personal Loan<br/>Amount: $50,000<br/>Status: ACTIVE]
        Loan2[Loan #1005<br/>Auto Loan<br/>Amount: $25,000<br/>Status: ACTIVE]
        Loan3[Loan #1012<br/>Business Loan<br/>Amount: $100,000<br/>Status: CLOSED]
    end

    subgraph "Savings Accounts"
        Savings1[Savings #2001<br/>Regular Savings<br/>Balance: $5,000<br/>Status: ACTIVE]
        Savings2[Savings #2005<br/>Fixed Deposit<br/>Amount: $50,000<br/>Maturity: 2025-06-15]
        Savings3[Savings #2010<br/>Recurring Deposit<br/>Monthly: $1,000<br/>Status: ACTIVE]
    end

    subgraph "Share Accounts"
        Share1[Share #3001<br/>100 Shares<br/>Value: $10,000<br/>Status: ACTIVE]
    end

    Client -->|Borrower| Loan1
    Client -->|Borrower| Loan2
    Client -->|Borrower| Loan3

    Client -->|Owner| Savings1
    Client -->|Owner| Savings2
    Client -->|Owner| Savings3

    Client -->|Shareholder| Share1

    %% Linked Accounts
    Loan1 -.->|Guarantee| Savings2
    Loan2 -.->|Auto-debit| Savings1

    style Client fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Loan1 fill:#fff3e0,stroke:#e65100
    style Loan2 fill:#fff3e0,stroke:#e65100
    style Loan3 fill:#efebe9,stroke:#3e2723
    style Savings1 fill:#e8f5e9,stroke:#1b5e20
    style Savings2 fill:#e8f5e9,stroke:#1b5e20
    style Savings3 fill:#e8f5e9,stroke:#1b5e20
    style Share1 fill:#f3e5f5,stroke:#4a148c
```

## 4. Group Hierarchy & Structure

```mermaid
graph TD
    subgraph "Office Hierarchy"
        HeadOffice[Head Office<br/>Mumbai]
        HeadOffice --> Branch1[Branch A<br/>Pune]
        HeadOffice --> Branch2[Branch B<br/>Bangalore]
        Branch1 --> SubBranch[Sub-branch A1<br/>Kothrud]
    end

    subgraph "Group Hierarchy Level 1: Centers"
        Center1[Center: Shivaji Nagar<br/>Level: Center<br/>Staff: Officer 1]
        Center2[Center: Koregaon Park<br/>Level: Center<br/>Staff: Officer 2]
    end

    subgraph "Group Hierarchy Level 2: Groups"
        Group1A[Group: Mahila Mandal A<br/>Level: Group<br/>Members: 15]
        Group1B[Group: Youth SHG<br/>Level: Group<br/>Members: 12]
        Group2A[Group: Farmers Group<br/>Level: Group<br/>Members: 20]
    end

    subgraph "Individual Clients"
        Client1[Client: Priya Sharma<br/>ID: 001]
        Client2[Client: Rahul Patel<br/>ID: 002]
        Client3[Client: Anita Desai<br/>ID: 003]
        Client4[Client: Suresh Kumar<br/>ID: 004]
    end

    Branch1 --> Center1
    Branch1 --> Center2

    Center1 --> Group1A
    Center1 --> Group1B
    Center2 --> Group2A

    Group1A --> Client1
    Group1A --> Client2
    Group1B --> Client3
    Group2A --> Client4

    style HeadOffice fill:#e3f2fd,stroke:#01579b
    style Branch1 fill:#e1f5ff,stroke:#0277bd
    style Center1 fill:#fff9c4,stroke:#f57f17
    style Center2 fill:#fff9c4,stroke:#f57f17
    style Group1A fill:#c8e6c9,stroke:#388e3c
    style Group1B fill:#c8e6c9,stroke:#388e3c
    style Group2A fill:#c8e6c9,stroke:#388e3c
    style Client1 fill:#f3e5f5,stroke:#7b1fa2
    style Client2 fill:#f3e5f5,stroke:#7b1fa2
    style Client3 fill:#f3e5f5,stroke:#7b1fa2
    style Client4 fill:#f3e5f5,stroke:#7b1fa2
```

## 5. Group Lending Workflow

```mermaid
sequenceDiagram
    actor GroupLeader as Group Leader
    actor Members as Group Members
    actor LO as Loan Officer
    participant API as REST API
    participant Service as Loan Service
    participant Group as Group Domain
    participant DB as Database
    participant Events as Event Bus

    %% Group Meeting
    rect rgb(230, 245, 255)
        Note over GroupLeader,Events: WEEKLY GROUP MEETING
        GroupLeader->>Members: Call Weekly Meeting
        Members->>GroupLeader: Discuss Loan Needs

        GroupLeader->>LO: Request Group Loan<br/>Amount: $10,000 (for 10 members)
        LO->>API: POST /loans<br/>{groupId: 100,<br/>loanType: GROUP,<br/>principal: 10000}
        API->>Service: createGroupLoan(command)

        Service->>Group: validateGroupLoan()
        Group->>Group: checkAllMembersActive()
        Group->>Group: checkNoDefaulters()

        Service->>DB: INSERT m_loan<br/>(group_id, loan_type=GROUP)
        Service->>Events: GroupLoanCreatedBusinessEvent

        Service-->>API: Loan Created
        API-->>LO: Group Loan #2001
    end

    %% Approval & Disbursement
    rect rgb(240, 255, 240)
        Note over GroupLeader,Events: APPROVAL & DISBURSEMENT
        LO->>API: POST /loans/2001?command=approve
        API->>Service: approveLoan()
        Service->>DB: UPDATE loan (status=APPROVED)

        LO->>API: POST /loans/2001/transactions?command=disburse<br/>{amount: 10000}
        API->>Service: disburseGroupLoan()

        Service->>DB: INSERT loan_transaction<br/>(type=DISBURSEMENT)

        alt GLIM (Group Loan Individual Monitoring)
            Service->>DB: INSERT individual_member_loans<br/>(member1: 1000,<br/>member2: 800, ...)
            Note over Service: Track individual shares
        end

        Service->>Events: LoanDisbursedBusinessEvent
        Service-->>API: Disbursed
        API-->>LO: Funds Released
        LO->>GroupLeader: Distribute to Members
        GroupLeader->>Members: Individual Disbursement
    end

    %% Group Repayment
    rect rgb(255, 250, 235)
        Note over GroupLeader,Events: WEEKLY REPAYMENT
        Members->>GroupLeader: Collect Individual EMIs
        GroupLeader->>LO: Group Payment: $500

        LO->>API: POST /loans/2001/transactions?command=repayment<br/>{amount: 500}
        API->>Service: recordGroupRepayment()

        Service->>DB: INSERT loan_transaction<br/>(type=REPAYMENT, amount=500)

        alt GLIM Allocation
            Service->>Service: allocateToMembers()<br/>(member1: 50,<br/>member2: 40, ...)
            Service->>DB: UPDATE individual_allocations
        end

        Service->>DB: UPDATE loan<br/>(balance -= 500)
        Service->>Events: RepaymentBusinessEvent

        Service-->>API: Payment Recorded
        API-->>LO: Recorded
        LO->>GroupLeader: Receipt Issued
    end

    %% Default Handling
    rect rgb(255, 235, 235)
        Note over GroupLeader,Events: DEFAULT SCENARIO
        alt Member Defaults
            Members->>GroupLeader: Member X Cannot Pay

            GroupLeader->>Members: Group Guarantees<br/>Collect from Others

            alt Guarantee Honored
                Members->>GroupLeader: Cover Shortfall
                GroupLeader->>LO: Full Payment
                Note over Group: Group Solidarity Works
            else Guarantee Failed
                GroupLeader->>LO: Partial Payment Only
                LO->>API: Mark Loan Delinquent
                Service->>DB: UPDATE loan<br/>(delinquency_status)
                Service->>Events: DelinquencyBusinessEvent
                Note over Group: Group Loan at Risk
            end
        end
    end
```

## 6. Client Transfer Between Offices

```mermaid
flowchart TD
    Start([Client Transfer Request]) --> Initiate[Loan Officer Initiates Transfer<br/>Client #456 from Mumbai to Pune]

    Initiate --> ValidateTransfer{Validate<br/>Transfer?}

    ValidateTransfer -->|Invalid| RejectTransfer[Reject Transfer<br/>Reason: Active Loans/Pending]
    ValidateTransfer -->|Valid| CreateRequest[Create Transfer Request<br/>POST /clients/456?command=proposeTransfer]

    RejectTransfer --> End([End - Rejected])

    CreateRequest --> UpdateStatus[Update Client Status<br/>ACTIVE → TRANSFER_IN_PROGRESS]

    UpdateStatus --> NotifyTarget[Notify Target Office<br/>Pune Branch]

    NotifyTarget --> TargetReview{Target Office<br/>Reviews}

    TargetReview -->|Reject| RejectByTarget[Reject Transfer<br/>POST ?command=rejectTransfer<br/>Status → TRANSFER_ON_HOLD]
    TargetReview -->|Accept| AcceptTransfer[Accept Transfer<br/>POST ?command=acceptTransfer]

    RejectByTarget --> Dispute[Dispute Resolution<br/>Between Offices]
    Dispute -->|Resolve| AcceptTransfer
    Dispute -->|Cancel| CancelTransfer[Withdraw Transfer<br/>POST ?command=withdrawTransfer<br/>Status → ACTIVE in Mumbai]

    AcceptTransfer --> TransferData[Transfer Client Data<br/>1. Update office_id<br/>2. Transfer loan officers<br/>3. Move account records]

    TransferData --> UpdateGL[Update Accounting<br/>Transfer GL balances<br/>between office codes]

    UpdateGL --> FinalizeTransfer[Finalize Transfer<br/>Status: ACTIVE in Pune<br/>Clear transfer fields]

    FinalizeTransfer --> NotifyAll[Notify All Parties<br/>1. Client<br/>2. Mumbai Office<br/>3. Pune Office]

    NotifyAll --> AuditLog[Create Audit Log<br/>Record transfer history]

    AuditLog --> EndSuccess([End - Transfer Complete])
    CancelTransfer --> EndCancel([End - Transfer Cancelled])

    style CreateRequest fill:#e3f2fd
    style AcceptTransfer fill:#c8e6c9
    style RejectByTarget fill:#ffccbc
    style FinalizeTransfer fill:#e8f5e9
    style RejectTransfer fill:#ffcdd2
```

## 7. Client Charges & Transactions

```mermaid
sequenceDiagram
    participant Client as Client
    participant LO as Loan Officer
    participant API as REST API
    participant Service as Client Service
    participant DB as Database
    participant GL as GL Service
    participant Events as Event Bus

    %% Add Client Charge
    rect rgb(230, 240, 255)
        Note over Client,Events: ADD CLIENT CHARGE
        LO->>API: POST /clients/456/charges<br/>{chargeId: 10,<br/>amount: 100,<br/>dueDate: 2024-12-31}
        API->>Service: addClientCharge(command)

        Service->>DB: INSERT m_client_charge<br/>(client_id, charge_id,<br/>amount, outstanding=100)

        Service->>Events: ClientChargeAddedBusinessEvent
        Service-->>API: Charge Added
        API-->>LO: Charge #789 Created
    end

    %% Pay Client Charge
    rect rgb(240, 255, 240)
        Note over Client,Events: PAY CHARGE
        Client->>LO: Make Payment: $100
        LO->>API: POST /clients/456/charges/789?command=paycharge<br/>{amount: 100,<br/>paymentDate: 2024-11-17}
        API->>Service: payClientCharge(command)

        Service->>DB: INSERT m_client_transaction<br/>(type=PAY_CHARGE,<br/>amount=100)

        Service->>DB: INSERT m_client_charge_paid_by<br/>(transaction_id,<br/>charge_id, amount=100)

        Service->>DB: UPDATE m_client_charge<br/>(amount_paid=100,<br/>outstanding=0,<br/>is_paid=true)

        Service->>GL: createJournalEntry()
        GL->>DB: INSERT journal_entry<br/>DR: Cash $100<br/>CR: Fee Income $100

        Service->>Events: ClientChargePaidBusinessEvent

        Service-->>API: Payment Recorded
        API-->>LO: Charge Paid
        LO->>Client: Receipt: $100
    end

    %% Waive Charge
    rect rgb(255, 250, 235)
        Note over Client,Events: WAIVE CHARGE
        LO->>API: POST /clients/456/charges/789?command=waive
        API->>Service: waiveClientCharge(command)

        Service->>DB: INSERT m_client_transaction<br/>(type=WAIVE_CHARGE,<br/>amount=100)

        Service->>DB: UPDATE m_client_charge<br/>(amount_waived=100,<br/>outstanding=0,<br/>waived=true)

        Service->>GL: createJournalEntry()
        GL->>DB: INSERT journal_entry<br/>DR: Fee Waived (Expense) $100<br/>CR: Fee Income $100

        Service->>Events: ClientChargeWaivedBusinessEvent

        Service-->>API: Charge Waived
        API-->>LO: Waiver Applied
    end
```

## 8. Family Members & Additional Data

```mermaid
graph TB
    Client[Client: Priya Sharma<br/>ID: 456<br/>DOB: 1985-06-15]

    subgraph "Family Members"
        Spouse[Spouse: Rajesh Sharma<br/>Age: 40<br/>Occupation: Teacher<br/>Dependent: No]
        Child1[Child: Aarav Sharma<br/>Age: 12<br/>Student<br/>Dependent: Yes]
        Child2[Child: Anaya Sharma<br/>Age: 8<br/>Student<br/>Dependent: Yes]
        Mother[Mother: Sunita Patel<br/>Age: 65<br/>Retired<br/>Dependent: Yes]
    end

    subgraph "Identifiers (KYC)"
        Passport[Passport<br/>AB1234567<br/>Valid: 2030-05-20]
        Aadhaar[Aadhaar Card<br/>1234 5678 9012<br/>Status: ACTIVE]
        PAN[PAN Card<br/>ABCDE1234F<br/>Verified: Yes]
        Voter[Voter ID<br/>XYZ1234567<br/>Status: ACTIVE]
    end

    subgraph "Addresses"
        Home[Home Address<br/>Type: PERMANENT<br/>123, MG Road, Pune<br/>PIN: 411001]
        Work[Work Address<br/>Type: BUSINESS<br/>456, Tech Park, Pune<br/>PIN: 411014]
        Corr[Correspondence<br/>Type: MAILING<br/>Same as Home]
    end

    subgraph "Documents"
        Photo[Client Photo<br/>Image ID: 789<br/>Upload: 2024-01-15]
        Income[Income Certificate<br/>Annual: ₹6,00,000<br/>Employer: ABC Corp]
        Property[Property Documents<br/>Plot: 500 sq ft<br/>Value: ₹50,00,000]
    end

    Client --> Spouse
    Client --> Child1
    Client --> Child2
    Client --> Mother

    Client --> Passport
    Client --> Aadhaar
    Client --> PAN
    Client --> Voter

    Client --> Home
    Client --> Work
    Client --> Corr

    Client --> Photo
    Client --> Income
    Client --> Property

    style Client fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Spouse fill:#c8e6c9
    style Child1 fill:#fff9c4
    style Child2 fill:#fff9c4
    style Mother fill:#ffccbc
```

## Client Status Summary

| Status | Code | Description | Allowed Actions |
|--------|------|-------------|-----------------|
| PENDING | 100 | Application submitted | Activate, Reject, Withdraw |
| ACTIVE | 300 | Fully operational | Open accounts, Apply loans, Transfer |
| TRANSFER_IN_PROGRESS | 303 | Being transferred | Accept, Reject transfer |
| TRANSFER_ON_HOLD | 304 | Transfer disputed | Resolve, Cancel |
| CLOSED | 600 | No active products | Reactivate |
| REJECTED | 700 | Application rejected | Undo rejection |
| WITHDRAWN | 800 | Client withdrew | Undo withdrawal |
