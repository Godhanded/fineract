# Accounting & General Ledger Integration - Workflow

## 1. Automatic Journal Entry Generation Flow

```mermaid
flowchart TB
    Start([Banking Transaction Occurs]) --> TxnType{Transaction<br/>Type?}

    %% Loan Disbursement
    TxnType -->|Loan Disbursement| LoanDisburse[Loan Disbursement<br/>Amount: $50,000<br/>Loan #1001]

    LoanDisburse --> GetLoanMapping[Get Product-to-GL Mapping<br/>Product: Personal Loan]

    GetLoanMapping --> LoanAccounts[GL Accounts:<br/>Fund Source: 1010 (Asset)<br/>Loan Portfolio: 1100 (Asset)]

    LoanAccounts --> CreateLoanJE[Create Journal Entries:<br/>Entry 1: DR Loan Portfolio $50,000<br/>Entry 2: CR Fund Source $50,000]

    CreateLoanJE --> PostLoanJE[POST to m_journal_entry<br/>transaction_id: loan_txn_123<br/>office_id: 1]

    PostLoanJE --> UpdateTrialBal1[Update Trial Balance<br/>Loan Portfolio +$50,000<br/>Fund Source -$50,000]

    UpdateTrialBal1 --> LoanComplete([Complete])

    %% Loan Repayment
    TxnType -->|Loan Repayment| LoanRepay[Loan Repayment<br/>Total: $1,662.76<br/>Principal: $1,162.76<br/>Interest: $500.00]

    LoanRepay --> GetRepayMapping[Get GL Mappings:<br/>Loan Portfolio: 1100<br/>Interest Income: 4100<br/>Fund Source: 1010]

    GetRepayMapping --> CreateRepayJE[Create Journal Entries:<br/>DR Cash/Fund $1,662.76<br/>CR Loan Portfolio $1,162.76<br/>CR Interest Income $500.00]

    CreateRepayJE --> PostRepayJE[POST to m_journal_entry<br/>3 entries created]

    PostRepayJE --> UpdateTrialBal2[Update Trial Balance<br/>Multiple accounts]

    UpdateTrialBal2 --> RepayComplete([Complete])

    %% Savings Deposit
    TxnType -->|Savings Deposit| SavDeposit[Savings Deposit<br/>Amount: $5,000<br/>Account #2001]

    SavDeposit --> GetSavMapping[Get GL Mappings:<br/>Cash: 1010<br/>Savings Deposits: 2100 (Liability)]

    GetSavMapping --> CreateSavJE[Create Journal Entries:<br/>DR Cash $5,000<br/>CR Savings Deposits $5,000]

    CreateSavJE --> PostSavJE[POST to m_journal_entry]

    PostSavJE --> UpdateTrialBal3[Update Trial Balance]

    UpdateTrialBal3 --> SavComplete([Complete])

    %% Interest Posting
    TxnType -->|Interest Posting| IntPosting[Interest Posting<br/>Savings Interest: $100<br/>Account #2001]

    IntPosting --> GetIntMapping[Get GL Mappings:<br/>Interest Expense: 5100<br/>Savings Deposits: 2100]

    GetIntMapping --> CheckAcctMethod{Accounting<br/>Method?}

    CheckAcctMethod -->|Cash| CashInt[Create Journal Entry:<br/>DR Interest Expense $100<br/>CR Savings Deposits $100]
    CheckAcctMethod -->|Accrual| AccrualInt[Create Journal Entries:<br/>1. DR Interest Expense $100<br/>   CR Interest Payable $100<br/>2. DR Interest Payable $100<br/>   CR Savings Deposits $100]

    CashInt --> PostIntJE
    AccrualInt --> PostIntJE[POST to m_journal_entry]

    PostIntJE --> UpdateTrialBal4[Update Trial Balance]

    UpdateTrialBal4 --> IntComplete([Complete])

    %% Loan Write-off
    TxnType -->|Loan Write-off| Writeoff[Loan Write-off<br/>Outstanding: $10,000<br/>Loan #1005]

    Writeoff --> GetWOMapping[Get GL Mappings:<br/>Loan Portfolio: 1100<br/>Loss Expense: 5200]

    GetWOMapping --> CreateWOJE[Create Journal Entries:<br/>DR Loss Written Off $10,000<br/>CR Loan Portfolio $10,000]

    CreateWOJE --> PostWOJE[POST to m_journal_entry]

    PostWOJE --> UpdateProvision[Update Provisioning<br/>Increase Provision 100%]

    UpdateProvision --> CreateProvJE[Create Journal Entry:<br/>DR Provision Expense $10,000<br/>CR Loan Loss Reserve $10,000]

    CreateProvJE --> PostProvJE[POST to m_journal_entry]

    PostProvJE --> UpdateTrialBal5[Update Trial Balance]

    UpdateTrialBal5 --> WOComplete([Complete])

    style CreateLoanJE fill:#e3f2fd
    style CreateRepayJE fill:#e8f5e9
    style CreateSavJE fill:#fff3e0
    style CashInt fill:#f3e5f5
    style AccrualInt fill:#fce4ec
    style CreateWOJE fill:#ffebee
```

## 2. Product-to-GL-Account Mapping Configuration

```mermaid
sequenceDiagram
    participant Admin as System Admin
    participant API as REST API
    participant Service as Product Service
    participant Mapping as GL Mapping Service
    participant DB as Database
    participant Validator as GL Validator

    %% Configure Loan Product Accounting
    rect rgb(230, 240, 255)
        Note over Admin,Validator: LOAN PRODUCT GL MAPPING
        Admin->>API: POST /loanproducts/5?command=updateAccountingRule
        API->>Service: updateProductAccounting(productId)

        Service->>Validator: validateGLAccounts()

        alt Cash-Based Accounting
            Admin->>API: {accountingRule: CASH_BASED,<br/>fundSourceAccountId: 1010,<br/>loanPortfolioAccountId: 1100,<br/>interestOnLoanAccountId: 4100,<br/>incomeFromFeeAccountId: 4110,<br/>incomeFromPenaltyAccountId: 4120,<br/>writeOffAccountId: 5200}

            Validator->>Validator: validateAsset(fundSource)
            Validator->>Validator: validateAsset(loanPortfolio)
            Validator->>Validator: validateIncome(interest)
            Validator->>Validator: validateIncome(fees)
            Validator->>Validator: validateExpense(writeOff)

            Validator-->>Service: Validation Passed

            Service->>DB: DELETE FROM product_gl_mapping<br/>WHERE product_id=5
            Service->>Mapping: createMappings(CASH_BASED)

            loop For Each Account Type
                Mapping->>DB: INSERT product_gl_mapping<br/>(product_id=5,<br/>product_type=LOAN,<br/>financial_account_type=X,<br/>glaccount_id=XXXX)
            end

        else Accrual-Based Accounting
            Admin->>API: {accountingRule: ACCRUAL_PERIODIC,<br/>...all cash accounts...<br/>PLUS:<br/>interestReceivableAccountId: 1110,<br/>feesReceivableAccountId: 1120,<br/>penaltiesReceivableAccountId: 1130}

            Service->>Mapping: createMappings(ACCRUAL_PERIODIC)

            Mapping->>DB: INSERT additional mappings<br/>(receivable accounts)
        end

        Service-->>API: Mapping Updated
        API-->>Admin: GL Accounts Configured
    end

    %% Configure Savings Product Accounting
    rect rgb(240, 255, 240)
        Note over Admin,Validator: SAVINGS PRODUCT GL MAPPING
        Admin->>API: POST /savingsproducts/10?command=updateAccountingRule
        API->>Service: updateProductAccounting(productId)

        Admin->>API: {accountingRule: CASH_BASED,<br/>savingsReferenceAccountId: 2100,<br/>savingsControlAccountId: 1200,<br/>interestOnSavingsAccountId: 5100,<br/>incomeFromFeeAccountId: 4110,<br/>transfersSuspenseAccountId: 2200,<br/>overdraftPortfolioAccountId: 1150}

        Service->>Mapping: createSavingsMappings()

        Mapping->>DB: INSERT product_gl_mapping<br/>(product_type=SAVINGS,<br/>mappings...)

        Service-->>API: Mapping Complete
        API-->>Admin: Savings GL Configured
    end

    %% Configure Charge-Specific Mapping
    rect rgb(255, 250, 235)
        Note over Admin,Validator: CHARGE-SPECIFIC MAPPING
        Admin->>API: Map Specific Charge to GL<br/>{chargeId: 25,<br/>incomeAccountId: 4115}

        API->>Service: mapChargeToGLAccount()
        Service->>DB: INSERT product_gl_mapping<br/>(charge_id=25,<br/>glaccount_id=4115)

        Service-->>API: Charge Mapped
        API-->>Admin: Processing Fee → 4115
    end
```

## 3. Chart of Accounts Structure

```mermaid
graph TB
    subgraph "ASSETS (1000-1999)"
        Assets[1000 - ASSETS]

        subgraph "Current Assets"
            Cash[1010 - Cash and Bank]
            LoanPortfolio[1100 - Loan Portfolio]
            InterestRecv[1110 - Interest Receivable]
            FeesRecv[1120 - Fees Receivable]
            PenaltyRecv[1130 - Penalties Receivable]
            SavingsControl[1200 - Savings Control]
            ODPortfolio[1150 - Overdraft Portfolio]
        end

        subgraph "Fixed Assets"
            Property[1500 - Property & Equipment]
            Depreciation[1510 - Accumulated Depreciation]
        end
    end

    subgraph "LIABILITIES (2000-2999)"
        Liabilities[2000 - LIABILITIES]

        subgraph "Current Liabilities"
            SavingsDeposits[2100 - Savings Deposits]
            FDDeposits[2110 - Fixed Deposits]
            RDDeposits[2120 - Recurring Deposits]
            Suspense[2200 - Transfers Suspense]
            InterestPayable[2210 - Interest Payable]
            Overpayment[2300 - Overpayment Liability]
            LoanLossReserve[2400 - Loan Loss Reserve]
        end

        subgraph "Long-term Liabilities"
            BorrowedFunds[2500 - Borrowed Funds]
        end
    end

    subgraph "EQUITY (3000-3999)"
        Equity[3000 - EQUITY]

        Capital[3100 - Capital]
        Reserves[3200 - Reserves]
        RetainedEarnings[3300 - Retained Earnings]
        CurrentYearPL[3400 - Current Year P&L]
    end

    subgraph "INCOME (4000-4999)"
        Income[4000 - INCOME]

        InterestIncome[4100 - Interest Income on Loans]
        FeeIncome[4110 - Income from Fees]
        PenaltyIncome[4120 - Income from Penalties]
        OtherIncome[4200 - Other Income]
        RecoveryIncome[4130 - Income from Recovery]
    end

    subgraph "EXPENSES (5000-5999)"
        Expenses[5000 - EXPENSES]

        InterestExpense[5100 - Interest Expense on Deposits]
        ProvisionExpense[5110 - Provision Expense]
        LossExpense[5200 - Losses Written Off]
        OperatingExp[5300 - Operating Expenses]
        Salaries[5310 - Salaries & Wages]
        Depreciation2[5320 - Depreciation Expense]
    end

    Assets --> Cash
    Assets --> LoanPortfolio
    Assets --> InterestRecv
    Assets --> FeesRecv
    Assets --> Property

    Liabilities --> SavingsDeposits
    Liabilities --> FDDeposits
    Liabilities --> Suspense
    Liabilities --> LoanLossReserve

    Equity --> Capital
    Equity --> Reserves
    Equity --> RetainedEarnings

    Income --> InterestIncome
    Income --> FeeIncome
    Income --> PenaltyIncome

    Expenses --> InterestExpense
    Expenses --> ProvisionExpense
    Expenses --> LossExpense
    Expenses --> OperatingExp

    style Assets fill:#e8f5e9,stroke:#1b5e20
    style Liabilities fill:#ffebee,stroke:#c62828
    style Equity fill:#e3f2fd,stroke:#1565c0
    style Income fill:#fff9c4,stroke:#f57f17
    style Expenses fill:#fce4ec,stroke:#880e4f
```

## 4. Trial Balance Generation Process

```mermaid
sequenceDiagram
    participant Scheduler as Job Scheduler
    participant Job as Trial Balance Job
    participant Service as TB Service
    participant DB as Database
    participant Report as Report Generator

    %% Daily Trial Balance Update
    rect rgb(230, 240, 255)
        Note over Scheduler,Report: DAILY: UPDATE TRIAL BALANCE
        Scheduler->>Job: Trigger: UPDATE_TRIAL_BALANCE_DETAILS
        Job->>Service: updateTrialBalance(date)

        Service->>DB: SELECT DISTINCT transaction_date<br/>FROM acc_gl_journal_entry<br/>WHERE NOT IN m_trial_balance

        DB-->>Service: [2024-11-17, 2024-11-16, ...]

        loop For Each Missing Date
            Service->>DB: SELECT glaccount_id,<br/>SUM(CASE type=DEBIT THEN amount<br/>     ELSE -amount END) as net<br/>GROUP BY glaccount_id, office_id

            DB-->>Service: [{account: 1100, net: +50000},<br/>{account: 1010, net: -50000}, ...]

            loop For Each Account
                Service->>DB: SELECT closing_balance<br/>FROM m_trial_balance<br/>WHERE account=X<br/>ORDER BY date DESC LIMIT 1

                DB-->>Service: previous_balance: 100000

                Service->>Service: new_balance =<br/>previous_balance + net<br/>= 100000 + 50000 = 150000

                Service->>DB: INSERT m_trial_balance<br/>(office_id, glaccount_id,<br/>entry_date, amount=net,<br/>closing_balance=150000)
            end
        end

        Service-->>Job: Trial Balance Updated
    end

    %% Generate Trial Balance Report
    rect rgb(240, 255, 240)
        Note over Scheduler,Report: GENERATE TRIAL BALANCE REPORT
        Job->>Service: generateTrialBalanceReport(date)

        Service->>DB: SELECT gl.name, gl.code,<br/>tb.closing_balance<br/>FROM m_trial_balance tb<br/>JOIN acc_gl_account gl<br/>WHERE entry_date = '2024-11-17'<br/>ORDER BY gl.gl_code

        DB-->>Service: Trial Balance Data

        Service->>Service: calculateTotals()<br/>total_debits = SUM(debit_accounts)<br/>total_credits = SUM(credit_accounts)

        Service->>Service: validateBalance()<br/>IF total_debits != total_credits<br/>   THEN ERROR

        Service->>Report: generateReport(data)

        Report->>Report: formatTrialBalance()

        Report-->>Service: Trial Balance Report:<br/>GL Code | Account Name | Debit | Credit<br/>1010 | Cash | 500,000 | -<br/>1100 | Loans | 5,000,000 | -<br/>2100 | Deposits | - | 3,000,000<br/>4100 | Interest | - | 800,000<br/>...<br/>TOTAL: 5,700,000 | 5,700,000 ✓

        Service-->>Job: Report Generated
    end
```

## 5. GL Closure & Period Management

```mermaid
flowchart TD
    Start([Month/Year End]) --> InitClosure[Initiate GL Closure<br/>Close Period: 2024-10-31]

    InitClosure --> ValidateClosure{Validate<br/>Closure?}

    ValidateClosure -->|Invalid| RejectClosure[Reject Closure<br/>Reasons:<br/>- Pending transactions<br/>- Unbalanced entries<br/>- Unposted journals]

    ValidateClosure -->|Valid| CheckTrialBal[Verify Trial Balance<br/>Debits = Credits?]

    RejectClosure --> End([End - Rejected])

    CheckTrialBal -->|Unbalanced| FixEntries[Fix Unbalanced Entries<br/>Investigate discrepancies]
    CheckTrialBal -->|Balanced| GenerateReports[Generate Period Reports:<br/>1. Trial Balance<br/>2. Balance Sheet<br/>3. Income Statement<br/>4. Cash Flow]

    FixEntries --> CheckTrialBal

    GenerateReports --> ReviewReports{Review<br/>Reports}

    ReviewReports -->|Issues Found| CorrectEntries[Make Adjusting Entries<br/>Accruals, Deferrals]
    ReviewReports -->|All Good| ApproveClosure[Approve Closure]

    CorrectEntries --> CheckTrialBal

    ApproveClosure --> CreateClosure[Create GL Closure<br/>POST /glclosures<br/>{officeId, closingDate,<br/>comments}]

    CreateClosure --> InsertClosure[INSERT m_gl_closure<br/>(office_id, closing_date,<br/>comments, created_by)]

    InsertClosure --> LockPeriod[Lock Period<br/>No transactions allowed<br/>with date <= closing_date]

    LockPeriod --> TransferPL{Transfer P&L<br/>to Equity?}

    TransferPL -->|Yes| ClosePL[Close P&L Accounts<br/>DR: Income Accounts<br/>CR: Current Year P&L<br/>---<br/>DR: Current Year P&L<br/>CR: Expense Accounts]

    TransferPL -->|No| NotifyUsers

    ClosePL --> UpdateRetained[Transfer to Retained Earnings<br/>DR: Current Year P&L<br/>CR: Retained Earnings]

    UpdateRetained --> ZeroPL[Zero Out P&L Accounts<br/>Ready for new period]

    ZeroPL --> NotifyUsers[Notify Users:<br/>Period Closed<br/>New Period Open]

    NotifyUsers --> ArchiveReports[Archive Period Reports<br/>Store for audit/compliance]

    ArchiveReports --> Success([End - Closure Complete])

    style ApproveClosure fill:#e8f5e9
    style CreateClosure fill:#e3f2fd
    style LockPeriod fill:#fff3e0
    style RejectClosure fill:#ffcdd2
```

## 6. Loan Loss Provisioning Workflow

```mermaid
sequenceDiagram
    participant Scheduler as Job Scheduler
    participant Job as Provisioning Job
    participant Service as Provisioning Service
    participant Criteria as Provisioning Criteria
    participant DB as Database
    participant GL as GL Service

    %% Monthly Provisioning Calculation
    rect rgb(230, 245, 255)
        Note over Scheduler,GL: MONTHLY: CALCULATE PROVISIONING
        Scheduler->>Job: Trigger: GENERATE_LOANLOSS_PROVISIONING
        Job->>Service: generateProvisioning(date)

        Service->>Criteria: getProvisioningCriteria()
        Criteria-->>Service: Criteria:<br/>0-30 days: 0%<br/>31-90 days: 25%<br/>91-180 days: 50%<br/>180+ days: 100%

        Service->>DB: SELECT loan_id,<br/>product_id,<br/>principal_outstanding,<br/>days_overdue,<br/>office_id<br/>FROM m_loan<br/>WHERE status = ACTIVE

        DB-->>Service: Active Loans Data

        loop For Each Loan
            Service->>Service: calculateDaysOverdue()<br/>= today - last_payment_date

            Service->>Service: classifyLoan(days_overdue)

            alt 0-30 days: Standard
                Service->>Service: provision = 0%<br/>amount = 0
            else 31-90 days: Sub-standard
                Service->>Service: provision = 25%<br/>amount = outstanding × 0.25
            else 91-180 days: Doubtful
                Service->>Service: provision = 50%<br/>amount = outstanding × 0.50
            else 180+ days: Loss
                Service->>Service: provision = 100%<br/>amount = outstanding × 1.00
            end

            Service->>DB: INSERT m_loanproduct_provisioning_entry<br/>(loan_id, category,<br/>provision_amount,<br/>liability_account,<br/>expense_account)
        end

        Service->>DB: INSERT m_provisioning_entry<br/>(created_date, reversible)

        Service-->>Job: Provisioning Calculated
    end

    %% Create GL Entries for Provisioning
    rect rgb(240, 255, 240)
        Note over Scheduler,GL: CREATE GL ENTRIES
        Job->>Service: createProvisioningJournalEntries()

        Service->>DB: SELECT SUM(provision_amount)<br/>GROUP BY product_id, category

        DB-->>Service: Aggregated Amounts

        loop For Each Product/Category
            Service->>GL: createJournalEntry()

            GL->>DB: INSERT journal_entry<br/>DR: Provision Expense $X<br/>CR: Loan Loss Reserve $X

            Service->>DB: UPDATE m_provisioning_entry<br/>(journal_entry_id)
        end

        Service-->>Job: GL Entries Created
    end

    %% Reversal if Needed
    rect rgb(255, 250, 235)
        Note over Scheduler,GL: REVERSE PROVISIONING (if needed)
        Job->>Service: reverseProvisioning(entryId)

        Service->>DB: SELECT journal_entry_id<br/>FROM m_provisioning_entry<br/>WHERE id = entryId

        Service->>GL: reverseJournalEntry()

        GL->>DB: INSERT journal_entry<br/>DR: Loan Loss Reserve $X<br/>CR: Provision Expense $X<br/>(reversal flag = true)

        Service->>DB: UPDATE m_provisioning_entry<br/>(is_reversed = true)

        Service-->>Job: Provisioning Reversed
    end
```

## 7. Financial Statement Generation

```mermaid
graph TB
    Start([Generate Financial Statements]) --> GetData[Retrieve Trial Balance Data<br/>As of Date: 2024-10-31]

    GetData --> ClassifyAccounts{Classify<br/>Accounts}

    ClassifyAccounts --> Assets[ASSETS<br/>Type: 1]
    ClassifyAccounts --> Liabilities[LIABILITIES<br/>Type: 2]
    ClassifyAccounts --> Equity[EQUITY<br/>Type: 3]
    ClassifyAccounts --> Income[INCOME<br/>Type: 4]
    ClassifyAccounts --> Expenses[EXPENSES<br/>Type: 5]

    subgraph "Balance Sheet"
        Assets --> CalcAssets[Calculate Total Assets<br/>Cash: 500,000<br/>Loans: 5,000,000<br/>Receivables: 50,000<br/>TOTAL: 5,550,000]

        Liabilities --> CalcLiab[Calculate Total Liabilities<br/>Deposits: 3,000,000<br/>Payables: 100,000<br/>TOTAL: 3,100,000]

        Equity --> CalcEquity[Calculate Total Equity<br/>Capital: 1,000,000<br/>Reserves: 500,000<br/>Retained: 950,000<br/>TOTAL: 2,450,000]

        CalcAssets --> BalanceEq{Assets =<br/>Liab + Equity?}
        CalcLiab --> BalanceEq
        CalcEquity --> BalanceEq

        BalanceEq -->|Yes| GenBalSheet[Generate Balance Sheet<br/>ASSETS: 5,550,000<br/>LIABILITIES: 3,100,000<br/>EQUITY: 2,450,000<br/>✓ Balanced]
        BalanceEq -->|No| Error[ERROR: Unbalanced<br/>Investigate Discrepancy]
    end

    subgraph "Income Statement"
        Income --> CalcIncome[Calculate Total Income<br/>Interest Income: 800,000<br/>Fee Income: 50,000<br/>TOTAL: 850,000]

        Expenses --> CalcExpenses[Calculate Total Expenses<br/>Interest Expense: 150,000<br/>Operating: 200,000<br/>Provision: 50,000<br/>TOTAL: 400,000]

        CalcIncome --> CalcNetIncome[Calculate Net Income<br/>Revenue: 850,000<br/>Less Expenses: 400,000<br/>NET INCOME: 450,000]
        CalcExpenses --> CalcNetIncome

        CalcNetIncome --> GenPL[Generate Income Statement<br/>REVENUE: 850,000<br/>EXPENSES: 400,000<br/>NET INCOME: 450,000]
    end

    GenBalSheet --> ExportFormats{Export<br/>Format?}
    GenPL --> ExportFormats

    ExportFormats -->|PDF| GenPDF[Generate PDF Report<br/>Formatted Statements]
    ExportFormats -->|Excel| GenExcel[Generate Excel<br/>With Formulas]
    ExportFormats -->|JSON| GenJSON[Generate JSON<br/>For API consumption]

    GenPDF --> Distribute[Distribute to Stakeholders]
    GenExcel --> Distribute
    GenJSON --> Distribute

    Distribute --> Archive[Archive Statements<br/>Compliance & Audit]

    Archive --> End([End])
    Error --> End

    style GenBalSheet fill:#e8f5e9
    style GenPL fill:#fff3e0
    style Error fill:#ffcdd2
    style CalcNetIncome fill:#e3f2fd
```

## GL Account Type Summary

| Type | Code | Normal Balance | Examples |
|------|------|----------------|----------|
| ASSET | 1 | Debit | Cash, Loans, Receivables |
| LIABILITY | 2 | Credit | Deposits, Payables, Reserves |
| EQUITY | 3 | Credit | Capital, Retained Earnings |
| INCOME | 4 | Credit | Interest Income, Fee Income |
| EXPENSE | 5 | Debit | Interest Expense, Operating Expenses |

## Journal Entry Rules

| Transaction | Debit | Credit |
|-------------|-------|--------|
| Loan Disbursement | Loan Portfolio (Asset) | Cash/Fund Source (Asset) |
| Loan Repayment | Cash (Asset) | Loan Portfolio (Asset) + Interest Income (Revenue) |
| Savings Deposit | Cash (Asset) | Savings Deposits (Liability) |
| Savings Withdrawal | Savings Deposits (Liability) | Cash (Asset) |
| Interest Posting (Savings) | Interest Expense (Expense) | Savings Deposits (Liability) |
| Loan Write-off | Loss Expense (Expense) | Loan Portfolio (Asset) |
| Provisioning | Provision Expense (Expense) | Loan Loss Reserve (Liability) |
