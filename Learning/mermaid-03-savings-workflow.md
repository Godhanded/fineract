# Savings Account Management - Complete Workflow

## 1. Savings Account Lifecycle State Diagram

```mermaid
stateDiagram-v2
    [*] --> SUBMITTED: Create Account

    SUBMITTED --> APPROVED: Approve
    SUBMITTED --> REJECTED: Reject
    SUBMITTED --> WITHDRAWN: Client Withdraws

    APPROVED --> ACTIVE: Activate<br/>(with deposit)
    APPROVED --> SUBMITTED: Undo Approval

    ACTIVE --> CLOSED: Close Account
    ACTIVE --> TRANSFER_IN_PROGRESS: Initiate Transfer
    ACTIVE --> DORMANT: No Activity<br/>(Sub-status)
    ACTIVE --> MATURED: FD/RD Maturity

    DORMANT --> ACTIVE: Reactivate
    DORMANT --> ESCHEAT: Long-term Dormant<br/>(Sub-status)

    TRANSFER_IN_PROGRESS --> ACTIVE: Complete Transfer
    TRANSFER_IN_PROGRESS --> TRANSFER_ON_HOLD: Reject Transfer

    MATURED --> CLOSED: Withdraw
    MATURED --> ACTIVE: Renew/Rollover

    CLOSED --> [*]
    REJECTED --> [*]
    WITHDRAWN --> [*]

    note right of SUBMITTED
        Status: 100
        Pending Approval
    end note

    note right of APPROVED
        Status: 200
        Ready for Activation
    end note

    note right of ACTIVE
        Status: 300
        Accepting Transactions
    end note

    note right of DORMANT
        Sub-status
        90+ days no activity
    end note

    note right of MATURED
        Status: 800
        FD/RD Term Complete
    end note
```

## 2. Savings Account Opening to Activation

```mermaid
sequenceDiagram
    actor Client as Client
    actor Staff as Branch Staff
    participant API as REST API
    participant Service as Savings Service
    participant Domain as Savings Domain
    participant Product as Savings Product
    participant DB as Database
    participant Events as Event Bus
    participant Accounting as GL Service

    %% Account Application
    rect rgb(200, 230, 250)
        Note over Client,Accounting: STEP 1: ACCOUNT APPLICATION
        Client->>Staff: Request Savings Account
        Staff->>API: POST /savingsaccounts<br/>{clientId, productId, nominalRate}
        API->>Service: createSavingsAccount(command)

        Service->>Product: getProduct(productId)
        Product-->>Service: {min_balance, interest_rate, charges}

        Service->>Domain: new SavingsAccount(SUBMITTED)
        Domain->>Domain: validateMinBalance()
        Domain->>Domain: applyProductDefaults()

        Domain->>DB: INSERT m_savings_account<br/>(status=100, balance=0)
        Service->>Events: SavingsAccountCreatedBusinessEvent

        Service-->>API: {savingsAccountId: 2001}
        API-->>Staff: Account #2001 Created
        Staff->>Client: Account Application Submitted
    end

    %% Approval
    rect rgb(255, 250, 200)
        Note over Client,Accounting: STEP 2: APPROVAL
        Staff->>API: POST /savingsaccounts/2001?command=approve
        API->>Service: approveSavingsAccount(command)
        Service->>Domain: approve(approvalDate)

        Domain->>DB: UPDATE m_savings_account<br/>(status=200, approved_on_date)
        Service->>Events: SavingsAccountApprovedBusinessEvent

        Service-->>API: Account Approved
        API-->>Staff: Ready for Activation
    end

    %% Activation with Initial Deposit
    rect rgb(200, 250, 200)
        Note over Client,Accounting: STEP 3: ACTIVATION
        Client->>Staff: Deposit Initial Amount: $5,000
        Staff->>API: POST /savingsaccounts/2001?command=activate<br/>{activationDate}
        API->>Service: activateSavingsAccount(command)
        Service->>Domain: activate(activationDate)

        Domain->>DB: UPDATE m_savings_account<br/>(status=300, activated_on_date)

        Note over Staff,Accounting: Record Initial Deposit
        Staff->>API: POST /savingsaccounts/2001/transactions<br/>?command=deposit<br/>{amount: 5000}
        API->>Service: deposit(accountId, amount)

        Service->>Domain: recordDeposit(5000)
        Domain->>Domain: validateMinBalance(OK)

        Domain->>DB: INSERT m_savings_account_transaction<br/>(type=DEPOSIT, amount=5000)
        Domain->>DB: UPDATE m_savings_account<br/>(balance_derived=5000,<br/>total_deposits=5000)

        Service->>Accounting: createJournalEntries()
        Accounting->>DB: INSERT journal_entry<br/>DR: Cash $5,000<br/>CR: Savings Deposits $5,000

        Service->>Events: SavingsAccountActivatedBusinessEvent
        Service->>Events: DepositTransactionCreatedBusinessEvent

        Service-->>API: Account Activated & Funded
        API-->>Staff: Account #2001 Active
        Staff->>Client: Account Ready for Use<br/>Balance: $5,000
    end
```

## 3. Daily Savings Transactions Flow

```mermaid
flowchart TD
    Start([Customer Initiates Transaction]) --> TxnType{Transaction Type?}

    %% Deposit Flow
    TxnType -->|Deposit| ValidateDeposit[Validate Deposit<br/>Check account active<br/>Check amount > 0]
    ValidateDeposit --> RecordDeposit[Create Transaction<br/>type: DEPOSIT<br/>amount: $500]
    RecordDeposit --> UpdateBalDeposit[Update Balance<br/>balance += $500<br/>total_deposits += $500]
    UpdateBalDeposit --> CheckInterest[Check if Interest<br/>Calculation Date]
    CheckInterest -->|Yes| CalcInterest[Calculate Interest<br/>Since Last Calculation]
    CheckInterest -->|No| GLDeposit
    CalcInterest --> GLDeposit[Journal Entry<br/>DR: Cash $500<br/>CR: Savings Deposits $500]
    GLDeposit --> NotifyDeposit[Send SMS:<br/>Deposited $500<br/>Balance: $X,XXX]
    NotifyDeposit --> EndDeposit([End])

    %% Withdrawal Flow
    TxnType -->|Withdrawal| ValidateWithdrawal[Validate Withdrawal<br/>Check balance sufficient<br/>Check min_balance<br/>Check lock-in period]

    ValidateWithdrawal -->|Valid| CheckWithdrawalFee{Withdrawal<br/>Fee Applicable?}
    ValidateWithdrawal -->|Invalid| RejectTxn[Reject Transaction<br/>Insufficient Balance]
    RejectTxn --> EndReject([End - Rejected])

    CheckWithdrawalFee -->|Yes| ApplyFee[Apply Withdrawal Fee<br/>Create WITHDRAWAL_FEE txn]
    CheckWithdrawalFee -->|No| RecordWithdrawal

    ApplyFee --> RecordWithdrawal[Create Transaction<br/>type: WITHDRAWAL<br/>amount: $200]
    RecordWithdrawal --> UpdateBalWithdraw[Update Balance<br/>balance -= $200<br/>total_withdrawals += $200]

    UpdateBalWithdraw --> CheckOverdraft{Overdraft<br/>Allowed?}
    CheckOverdraft -->|Yes & Negative| CalcODInterest[Calculate Overdraft<br/>Interest Charge]
    CheckOverdraft -->|No/Positive| GLWithdrawal

    CalcODInterest --> GLWithdrawal[Journal Entry<br/>DR: Savings Deposits $200<br/>CR: Cash $200]
    GLWithdrawal --> NotifyWithdrawal[Send SMS:<br/>Withdrawn $200<br/>Balance: $X,XXX]
    NotifyWithdrawal --> EndWithdrawal([End])

    %% Hold Amount
    TxnType -->|Hold Amount| ValidateHold[Validate Hold<br/>Check balance >= hold]
    ValidateHold --> RecordHold[Create Transaction<br/>type: AMOUNT_HOLD<br/>frozen_amount: $1,000]
    RecordHold --> UpdateHold[Update<br/>on_hold_funds += $1,000<br/>available_balance -= $1,000]
    UpdateHold --> NotifyHold[Notify: $1,000 On Hold]
    NotifyHold --> EndHold([End])

    %% Release Hold
    TxnType -->|Release Hold| FindHold[Find Hold Transaction]
    FindHold --> RecordRelease[Create Transaction<br/>type: AMOUNT_RELEASE<br/>release_id: hold_txn_id]
    RecordRelease --> UpdateRelease[Update<br/>on_hold_funds -= $1,000<br/>available_balance += $1,000]
    UpdateRelease --> NotifyRelease[Notify: Hold Released]
    NotifyRelease --> EndRelease([End])

    style ValidateDeposit fill:#e8f5e9
    style RecordWithdrawal fill:#fff3e0
    style RejectTxn fill:#ffcdd2
    style CalcInterest fill:#e1f5ff
    style GLDeposit fill:#f3e5f5
```

## 4. Interest Calculation & Posting Process

```mermaid
sequenceDiagram
    participant Scheduler as Job Scheduler
    participant COB as COB Batch Job
    participant Service as Interest Service
    participant Account as Savings Account
    participant DB as Database
    participant GL as GL Service
    participant Events as Event Bus

    %% Daily Interest Calculation
    rect rgb(230, 245, 255)
        Note over Scheduler,Events: DAILY: INTEREST CALCULATION
        Scheduler->>COB: Trigger Daily COB
        COB->>Service: calculateDailyInterest()

        Service->>DB: SELECT all active accounts
        DB-->>Service: [Account #2001, #2002, ...]

        loop For Each Account
            Service->>Account: getDailyBalance()
            Account-->>Service: balance: $10,000

            Service->>Account: getInterestConfig()
            Account-->>Service: rate: 12% annual,<br/>compounding: DAILY,<br/>posting: MONTHLY

            Service->>Service: dailyInterest =<br/>balance × (rate/365)<br/>= $10,000 × (0.12/365)<br/>= $3.29

            Service->>DB: UPDATE daily_balance_tracker<br/>(date, balance, interest_accrued)
            Service->>DB: UPDATE m_savings_account<br/>(last_interest_calculation_date)
        end

        Service-->>COB: Calculation Complete
    end

    %% Monthly Interest Posting
    rect rgb(245, 255, 245)
        Note over Scheduler,Events: MONTHLY: INTEREST POSTING
        Scheduler->>Service: postInterestForSavings()<br/>(if posting date)

        Service->>DB: SELECT accounts<br/>WHERE posting_date = today

        loop For Each Account
            Service->>DB: SELECT SUM(daily_interest)<br/>FROM daily_balance_tracker<br/>WHERE month = current

            DB-->>Service: total_interest: $100.00

            alt Withhold Tax Applicable
                Service->>Service: tax = $100 × 15% = $15
                Service->>DB: INSERT transaction<br/>(type=WITHHOLD_TAX, amount=$15)
                Service->>DB: INSERT transaction<br/>(type=INTEREST_POSTING, amount=$85)
                Service->>DB: UPDATE account<br/>(balance += $85,<br/>total_interest_posted += $85,<br/>total_withhold_tax += $15)
            else No Tax
                Service->>DB: INSERT transaction<br/>(type=INTEREST_POSTING, amount=$100)
                Service->>DB: UPDATE account<br/>(balance += $100,<br/>total_interest_posted += $100)
            end

            Service->>GL: createJournalEntries()

            alt Cash Accounting
                GL->>DB: DR: Interest Expense $100<br/>CR: Savings Deposits $100
            else Accrual Accounting
                GL->>DB: DR: Interest Expense $100<br/>CR: Interest Payable $100
                Note over GL: Separate entry when posted to account
            end

            Service->>Events: InterestPostedBusinessEvent

            alt Transfer to Linked Account
                Service->>Service: transferToLinkedAccount($100)
            end
        end

        Service-->>Scheduler: Posting Complete
    end
```

## 5. Interest Calculation Methods Comparison

```mermaid
graph TB
    subgraph "Daily Balance Method"
        DB_Account[Account: #2001<br/>Rate: 12% annual<br/>Posting: Monthly]

        DB_Account --> DB_Day1[Day 1: Balance $10,000<br/>Interest: $10,000 × 0.12/365 = $3.29]
        DB_Day1 --> DB_Day2[Day 2: Balance $10,000<br/>Interest: $10,000 × 0.12/365 = $3.29]
        DB_Day2 --> DB_Day3[Day 3: Deposit $5,000<br/>Balance: $15,000<br/>Interest: $15,000 × 0.12/365 = $4.93]
        DB_Day3 --> DB_Day15[Day 15: Withdrawal $3,000<br/>Balance: $12,000<br/>Interest: $12,000 × 0.12/365 = $3.95]
        DB_Day15 --> DB_Month[Month End: Sum Daily Interest<br/>Total: $110.50]
        DB_Month --> DB_Post[Post to Account<br/>Balance: $12,000 + $110.50 = $12,110.50]
    end

    subgraph "Average Daily Balance Method"
        ADB_Account[Account: #2002<br/>Rate: 12% annual<br/>Posting: Monthly]

        ADB_Account --> ADB_Days[Day 1-10: $10,000<br/>Day 11-20: $15,000<br/>Day 21-30: $12,000]
        ADB_Days --> ADB_Calc[Average Balance:<br/>(10K×10 + 15K×10 + 12K×10) / 30<br/>= $12,333.33]
        ADB_Calc --> ADB_Interest[Monthly Interest:<br/>$12,333.33 × 12% × (30/365)<br/>= $121.64]
        ADB_Interest --> ADB_Post[Post to Account<br/>Balance: $12,000 + $121.64 = $12,121.64]
    end

    subgraph "Minimum Balance Method"
        MB_Account[Account: #2003<br/>Rate: 12% annual<br/>Posting: Monthly<br/>Method: Minimum Balance]

        MB_Account --> MB_Days[Day 1-10: $10,000<br/>Day 11-20: $15,000<br/>Day 21-30: $8,000 ← Minimum]
        MB_Days --> MB_Interest[Monthly Interest:<br/>$8,000 × 12% × (30/365)<br/>= $78.90]
        MB_Interest --> MB_Post[Post to Account<br/>Balance varies, but interest<br/>based on minimum: $8,000]
    end

    style DB_Post fill:#e8f5e9
    style ADB_Post fill:#fff9c4
    style MB_Post fill:#e1f5ff
```

## 6. Fixed Deposit Workflow

```mermaid
flowchart TD
    Start([Client Requests Fixed Deposit]) --> CreateFD[Create Fixed Deposit Account<br/>Principal: $100,000<br/>Term: 1 year<br/>Rate: 8% annual]

    CreateFD --> Approve[Approve Account]
    Approve --> Activate[Activate with Deposit<br/>Record Initial Transaction]

    Activate --> LockFunds[Lock Funds<br/>No withdrawals allowed<br/>Lock-in period: 12 months]

    LockFunds --> DailyInterest[Daily Interest Calculation<br/>Simple/Compound based on config]

    DailyInterest --> CheckMaturity{Check Maturity Date}

    CheckMaturity -->|Not Reached| ContinueAccrual[Continue Accruing<br/>Daily Interest]
    ContinueAccrual --> DailyInterest

    CheckMaturity -->|Maturity Reached| CalcMaturity[Calculate Maturity Amount<br/>Principal: $100,000<br/>Interest: $8,000<br/>Total: $108,000]

    CalcMaturity --> UpdateStatus[Update Status<br/>ACTIVE → MATURED<br/>Set maturity_date]

    UpdateStatus --> NotifyClient[Notify Client<br/>SMS/Email: FD Matured<br/>Amount Available: $108,000]

    NotifyClient --> ClientChoice{Client Decision?}

    ClientChoice -->|Withdraw| ProcessWithdrawal[Close Account<br/>Transfer $108,000 to Client]
    ClientChoice -->|Renew| RenewFD[Create New FD<br/>Principal: $108,000<br/>Same/New Terms]
    ClientChoice -->|Transfer| TransferSavings[Transfer to Savings Account<br/>Linked Account]

    ProcessWithdrawal --> GLWithdraw[Journal Entry<br/>DR: FD Deposits $100,000<br/>DR: Interest Expense $8,000<br/>CR: Cash $108,000]
    RenewFD --> CreateFD
    TransferSavings --> GLTransfer[Journal Entry<br/>DR: FD Deposits $108,000<br/>CR: Savings Deposits $108,000]

    GLWithdraw --> CloseAccount[Close FD Account<br/>Status: CLOSED]
    GLTransfer --> CloseAccount

    CloseAccount --> End([End])

    %% Pre-mature Closure Path
    CheckMaturity -->|Pre-mature Request| ValidatePreMature{Pre-mature<br/>Allowed?}

    ValidatePreMature -->|Yes| CalcPenalty[Calculate Penalty<br/>Reduced Rate: 6% instead of 8%<br/>Tenure: 6 months<br/>Interest: $3,000<br/>Penalty: $1,000]
    ValidatePreMature -->|No| RejectRequest[Reject Closure<br/>Wait for Maturity]

    CalcPenalty --> ApplyPenalty[Total Payout:<br/>Principal: $100,000<br/>Interest: $3,000<br/>Less Penalty: -$1,000<br/>Net: $102,000]

    ApplyPenalty --> ProcessPreMature[Process Closure<br/>Status: PRE_MATURE_CLOSED]
    ProcessPreMature --> GLPreMature[Journal Entry<br/>DR: FD Deposits $100,000<br/>DR: Interest Expense $3,000<br/>CR: Penalty Income $1,000<br/>CR: Cash $102,000]

    GLPreMature --> End
    RejectRequest --> End

    style CreateFD fill:#e3f2fd
    style CalcMaturity fill:#e8f5e9
    style CalcPenalty fill:#ffebee
    style ProcessWithdrawal fill:#f3e5f5
```

## 7. Recurring Deposit Workflow

```mermaid
sequenceDiagram
    participant Client as Client
    participant Staff as Staff
    participant API as REST API
    participant Service as RD Service
    participant Scheduler as Job Scheduler
    participant DB as Database
    participant GL as GL Service

    %% RD Account Creation
    rect rgb(230, 240, 250)
        Note over Client,GL: RD ACCOUNT CREATION
        Client->>Staff: Request Recurring Deposit<br/>Monthly: $1,000 for 12 months
        Staff->>API: POST /savingsaccounts (RD)<br/>{deposit_amount: 1000,<br/>frequency: MONTHLY,<br/>term: 12 months}
        API->>Service: createRecurringDeposit()

        Service->>Service: calculateMaturityAmount()<br/>Expected Total: $12,000<br/>Expected Interest: $800<br/>Maturity Amount: $12,800

        Service->>DB: INSERT m_savings_account<br/>(deposit_type=RECURRING_DEPOSIT)
        Service->>DB: INSERT m_deposit_recurring_detail<br/>(deposit_amount=1000)
        Service->>DB: INSERT recurring_schedule<br/>(12 monthly installments)

        Service-->>API: RD Account Created
        API-->>Staff: Account #3001 Created
    end

    %% Monthly Deposit Installments
    rect rgb(240, 250, 240)
        Note over Client,GL: MONTHLY DEPOSITS
        loop Each Month (12 times)
            Client->>Staff: Monthly Deposit: $1,000
            Staff->>API: POST /savingsaccounts/3001/transactions<br/>?command=deposit<br/>{amount: 1000}
            API->>Service: recordDeposit(accountId, 1000)

            Service->>DB: INSERT transaction<br/>(type=DEPOSIT, amount=1000)
            Service->>DB: UPDATE recurring_schedule<br/>(installment_X completed)
            Service->>DB: UPDATE account<br/>(balance += 1000,<br/>total_deposits += 1000)

            Service->>GL: createJournalEntry()
            GL->>DB: DR: Cash $1,000<br/>CR: RD Deposits $1,000

            Service-->>API: Deposit Recorded
        end
    end

    %% Missed Deposit Handling
    rect rgb(255, 245, 230)
        Note over Client,GL: MISSED DEPOSIT PENALTY
        Scheduler->>Service: checkMissedDeposits()
        Service->>DB: SELECT accounts<br/>WHERE installment overdue

        alt Deposit Missed
            Service->>DB: UPDATE account<br/>(no_of_overdue_installments += 1,<br/>total_overdue_amount += 1000)
            Service->>Service: applyPenalty()<br/>Penalty: $50
            Service->>DB: INSERT transaction<br/>(type=PENALTY, amount=50)
            Service->>DB: UPDATE account<br/>(balance -= 50)
        end
    end

    %% Maturity Processing
    rect rgb(245, 255, 245)
        Note over Client,GL: MATURITY & PAYOUT
        Scheduler->>Service: processMaturedAccounts()
        Service->>DB: SELECT accounts<br/>WHERE maturity_date = today

        Service->>Service: calculateFinalAmount()<br/>Deposits: $12,000<br/>Interest Earned: $850<br/>Penalties: -$50<br/>Final Amount: $12,800

        Service->>DB: UPDATE account<br/>(status=MATURED,<br/>maturity_amount=12800)

        Service->>Client: Notify: RD Matured<br/>Amount: $12,800

        Client->>Staff: Withdraw Maturity Amount
        Staff->>API: POST /savingsaccounts/3001?command=close
        API->>Service: closeAccount()

        Service->>GL: createJournalEntry()
        GL->>DB: DR: RD Deposits $12,800<br/>CR: Cash $12,800

        Service->>DB: UPDATE account (status=CLOSED)
        Service-->>Client: Amount Disbursed
    end
```

## 8. Dormancy & Escheat Process

```mermaid
flowchart TD
    Start([Daily Dormancy Check]) --> CheckInactive[Check Accounts<br/>No transactions in X days]

    CheckInactive --> CalcDays{Days Since<br/>Last Transaction}

    CalcDays -->|< 90 days| Active[Status: ACTIVE<br/>No action]
    CalcDays -->|90-180 days| Inactive[Sub-status: INACTIVE<br/>Mark for monitoring]
    CalcDays -->|180-365 days| Dormant[Sub-status: DORMANT<br/>Apply dormancy fee]
    CalcDays -->|> 7 years| Escheat[Sub-status: ESCHEAT<br/>Transfer to government]

    Active --> End([End])
    Inactive --> NotifyInactive[Send Warning Notice<br/>Email/SMS to Client]
    NotifyInactive --> End

    Dormant --> ApplyDormancyFee[Apply Dormancy Fee<br/>Monthly: $10]
    ApplyDormancyFee --> CreateFeeTxn[Create Transaction<br/>type: DORMANCY_FEE<br/>amount: $10]
    CreateFeeTxn --> UpdateBalance[Update Balance<br/>balance -= $10<br/>total_fees += $10]
    UpdateBalance --> CheckZeroBalance{Balance<br/>= $0?}
    CheckZeroBalance -->|Yes| CloseAccount[Auto-close Account<br/>Status: CLOSED]
    CheckZeroBalance -->|No| NotifyDormant[Notify Client<br/>Account Dormant<br/>Reactivate Required]
    CloseAccount --> End
    NotifyDormant --> End

    Escheat --> CalculateEscheat[Calculate Escheat Amount<br/>Remaining Balance + Interest]
    CalculateEscheat --> CreateEscheatTxn[Create Transaction<br/>type: ESCHEAT<br/>amount: balance]
    CreateEscheatTxn --> TransferFunds[Transfer to<br/>Government Escheat Account]
    TransferFunds --> GLEscheat[Journal Entry<br/>DR: Savings Deposits<br/>CR: Escheat Liability]
    GLEscheat --> CloseEscheat[Close Account<br/>Status: CLOSED<br/>Reason: ESCHEAT]
    CloseEscheat --> NotifyAuthority[Notify Regulatory Authority<br/>Escheat Report]
    NotifyAuthority --> End

    style Active fill:#c8e6c9
    style Inactive fill:#fff9c4
    style Dormant fill:#ffe0b2
    style Escheat fill:#ffcdd2
```

## Transaction Type Summary

| Type | Code | Purpose | Impact |
|------|------|---------|--------|
| DEPOSIT | 1 | Customer deposit | balance += amount |
| WITHDRAWAL | 2 | Customer withdrawal | balance -= amount |
| INTEREST_POSTING | 3 | Post accrued interest | balance += interest |
| WITHDRAWAL_FEE | 4 | Charge withdrawal fee | balance -= fee |
| ANNUAL_FEE | 5 | Annual maintenance | balance -= fee |
| WAIVE_CHARGES | 6 | Waive fees | balance += waived |
| OVERDRAFT_INTEREST | 17 | Overdraft charge | balance -= OD_interest |
| WITHHOLD_TAX | 18 | Tax on interest | balance -= tax |
| AMOUNT_HOLD | 20 | Hold/freeze funds | on_hold_funds += amount |
| AMOUNT_RELEASE | 21 | Release hold | on_hold_funds -= amount |
| ESCHEAT | 19 | Transfer to government | balance = 0 |
