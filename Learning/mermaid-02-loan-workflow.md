# Loan Management - Complete Workflow

## 1. Loan Lifecycle State Diagram

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED: Create Application

    SUBMITTED --> APPROVED: Approve
    SUBMITTED --> REJECTED: Reject
    SUBMITTED --> WITHDRAWN: Client Withdraws

    APPROVED --> ACTIVE: Disburse
    APPROVED --> SUBMITTED: Undo Approval

    ACTIVE --> CLOSED_OBLIGATIONS_MET: Full Repayment
    ACTIVE --> CLOSED_WRITTEN_OFF: Write Off
    ACTIVE --> OVERPAID: Overpayment
    ACTIVE --> CLOSED_RESCHEDULE: Reschedule
    ACTIVE --> TRANSFER_IN_PROGRESS: Initiate Transfer

    TRANSFER_IN_PROGRESS --> ACTIVE: Accept Transfer
    TRANSFER_IN_PROGRESS --> TRANSFER_ON_HOLD: Reject Transfer
    TRANSFER_ON_HOLD --> ACTIVE: Resolve

    OVERPAID --> CLOSED_OBLIGATIONS_MET: Refund Overpayment
    OVERPAID --> ACTIVE: Additional Charges

    CLOSED_OBLIGATIONS_MET --> [*]
    CLOSED_WRITTEN_OFF --> [*]
    CLOSED_RESCHEDULE --> [*]
    REJECTED --> [*]
    WITHDRAWN --> [*]

    note right of SUBMITTED
        Status: 100
        Can Edit Application
    end note

    note right of APPROVED
        Status: 200
        Schedule Generated
    end note

    note right of ACTIVE
        Status: 300
        Accepting Payments
    end note

    note right of CLOSED_OBLIGATIONS_MET
        Status: 600
        Loan Complete
    end note
```

## 2. Loan Application to Disbursement Sequence

```mermaid
sequenceDiagram
    actor LO as Loan Officer
    actor Client as Client
    actor Manager as Branch Manager
    participant API as REST API
    participant Service as Loan Service
    participant Domain as Loan Domain
    participant DB as Database
    participant Events as Event Bus
    participant Accounting as GL Service

    %% Application Submission
    rect rgb(200, 230, 250)
        Note over LO,Accounting: STEP 1: LOAN APPLICATION
        Client->>LO: Request Loan
        LO->>API: POST /loans<br/>{client, product, amount, term}
        API->>Service: createLoan(command)
        Service->>Domain: new Loan(SUBMITTED)
        Domain->>DB: INSERT loan (status=100)
        Service->>Events: LoanCreatedBusinessEvent
        DB-->>Service: loan_id: 123
        Service-->>API: {loanId: 123}
        API-->>LO: Application Created
        LO->>Client: Application #123 Submitted
    end

    %% Maker-Checker Approval
    rect rgb(255, 250, 200)
        Note over LO,Accounting: STEP 2: MAKER-CHECKER APPROVAL
        LO->>API: POST /loans/123?command=approve<br/>(MAKER)
        API->>Service: approveLoan(command)
        Service->>DB: INSERT m_audit_log<br/>(status=PENDING, maker_id)
        Service-->>API: Command Queued for Approval
        API-->>LO: Pending Manager Approval

        Manager->>API: GET /makercheckers<br/>(pending approvals)
        API-->>Manager: [Loan #123 approval pending]
        Manager->>API: POST /makercheckers/456?command=approve<br/>(CHECKER)
        API->>Service: executeApproval(auditId)
        Service->>Domain: approve(approvalDate, amount)

        Domain->>Domain: generateRepaymentSchedule()
        Note over Domain: Calculate EMI, Interest,<br/>Principal breakdown

        Domain->>DB: UPDATE loan (status=200)
        Domain->>DB: INSERT loan_repayment_schedule<br/>(36 installments)
        Service->>Events: LoanApprovedBusinessEvent
        Service->>DB: UPDATE m_audit_log<br/>(status=APPROVED, checker_id)

        Service-->>API: Approval Complete
        API-->>Manager: Loan Approved
        Manager->>LO: Loan #123 Approved
    end

    %% Disbursement
    rect rgb(200, 250, 200)
        Note over LO,Accounting: STEP 3: LOAN DISBURSEMENT
        LO->>API: POST /loans/123/transactions?command=disburse<br/>{date, amount, paymentType}
        API->>Service: disburseLoan(command)
        Service->>Domain: disburse(date, amount)

        Domain->>DB: INSERT loan_transaction<br/>(type=DISBURSEMENT, amount)
        Domain->>DB: UPDATE loan<br/>(status=300, principal_disbursed)

        Service->>Accounting: createJournalEntries(disbursement)
        Accounting->>DB: INSERT journal_entry<br/>DR: Loan Portfolio<br/>CR: Fund Source

        Service->>Events: LoanDisbursedBusinessEvent
        Events-->>LO: SMS: Loan Disbursed
        Events-->>Client: SMS: Amount Credited

        Service-->>API: Disbursement Complete
        API-->>LO: Loan #123 Disbursed
        LO->>Client: Funds Transferred
    end
```

## 3. Loan Repayment Processing Flow

```mermaid
flowchart TD
    Start([Customer Makes Payment]) --> Receive[Receive Payment<br/>Amount: $1,662.76]
    Receive --> CreateTxn[Create LoanTransaction<br/>type: REPAYMENT]

    CreateTxn --> LoadSchedule[Load Repayment Schedule<br/>Get Overdue & Current Installments]

    LoadSchedule --> SelectStrategy{Select Allocation<br/>Strategy}

    SelectStrategy -->|Progressive| Progressive[Progressive Strategy<br/>Past → Current → Future]
    SelectStrategy -->|Penalties First| PenaltyFirst[Penalty First Strategy]
    SelectStrategy -->|Principal First| PrincipalFirst[Principal Priority]

    Progressive --> AllocateOrder[Allocation Order:<br/>1. Past Due Penalties<br/>2. Past Due Fees<br/>3. Past Due Interest<br/>4. Past Due Principal<br/>5. Current Penalty<br/>6. Current Fee<br/>7. Current Interest<br/>8. Current Principal]

    AllocateOrder --> AllocatePenalty{Penalties Due?}
    AllocatePenalty -->|Yes| PayPenalty[Allocate to Penalties<br/>Update penalty_paid_derived]
    AllocatePenalty -->|No| AllocateFee{Fees Due?}
    PayPenalty --> AllocateFee

    AllocateFee -->|Yes| PayFee[Allocate to Fees<br/>Update fee_paid_derived]
    AllocateFee -->|No| AllocateInterest{Interest Due?}
    PayFee --> AllocateInterest

    AllocateInterest -->|Yes| PayInterest[Allocate to Interest<br/>Amount: $500<br/>Update interest_paid_derived]
    AllocateInterest -->|No| AllocatePrincipal{Principal Due?}
    PayInterest --> AllocatePrincipal

    AllocatePrincipal -->|Yes| PayPrincipal[Allocate to Principal<br/>Amount: $1,162.76<br/>Update principal_paid_derived]
    AllocatePrincipal -->|No| CheckRemaining
    PayPrincipal --> CheckRemaining{Amount Remaining?}

    CheckRemaining -->|Yes| Overpayment[Record Overpayment<br/>Update overpayment_derived<br/>Status → OVERPAID]
    CheckRemaining -->|No| UpdateInstallment[Update Installment<br/>Mark as Paid/Partial]

    Overpayment --> UpdateDerived
    UpdateInstallment --> UpdateDerived[Update Loan Derived Balances<br/>principal_outstanding -= paid<br/>interest_outstanding -= paid<br/>total_outstanding -= paid]

    UpdateDerived --> CreateJE[Create Journal Entries<br/>DR: Cash/Bank $1,662.76<br/>CR: Loan Portfolio $1,162.76<br/>CR: Interest Income $500.00]

    CreateJE --> CheckStatus{Loan Fully Paid?}

    CheckStatus -->|Yes| CloseLoan[Update Status<br/>ACTIVE → CLOSED_OBLIGATIONS_MET<br/>Set closedon_date]
    CheckStatus -->|No| StayActive[Status Remains ACTIVE<br/>Update next_payment_date]

    CloseLoan --> PublishEvent[Publish LoanClosedBusinessEvent]
    StayActive --> PublishEvent2[Publish LoanRepaymentBusinessEvent]

    PublishEvent --> NotifyCustomer[Send SMS/Email<br/>Loan Closed Successfully]
    PublishEvent2 --> NotifyCustomer2[Send SMS<br/>Payment Received<br/>Balance: $XX,XXX]

    NotifyCustomer --> End([End])
    NotifyCustomer2 --> End

    style Start fill:#e1f5ff
    style End fill:#c8e6c9
    style CloseLoan fill:#fff9c4
    style Overpayment fill:#ffe0b2
    style CreateJE fill:#f3e5f5
```

## 4. Loan Transaction Types & Actions

```mermaid
graph LR
    subgraph "Core Transactions"
        Disbursement[DISBURSEMENT<br/>Type: 1<br/>Action: Disburse Funds]
        Repayment[REPAYMENT<br/>Type: 2<br/>Action: Receive Payment]
        Writeoff[WRITEOFF<br/>Type: 6<br/>Action: Bad Debt]
    end

    subgraph "Adjustment Transactions"
        WaiveInterest[WAIVE_INTEREST<br/>Type: 4<br/>Action: Forgive Interest]
        WaiveCharges[WAIVE_CHARGES<br/>Type: 9<br/>Action: Forgive Fees]
        Refund[REFUND<br/>Type: 16<br/>Action: Return Overpayment]
        ChargeAdjust[CHARGE_ADJUSTMENT<br/>Type: 24<br/>Action: Modify Charge]
    end

    subgraph "Advanced Transactions"
        ChargeOff[CHARGE_OFF<br/>Type: 25<br/>Action: Mark as Loss]
        DownPayment[DOWN_PAYMENT<br/>Type: 26<br/>Action: Initial Payment]
        Reage[REAGE<br/>Type: 27<br/>Action: Reset Aging]
        Reamortize[REAMORTIZE<br/>Type: 28<br/>Action: Restructure]
        Chargeback[CHARGEBACK<br/>Type: 29<br/>Action: Reverse Payment]
    end

    subgraph "Accounting Transactions"
        Accrual[ACCRUAL<br/>Type: 10<br/>Action: Accrue Interest]
        IncomePosting[INCOME_POSTING<br/>Type: 19<br/>Action: Post Income]
    end

    subgraph "Transfer Transactions"
        InitTransfer[INITIATE_TRANSFER<br/>Type: 12<br/>Action: Start Transfer]
        ApproveTransfer[APPROVE_TRANSFER<br/>Type: 13<br/>Action: Accept Transfer]
        WithdrawTransfer[WITHDRAW_TRANSFER<br/>Type: 14<br/>Action: Cancel Transfer]
        RejectTransfer[REJECT_TRANSFER<br/>Type: 15<br/>Action: Decline Transfer]
    end

    subgraph "Recovery Transactions"
        Recovery[RECOVERY_REPAYMENT<br/>Type: 8<br/>Action: Collect After Writeoff]
        GoodwillCredit[GOODWILL_CREDIT<br/>Type: 23<br/>Action: Service Credit]
    end

    Disbursement -.Creates.-> LoanPortfolio[Loan Portfolio<br/>Asset Account]
    Repayment -.Reduces.-> LoanPortfolio
    Writeoff -.Removes.-> LoanPortfolio
    Writeoff -.Creates.-> LossExpense[Loss Expense<br/>Expense Account]
    ChargeOff -.Creates.-> ChargeOffExpense[Charge-off Expense<br/>Expense Account]
    Accrual -.Creates.-> InterestReceivable[Interest Receivable<br/>Asset Account]

    style Disbursement fill:#e3f2fd
    style Repayment fill:#e8f5e9
    style Writeoff fill:#ffebee
    style ChargeOff fill:#fff3e0
    style Reage fill:#f3e5f5
    style Reamortize fill:#e0f2f1
```

## 5. Interest Calculation Methods

```mermaid
graph TB
    subgraph "Declining Balance Method"
        DB_Start[Principal: $50,000<br/>Rate: 12% annual = 1% monthly]
        DB_Start --> DB_M1[Month 1:<br/>Balance: $50,000<br/>Interest: $50,000 × 1% = $500<br/>Principal: $1,162.76<br/>Payment: $1,662.76]
        DB_M1 --> DB_M2[Month 2:<br/>Balance: $48,837.24<br/>Interest: $48,837.24 × 1% = $488.37<br/>Principal: $1,174.39<br/>Payment: $1,662.76]
        DB_M2 --> DB_M3[Month 3:<br/>Balance: $47,662.85<br/>Interest: $47,662.85 × 1% = $476.63<br/>Principal: $1,186.13<br/>Payment: $1,662.76]
        DB_M3 --> DB_Note[Interest DECREASES<br/>Principal INCREASES<br/>Payment CONSTANT]
    end

    subgraph "Flat Interest Method"
        Flat_Start[Principal: $50,000<br/>Rate: 12% annual<br/>Term: 36 months]
        Flat_Start --> Flat_Calc[Total Interest:<br/>$50,000 × 12% × 3 years = $18,000]
        Flat_Calc --> Flat_Monthly[Per Month:<br/>Interest: $18,000 ÷ 36 = $500<br/>Principal: $50,000 ÷ 36 = $1,388.89<br/>Payment: $1,888.89]
        Flat_Monthly --> Flat_Note[Interest CONSTANT<br/>Principal CONSTANT<br/>Payment CONSTANT]
    end

    subgraph "Compound Interest Formula"
        Formula[EMI = P × r × (1+r)^n / ((1+r)^n - 1)]
        Formula --> Variables[P = Principal Amount<br/>r = Monthly Interest Rate<br/>n = Number of Months]
        Variables --> Example[Example:<br/>P = $50,000<br/>r = 0.01 monthly<br/>n = 36 months<br/>EMI = $1,662.76]
    end

    style DB_Note fill:#e8f5e9
    style Flat_Note fill:#fff3e0
    style Example fill:#e1f5ff
```

## 6. Loan Charge Management

```mermaid
sequenceDiagram
    participant LO as Loan Officer
    participant API as REST API
    participant Service as Charge Service
    participant Loan as Loan Domain
    participant DB as Database
    participant Events as Event Bus

    %% Add Charge
    rect rgb(230, 240, 255)
        Note over LO,Events: ADD CHARGE TO LOAN
        LO->>API: POST /loans/123/charges<br/>{chargeId, amount, dueDate}
        API->>Service: addCharge(loanId, command)
        Service->>Loan: validateCharge()

        alt Disbursement Charge
            Loan->>Loan: Apply at disbursement
        else Installment Charge
            Loan->>Loan: Distribute across installments
        else Specified Due Date
            Loan->>Loan: Due on specific date
        else Overdue Charge
            Loan->>Loan: Apply when overdue
        end

        Loan->>DB: INSERT loan_charge<br/>(amount, outstanding)
        Service->>Events: LoanChargeAddedBusinessEvent
        Service-->>API: Charge Added
        API-->>LO: Charge #456 Added
    end

    %% Pay Charge
    rect rgb(240, 255, 240)
        Note over LO,Events: PAY CHARGE
        LO->>API: POST /loans/123/charges/456?command=pay<br/>{amount, paymentDate}
        API->>Service: payCharge(chargeId, amount)
        Service->>DB: INSERT loan_transaction<br/>(type=CHARGE_PAYMENT)
        Service->>DB: UPDATE loan_charge<br/>(amount_paid_derived += amount)
        Service->>DB: INSERT loan_charge_paid_by<br/>(txn_id, charge_id, amount)

        Service->>Events: LoanChargePaidBusinessEvent
        Service-->>API: Charge Paid
        API-->>LO: Payment Recorded
    end

    %% Waive Charge
    rect rgb(255, 245, 230)
        Note over LO,Events: WAIVE CHARGE
        LO->>API: POST /loans/123/charges/456?command=waive
        API->>Service: waiveCharge(chargeId)
        Service->>DB: UPDATE loan_charge<br/>(waived=true, amount_waived)
        Service->>DB: INSERT loan_transaction<br/>(type=WAIVE_CHARGES)
        Service->>Events: LoanChargeWaivedBusinessEvent
        Service-->>API: Charge Waived
        API-->>LO: Waiver Applied
    end
```

## 7. Loan Delinquency & NPA Classification

```mermaid
flowchart TD
    Start([Daily COB Process]) --> CheckLoans[Check All Active Loans]

    CheckLoans --> CalcOverdue[Calculate Days Overdue<br/>days = today - due_date]

    CalcOverdue --> ClassifyBucket{Classify<br/>Delinquency Bucket}

    ClassifyBucket -->|0 days| Current[Current<br/>No Action]
    ClassifyBucket -->|1-30 days| Early[Early Delinquency<br/>Bucket: 1-30]
    ClassifyBucket -->|31-60 days| Medium[Medium Delinquency<br/>Bucket: 31-60]
    ClassifyBucket -->|61-90 days| Late[Late Delinquency<br/>Bucket: 61-90]
    ClassifyBucket -->|90+ days| NPA[Non-Performing Asset<br/>Bucket: 90+]

    Current --> UpdateAging
    Early --> ApplyPenalty[Apply Overdue Penalty<br/>Late Payment Fee]
    Medium --> ApplyPenalty
    Late --> ApplyPenalty

    ApplyPenalty --> UpdateAging[Update m_loan_arrears_aging<br/>days_overdue, principal_overdue<br/>interest_overdue]

    NPA --> MarkNPA[Mark as NPA<br/>npa_status = true]
    MarkNPA --> CalcProvisioning[Calculate Provisioning<br/>Based on Aging]

    CalcProvisioning --> ProvisionPercent{Provisioning %}
    ProvisionPercent -->|Standard 0-30| Prov0[0% Provision]
    ProvisionPercent -->|Sub-standard 31-90| Prov25[25% Provision]
    ProvisionPercent -->|Doubtful 91-180| Prov50[50% Provision]
    ProvisionPercent -->|Loss 180+| Prov100[100% Provision]

    Prov0 --> UpdateAging
    Prov25 --> CreateProvision[Create Provisioning Entry<br/>DR: Provision Expense<br/>CR: Loan Loss Reserve]
    Prov50 --> CreateProvision
    Prov100 --> CreateProvision

    CreateProvision --> UpdateAging
    UpdateAging --> PublishEvent[Publish Delinquency Event]
    PublishEvent --> Notify{Notification<br/>Threshold?}

    Notify -->|Yes| SendAlert[Send SMS/Email Alert<br/>To Loan Officer & Client]
    Notify -->|No| End
    SendAlert --> End([End])

    style Current fill:#c8e6c9
    style Early fill:#fff9c4
    style Medium fill:#ffe0b2
    style Late fill:#ffccbc
    style NPA fill:#ffcdd2
    style CreateProvision fill:#f3e5f5
```

## Transaction Type Summary

| Type | Code | Purpose | GL Impact |
|------|------|---------|-----------|
| DISBURSEMENT | 1 | Disburse loan funds | DR: Loan Portfolio, CR: Cash |
| REPAYMENT | 2 | Customer payment | DR: Cash, CR: Loan Portfolio/Interest |
| WAIVE_INTEREST | 4 | Forgive interest | DR: Interest Waived, CR: Interest Receivable |
| WRITEOFF | 6 | Bad debt write-off | DR: Loss Expense, CR: Loan Portfolio |
| WAIVE_CHARGES | 9 | Forgive fees | DR: Fee Waived, CR: Fee Receivable |
| ACCRUAL | 10 | Accrue interest | DR: Interest Receivable, CR: Interest Income |
| CHARGE_PAYMENT | 17 | Pay specific charge | DR: Cash, CR: Fee Income |
| CHARGE_OFF | 25 | Mark as charge-off | DR: Charge-off Expense, CR: Loan Portfolio |
| REAGE | 27 | Reset aging clock | Modify schedule, reset aging |
| REAMORTIZE | 28 | Restructure payments | Recalculate schedule |
| CHARGEBACK | 29 | Reverse payment | Reverse original GL entries |
