# Apache Fineract - Complete Entity Catalog

## Overview

This document catalogs ALL 225+ database entities in the Apache Fineract system, organized by domain.

**Total Entities: 225**

---

## Entity Count by Domain

| Domain | Entity Count | ERD File |
|--------|-------------|----------|
| Core (Office, Staff, Client, Group) | 39 | fineract-erd-core-entities.puml |
| Loan Management | 45 | fineract-erd-loan-only.puml |
| Savings & Deposits | 42 | fineract-erd-savings-only.puml |
| Accounting & GL | 28 | fineract-erd-accounting-only.puml |
| Security & User Administration | 18 | fineract-erd-security.puml |
| Infrastructure & Configuration | 35 | fineract-erd-infrastructure.puml |
| Delinquency & Provisioning | 20 | fineract-erd-delinquency-provisioning.puml |
| Shares & Dividends | 12 | fineract-erd-shares.puml |
| Additional (Tax, Meeting, Interop, Teller, etc.) | 36 | fineract-erd-additional.puml |

---

## How to Use This Catalog

### Viewing Individual Domain ERDs

For better readability and manageability, entities are organized into domain-specific ERD files:

1. **Core Entities** (`fineract-erd-core-entities.puml`)
   - Office, Staff, Client, Group, ClientIdentifier, ClientAddress
   - Basic loan and savings entities
   - Infrastructure entities (Code, Calendar, Image, PaymentDetail)

2. **Loan Management** (`fineract-erd-loan-only.puml`)
   - Complete loan domain with all 43 transaction types
   - Loan products, schedules, charges, collateral, guarantors
   - Loan disbursement, repayment, write-off, recovery

3. **Savings & Deposits** (`fineract-erd-savings-only.puml`)
   - Savings accounts, transactions, charges
   - Fixed Deposit accounts with maturity details
   - Recurring Deposit accounts with installments
   - Interest rate charts and slabs

4. **Accounting & GL** (`fineract-erd-accounting-only.puml`)
   - Chart of accounts (GL accounts)
   - Journal entries and automatic posting
   - Product-to-GL account mappings
   - GL closures and accounting rules

5. **Security & User Administration** (`fineract-erd-security.puml`)
   - AppUser, Role, Permission (RBAC model)
   - Two-Factor Authentication
   - API Keys and OAuth2
   - Maker-Checker command source
   - Security audit and login attempts

6. **Infrastructure & Configuration** (`fineract-erd-infrastructure.puml`)
   - Batch jobs and schedulers
   - Webhooks and integration
   - Notifications (SMS, Email, In-app)
   - Reports and templates
   - Global configuration
   - Survey and scoring (PPI)

7. **Delinquency & Provisioning** (`fineract-erd-delinquency-provisioning.puml`)
   - Delinquency buckets and ranges
   - Loan arrears aging
   - Provisioning criteria and calculations
   - Credit bureau integration
   - NPA (Non-Performing Assets)
   - Write-off and recovery

8. **Shares & Dividends** (`fineract-erd-shares.puml`)
   - Share products and accounts
   - Share transactions (purchase/redemption)
   - Dividend payouts
   - Market prices

9. **Additional Entities** (`fineract-erd-additional.puml`)
   - Tax (TaxGroup, TaxComponent, Withholding)
   - Meeting and attendance
   - Interoperability (Mojaloop integration)
   - Account transfers and standing instructions
   - Teller and cashier operations
   - Notes and documents
   - Address management
   - Floating rates
   - Interest rate charts
   - Product mix restrictions

---

## Complete Entity List

### 1. ORGANIZATIONAL ENTITIES (5)

| Entity | Table Name | Description |
|--------|------------|-------------|
| Office | m_office | Branch offices in hierarchy |
| Staff | m_staff | Staff members (loan officers, etc.) |
| Holiday | m_holiday | Office holidays |
| OfficeTransaction | m_office_transaction | Cash transfers between offices |
| WorkingDays | m_working_days | Working days configuration |

### 2. CLIENT & GROUP ENTITIES (15)

| Entity | Table Name | Description |
|--------|------------|-------------|
| Client | m_client | Customer/borrower information |
| ClientIdentifier | m_client_identifier | ID documents (passport, national ID) |
| ClientAddress | m_client_address | Client addresses |
| ClientFamilyMembers | m_family_details | Family member information |
| ClientNonPerson | m_client_non_person | Business client details |
| ClientTransaction | m_client_transaction | Client-level transactions |
| ClientCharge | m_client_charge | Charges applied to clients |
| ClientChargePaidBy | m_client_charge_paid_by | Charge payment tracking |
| ClientTransferDetails | m_client_transfer_details | Client office transfers |
| Group | m_group | Groups and centers |
| GroupLevel | m_group_level | Group hierarchy levels |
| GroupClient | m_group_client | Group membership |
| GroupSavingsIndividualMonitoring | m_gsim_accounts | Group savings monitoring |
| Meeting | m_meeting | Group meetings |
| ClientAttendance | m_meeting_attendance | Meeting attendance |

### 3. LOAN ENTITIES (45)

| Entity | Table Name | Description |
|--------|------------|-------------|
| LoanProduct | m_product_loan | Loan product configuration |
| Loan | m_loan | Loan accounts |
| LoanTransaction | m_loan_transaction | All loan transactions (43 types) |
| LoanRepaymentScheduleInstallment | m_loan_repayment_schedule | Repayment schedule |
| LoanCharge | m_loan_charge | Charges applied to loans |
| LoanCollateral | m_loan_collateral | Collateral for loans |
| LoanOfficerAssignmentHistory | m_loan_officer_assignment_history | Loan officer changes |
| Guarantor | m_guarantor | Loan guarantors |
| GuarantorFundingDetails | m_guarantor_funding_details | Guarantor funding |
| GuarantorFundingTransaction | m_guarantor_transaction | Guarantor transactions |
| LoanArrearsAging | m_loan_arrears_aging | Arrears tracking |
| LoanDisbursementDetails | m_loan_disbursement_detail | Disbursement details |
| LoanTopupDetails | m_loan_topup | Top-up loan details |
| LoanTrancheDetail | m_loan_tranche_disbursement_charge | Tranche disbursements |
| LoanInterestRecalculationDetails | m_loan_recalculation_details | Interest recalculation config |
| LoanTermVariations | m_loan_term_variations | Loan term modifications |
| LoanRepaymentScheduleHistory | m_loan_repayment_schedule_history | Schedule change history |
| LoanProductProvisioningCriteria | m_loanproduct_provisioning_criteria | Provisioning rules |
| LoanLossProvision | m_loan_loss_provision | Provisioning amounts |
| LoanDelinquencyTags | m_loan_delinquency_tags | Delinquency classification |
| LoanDelinquencyAction | m_loan_delinquency_action | Automated delinquency actions |
| LoanWriteOff | m_loan_write_off | Write-off records |
| LoanRecovery | m_loan_recovery_repayment | Post-write-off recoveries |
| LoanNPAStatus | m_loan_npa_status | NPA classification |
| LoanCreditCheck | m_loan_credit_check | Credit bureau checks |
| LoanProductFloatingRates | m_product_loan_floating_rates | Floating rate config |
| LoanFloatingRateHistory | m_loan_floating_rate_history | Rate change history |
| ...and 18 more loan-related entities | | |

### 4. SAVINGS & DEPOSIT ENTITIES (42)

| Entity | Table Name | Description |
|--------|------------|-------------|
| SavingsProduct | m_savings_product | Savings product configuration |
| SavingsAccount | m_savings_account | Savings accounts |
| SavingsAccountTransaction | m_savings_account_transaction | Savings transactions |
| SavingsAccountCharge | m_savings_account_charge | Savings charges |
| SavingsAccountChargePaidBy | m_savings_charge_paid_by | Charge payments |
| SavingsAccountTransactionTaxDetails | m_savings_account_transaction_tax_details | Tax withholding |
| SavingsOfficerAssignmentHistory | m_savings_officer_assignment_history | Officer changes |
| FixedDepositProduct | m_deposit_product_term | FD product configuration |
| FixedDepositAccount | m_deposit_account_term | Fixed deposit accounts |
| RecurringDepositProduct | m_deposit_product_recurring | RD product configuration |
| RecurringDepositAccount | m_deposit_account_recurring | Recurring deposit accounts |
| RecurringDepositScheduleInstallment | m_mandatory_savings_schedule | RD installment schedule |
| DepositAccountOnHoldTransaction | m_deposit_account_on_hold_transaction | Fund holds |
| DepositAccountTermAndPreClosure | m_deposit_account_term_and_preclosure | Maturity and closure terms |
| DepositAccountRecurringDetail | m_deposit_account_recurring_detail | RD deposit details |
| DepositAccountInterestRateChart | m_savings_account_interest_rate_chart | Account-specific interest rates |
| DepositAccountInterestRateChartSlabs | m_savings_account_interest_rate_slab | Interest rate tiers |
| DepositProductRecurringDetail | m_deposit_product_recurring_detail | RD product configuration |
| DepositProductTermAndPreClosure | m_deposit_product_term_and_preclosure | Product maturity terms |
| InterestRateChart | m_interest_rate_chart | Interest rate chart definition |
| InterestRateChartSlab | m_interest_rate_slab | Interest rate slabs/tiers |
| InterestIncentives | m_interest_incentives | Interest incentive conditions |
| DepositAccountInterestIncentives | m_savings_account_interest_incentives | Account incentives |
| ...and 19 more savings-related entities | | |

### 5. ACCOUNTING & GL ENTITIES (28)

| Entity | Table Name | Description |
|--------|------------|-------------|
| GLAccount | acc_gl_account | Chart of accounts |
| JournalEntry | acc_gl_journal_entry | All journal entries |
| GLClosure | acc_gl_closure | Period-end closures |
| AccountingRule | acc_accounting_rule | Accounting rule definitions |
| ProductToGLAccountMapping | acc_product_mapping | Product GL mappings |
| FinancialActivityAccount | acc_financial_activity_account | Activity-to-GL mappings |
| ProvisioningCategory | m_provisioning_category | Provisioning categories |
| ProvisioningCriteria | m_provisioning_criteria | Provisioning rules |
| ProvisioningCriteriaDefinition | m_provisioning_criteria_definition | Provisioning definitions |
| ProvisioningEntry | m_provisioning_entry | Provisioning journal entries |
| LoanProductProvisioningEntry | m_loanproduct_provisioning_entry | Product provisioning details |
| JournalEntryAggregationTracking | m_journal_entry_aggregation_tracking | Aggregation tracking |
| JournalEntrySummary | m_journal_entry_summary | Journal entry summaries |
| OfficeOpeningBalances | m_office_opening_balance | Opening balances |
| TrialBalance | (generated) | Trial balance report |
| ...and 13 more accounting entities | | |

### 6. SECURITY & USER ADMIN ENTITIES (18)

| Entity | Table Name | Description |
|--------|------------|-------------|
| AppUser | m_appuser | System users |
| Role | m_role | User roles |
| Permission | m_permission | Granular permissions |
| AppUserRole | m_appuser_role | User-role assignments |
| RolePermission | m_role_permission | Role-permission mappings |
| AppUserPreviousPassword | m_appuser_previous_password | Password history |
| AppUserClientMapping | m_selfservice_user_client_mapping | Self-service user mappings |
| PasswordValidationPolicy | m_password_validation_policy | Password policies |
| TwoFactorConfiguration | twofactor_configuration | 2FA configuration |
| TFAccessToken | m_twofactor_access_token | 2FA tokens |
| ApiKey | m_api_key | API key authentication |
| OAuth2Client | oauth_client_details | OAuth2 clients |
| LoginAttempt | m_login_attempt | Login attempt tracking |
| UserSession | m_user_session | Active user sessions |
| SecurityAudit | m_security_audit | Security audit trail |
| CommandSource | m_portfolio_command_source | Maker-checker commands |
| EntityAccess | m_entity_to_entity_access | Entity access control |
| EntityToEntityMapping | m_entity_to_entity_mapping | Entity mappings |

### 7. INFRASTRUCTURE & CONFIGURATION ENTITIES (35)

| Entity | Table Name | Description |
|--------|------------|-------------|
| ScheduledJobDetail | job | Scheduled batch jobs |
| ScheduledJobRunHistory | job_run_history | Job execution history |
| JobParameter | job_parameters | Job parameters |
| SchedulerDetail | scheduler_detail | Scheduler configuration |
| Hook | m_hook | Webhook definitions |
| HookTemplate | m_hook_templates | Hook templates |
| HookConfiguration | m_hook_configuration | Hook settings |
| HookResource | m_hook_registered_events | Hook event registrations |
| HookHistory | m_hook_history | Hook delivery history |
| Schema | m_hook_schema | Hook schema definitions |
| Notification | notification | In-app notifications |
| NotificationMapper | notification_mapper | Notification templates |
| SmsMessage | sms_messages_outbound | Outbound SMS |
| EmailMessage | m_email_outbound | Outbound emails |
| NotificationCampaign | m_notification_campaign | Notification campaigns |
| Report | m_report | Stretchy reports |
| ReportParameter | m_report_parameter | Report parameters |
| ReportMailingJob | m_report_mailing_job | Scheduled report emails |
| ReportMailingJobRunHistory | m_report_mailing_job_run_history | Report mailing history |
| Template | m_template | Document templates |
| TemplateMapper | m_template_entity_mapping | Template mappings |
| GlobalConfiguration | c_configuration | System configuration |
| ExternalService | m_external_service | External service integrations |
| ExternalServiceProperties | m_external_service_properties | Service properties |
| BusinessDate | m_business_date | Business date management |
| CacheRecord | m_cache | Cache configuration |
| RegisteredTable | x_registered_table | Data table registry |
| TableMetadata | x_table_metadata | Data table metadata |
| DataTable | m_datatable | Custom data tables |
| Survey | ppi_surveys | PPI surveys |
| Component | ppi_components | Survey components |
| Question | ppi_questions | Survey questions |
| Response | ppi_responses | Survey responses |
| LookupTable | ppi_likelihoods | Scoring lookup tables |
| Audit | m_audit | System audit log |

### 8. DELINQUENCY & PROVISIONING ENTITIES (20)

| Entity | Table Name | Description |
|--------|------------|-------------|
| DelinquencyBucket | m_delinquency_bucket | Delinquency bucket definitions |
| DelinquencyRange | m_delinquency_range | Delinquency ranges |
| DelinquencyBucketMappings | m_delinquency_bucket_mappings | Bucket-range mappings |
| LoanDelinquencyTags | m_loan_delinquency_tags | Loan delinquency tags |
| LoanDelinquencyAction | m_loan_delinquency_action | Automated delinquency actions |
| LoanArrearsAging | m_loan_arrears_aging | Arrears aging calculation |
| ProvisioningCategory | m_provisioning_category | Provisioning categories |
| ProvisioningCriteria | m_provisioning_criteria | Provisioning criteria |
| ProvisioningCriteriaDefinition | m_provisioning_criteria_definition | Criteria definitions |
| LoanProductProvisioningCriteria | m_loanproduct_provisioning_criteria | Product provisioning |
| ProvisioningEntry | m_provisioning_entry | Provisioning entries |
| LoanProductProvisioningEntry | m_loanproduct_provisioning_entry | Product provisioning details |
| LoanLossProvision | m_loan_loss_provision | Loan loss provisions |
| ImpairmentTracking | m_impairment_tracking | IFRS 9 impairment |
| CreditBureau | m_organisation_creditbureau | Credit bureau configuration |
| CreditBureauConfiguration | m_creditbureau_configuration | Bureau settings |
| CreditBureauLoanProductMapping | m_creditbureau_loanproduct_mapping | Product-bureau mappings |
| CreditBureauReport | m_creditbureau_report | Credit reports |
| NPAConfiguration | m_npa_configuration | NPA configuration |
| PortfolioAtRisk | m_portfolio_at_risk | PAR metrics |

### 9. SHARES & DIVIDENDS ENTITIES (12)

| Entity | Table Name | Description |
|--------|------------|-------------|
| ShareProduct | m_share_product | Share product configuration |
| ShareProductMarketPrice | m_share_product_market_price | Historical market prices |
| ShareProductDividendPayOutDetails | m_share_product_dividend_pay_out | Dividend declarations |
| ShareAccount | m_share_account | Share accounts |
| ShareAccountTransaction | m_share_account_transactions | Share transactions |
| ShareAccountCharge | m_share_account_charge | Share charges |
| ShareAccountChargePaidBy | m_share_account_charge_paid_by | Charge payments |
| ShareAccountDividendDetails | m_share_account_dividend_details | Dividend distributions |
| SharePurchasePeriod | m_share_purchase_period | Purchase windows |

### 10. ADDITIONAL ENTITIES (36)

| Entity | Table Name | Description |
|--------|------------|-------------|
| TaxGroup | m_tax_group | Tax group definitions |
| TaxComponent | m_tax_component | Individual tax components |
| TaxGroupMappings | m_tax_group_mappings | Group-component mappings |
| TaxComponentHistory | m_tax_component_history | Tax rate history |
| Meeting | m_meeting | Group meetings |
| ClientAttendance | m_meeting_attendance | Meeting attendance |
| CalendarInstance | m_calendar_instance | Calendar instances |
| Calendar | m_calendar | Calendar definitions |
| InteropIdentifier | m_interop_identifier | Interop identifiers |
| InteropKYCData | m_interop_kyc_data | Interop KYC data |
| AccountTransferDetails | m_account_transfer_details | Transfer definitions |
| AccountTransferTransaction | m_account_transfer_transaction | Transfer transactions |
| AccountTransferStandingInstruction | m_account_transfer_standing_instructions | Standing instructions |
| AccountAssociations | m_portfolio_account_associations | Account associations |
| Teller | m_tellers | Teller definitions |
| Cashier | m_cashiers | Cashier assignments |
| CashierTransaction | m_cashier_transactions | Cashier transactions |
| TellerCashManagement | m_teller_cash_management | Cash management |
| Note | m_note | Notes/comments |
| Document | m_document | Document storage |
| Address | m_address | Address records |
| FieldConfiguration | m_field_configuration | Field validation config |
| FloatingRate | m_floating_rates | Floating rate definitions |
| FloatingRatePeriod | m_floating_rates_periods | Rate periods |
| ProductMix | m_product_mix | Product restrictions |
| SelfBeneficiariesTPT | m_selfservice_beneficiaries_tpt | Self-service beneficiaries |
| Charge | m_charge | Charge definitions |
| Fund | m_fund | Fund sources |
| PaymentType | m_payment_type | Payment types |
| PaymentDetail | m_payment_detail | Payment details |
| Code | m_code | Code/value list definitions |
| CodeValue | m_code_value | Code values |
| Image | m_image | Image storage |
| ...and 3 more entities | | |

---

## Viewing Recommendations

### For Developers
1. Start with **Core Entities** to understand the basic structure
2. Deep dive into **Loan** and **Savings** entities for business logic
3. Study **Accounting** entities for GL integration
4. Review **Security** entities for authentication/authorization

### For Business Analysts
1. Focus on **Loan**, **Savings**, and **Shares** workflow entities
2. Review **Delinquency & Provisioning** for risk management
3. Study **Tax** and **Meeting** entities for operational processes

### For Integration Developers
1. Study **Infrastructure** entities (Hooks, Notifications, Reports)
2. Review **Interop** entities for mobile money integration
3. Understand **Account Transfer** entities for payment flows
4. Review **API Key** and **OAuth2** entities for authentication

### For DevOps/Database Admins
1. Review **Job** and **Scheduler** entities for batch processing
2. Study **Business Date** and **Global Configuration** entities
3. Review **Audit** and **Security Audit** entities for compliance
4. Understand **Data Table** entities for custom fields

---

## Cross-Domain Relationships

### Key Foreign Key Relationships

- **Office** → Client, Group, Loan, Savings, Staff, Teller
- **Client** → Loan, Savings, ShareAccount, ClientCharge, ClientTransaction
- **Group** → Loan, Savings, Meeting, GroupClient
- **Loan** → LoanTransaction, LoanSchedule, LoanCharge, Guarantor, Collateral
- **Savings** → SavingsTransaction, SavingsCharge, FixedDeposit, RecurringDeposit
- **LoanTransaction** → JournalEntry (automatic GL posting)
- **SavingsTransaction** → JournalEntry (automatic GL posting)
- **AppUser** → Office, Staff, Role, CommandSource (maker/checker)
- **LoanProduct** → ProductToGLAccountMapping, LoanProductProvisioningCriteria
- **SavingsProduct** → ProductToGLAccountMapping, InterestRateChart

---

## Database Size Considerations

With 225+ tables, a typical Fineract production database contains:
- **10-50 GB** for small microfinance institutions (< 10,000 clients)
- **50-200 GB** for medium institutions (10,000 - 100,000 clients)
- **200 GB - 1 TB+** for large institutions (100,000+ clients)

Largest tables by row count:
1. **acc_gl_journal_entry** - Millions of rows (every transaction generates 2+ entries)
2. **m_loan_transaction** - Hundreds of thousands to millions
3. **m_savings_account_transaction** - Hundreds of thousands to millions
4. **m_audit** - Millions (every API call logged)
5. **notification** - Hundreds of thousands

---

## Documentation Structure

```
Learning/
├── APACHE_FINERACT_COMPLETE_GUIDE.md      # Comprehensive text guide
├── COMPLETE_ENTITY_CATALOG.md             # This file
├── ERD_DIAGRAMS_README.md                 # PlantUML ERD usage guide
├── MERMAID_DIAGRAMS_README.md             # Mermaid workflow usage guide
│
├── fineract-erd-core-entities.puml        # Core entities (Office, Client, Group)
├── fineract-erd-loan-only.puml            # Loan domain
├── fineract-erd-savings-only.puml         # Savings domain
├── fineract-erd-accounting-only.puml      # Accounting domain
├── fineract-erd-security.puml             # Security & user admin
├── fineract-erd-infrastructure.puml       # Infrastructure & config
├── fineract-erd-delinquency-provisioning.puml  # Delinquency & risk
├── fineract-erd-shares.puml               # Shares & dividends
├── fineract-erd-additional.puml           # Additional entities
│
└── mermaid-01-system-architecture.md through mermaid-09-security-auth-workflow.md
```

---

**Last Updated:** 2025-11-24  
**Apache Fineract Version:** 1.x  
**Total Entities Documented:** 225

