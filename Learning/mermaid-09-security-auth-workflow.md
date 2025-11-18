# Apache Fineract - Security & Authorization Architecture

## 1. Authentication Flow (Multi-Tenant)

```mermaid
sequenceDiagram
    participant Client
    participant API as REST API
    participant AuthFilter as Authentication Filter
    participant TenantResolver as Tenant Resolver
    participant MasterDB as Master DB<br/>(Tenant Registry)
    participant UserService as User Service
    participant TenantDB as Tenant DB
    participant SecurityContext as Security Context

    Client->>API: POST /authentication<br/>Headers:<br/>- Fineract-Platform-TenantId: acme<br/>Body:<br/>- username: john.doe<br/>- password: ********

    API->>AuthFilter: Process Authentication

    AuthFilter->>TenantResolver: Extract Tenant ID from Header
    TenantResolver-->>AuthFilter: Tenant: "acme"

    AuthFilter->>MasterDB: Query Tenant Configuration<br/>SELECT * FROM tenants<br/>WHERE identifier = 'acme'

    alt Tenant Not Found
        MasterDB-->>AuthFilter: No Results
        AuthFilter-->>Client: HTTP 401 Unauthorized<br/>Error: Invalid tenant
    else Tenant Found
        MasterDB-->>AuthFilter: Tenant Config:<br/>- schema_name: acme_db<br/>- connection_params<br/>- timezone

        AuthFilter->>AuthFilter: Switch DataSource to Tenant DB

        AuthFilter->>UserService: Authenticate User<br/>username: john.doe<br/>password: ********

        UserService->>TenantDB: SELECT * FROM m_appuser<br/>WHERE username = 'john.doe'<br/>AND is_deleted = 0

        alt User Not Found
            TenantDB-->>UserService: No Results
            UserService-->>AuthFilter: Authentication Failed
            AuthFilter-->>Client: HTTP 401 Unauthorized<br/>Invalid credentials
        else User Found
            TenantDB-->>UserService: User Record

            UserService->>UserService: Verify Password Hash<br/>BCrypt.checkpw(password, hash)

            alt Password Mismatch
                UserService-->>AuthFilter: Authentication Failed
                UserService->>TenantDB: Log Failed Attempt
                AuthFilter-->>Client: HTTP 401 Unauthorized
            else Password Match
                UserService->>TenantDB: Load User Roles & Permissions<br/>JOIN m_appuser_role<br/>JOIN m_role<br/>JOIN m_role_permission

                TenantDB-->>UserService: User Details:<br/>- ID: 123<br/>- Roles: [Loan Officer, Maker]<br/>- Office: Branch A<br/>- Permissions: [...]

                UserService->>UserService: Generate JWT Token<br/>Payload:<br/>- user_id<br/>- tenant_id<br/>- office_id<br/>- roles<br/>- exp: 24 hours

                UserService-->>AuthFilter: Authentication Success<br/>JWT Token

                AuthFilter->>SecurityContext: Set Authentication<br/>- Principal: john.doe<br/>- Tenant: acme<br/>- Office: Branch A<br/>- Roles: [...]

                AuthFilter-->>Client: HTTP 200 OK<br/>Response:<br/>- token: eyJhbG...<br/>- userId: 123<br/>- officeId: 5
            end
        end
    end
```

## 2. Authorization Flow (RBAC + Data Scoping)

```mermaid
flowchart TB
    Start([API Request:<br/>POST /loans/{id}/approve]) --> ExtractToken[Extract JWT Token<br/>from Authorization Header]

    ExtractToken --> ValidateToken{Valid JWT?}

    ValidateToken -->|No| Unauthorized[HTTP 401<br/>Unauthorized]
    ValidateToken -->|Yes| ParseToken[Parse Token Claims:<br/>- user_id<br/>- tenant_id<br/>- office_id<br/>- roles]

    ParseToken --> LoadPerms[Load User Permissions<br/>from Database]

    LoadPerms --> CheckPermission{Has Permission:<br/>APPROVE_LOAN?}

    CheckPermission -->|No| Forbidden[HTTP 403<br/>Forbidden]
    CheckPermission -->|Yes| LoadResource[Load Loan Record<br/>from Database]

    LoadResource --> CheckOffice{Loan Office<br/>in User's<br/>Office Hierarchy?}

    CheckOffice -->|No| ForbiddenOffice[HTTP 403<br/>Out of Office Scope]
    CheckOffice -->|Yes| CheckStatus{Loan Status<br/>Allows Approval?}

    CheckStatus -->|No| BadRequest[HTTP 400<br/>Invalid State]
    CheckStatus -->|Yes| Execute[Execute Business Logic:<br/>Approve Loan]

    Execute --> Success[HTTP 200 OK<br/>Loan Approved]

    Unauthorized --> End([End])
    Forbidden --> End
    ForbiddenOffice --> End
    BadRequest --> End
    Success --> End

    style Start fill:#e1f5ff
    style Success fill:#c8e6c9
    style Unauthorized fill:#ffcdd2
    style Forbidden fill:#ffcdd2
    style ForbiddenOffice fill:#ffcdd2
    style BadRequest fill:#fff9c4
```

## 3. Role-Based Access Control (RBAC) Model

```mermaid
erDiagram
    M_APPUSER {
        BIGINT id PK
        VARCHAR username UK
        VARCHAR password "BCrypt hash"
        VARCHAR firstname
        VARCHAR lastname
        VARCHAR email
        BIGINT office_id FK "Home office"
        BIGINT staff_id FK "Optional staff link"
        TINYINT is_deleted
        TINYINT is_self_service_user
    }

    M_ROLE {
        BIGINT id PK
        VARCHAR name UK "Loan Officer, Manager, etc"
        VARCHAR description
        TINYINT is_disabled
    }

    M_APPUSER_ROLE {
        BIGINT appuser_id FK
        BIGINT role_id FK
    }

    M_PERMISSION {
        BIGINT id PK
        VARCHAR grouping "portfolio, organisation, etc"
        VARCHAR code UK "APPROVE_LOAN, READ_CLIENT"
        VARCHAR entity_name "Loan, Client, etc"
        VARCHAR action_name "CREATE, READ, UPDATE, DELETE"
        TINYINT can_maker_checker "Requires approval?"
    }

    M_ROLE_PERMISSION {
        BIGINT role_id FK
        BIGINT permission_id FK
    }

    M_OFFICE {
        BIGINT id PK
        VARCHAR name
        BIGINT parent_id FK "Office hierarchy"
        VARCHAR hierarchy "Path in tree"
        DATE opening_date
    }

    M_APPUSER }o--|| M_OFFICE : "belongs to"
    M_APPUSER ||--o{ M_APPUSER_ROLE : "has roles"
    M_APPUSER_ROLE }o--|| M_ROLE : "role"
    M_ROLE ||--o{ M_ROLE_PERMISSION : "has permissions"
    M_ROLE_PERMISSION }o--|| M_PERMISSION : "permission"
    M_OFFICE ||--o{ M_OFFICE : "parent-child"
```

## 4. Office Hierarchy & Data Scoping

```mermaid
graph TB
    Head[Head Office<br/>ID: 1<br/>Hierarchy: .1.]

    Regional1[Regional Office - North<br/>ID: 2<br/>Hierarchy: .1.2.]
    Regional2[Regional Office - South<br/>ID: 3<br/>Hierarchy: .1.3.]

    Branch1[Branch A<br/>ID: 4<br/>Hierarchy: .1.2.4.]
    Branch2[Branch B<br/>ID: 5<br/>Hierarchy: .1.2.5.]
    Branch3[Branch C<br/>ID: 6<br/>Hierarchy: .1.3.6.]
    Branch4[Branch D<br/>ID: 7<br/>Hierarchy: .1.3.7.]

    Head --> Regional1
    Head --> Regional2

    Regional1 --> Branch1
    Regional1 --> Branch2

    Regional2 --> Branch3
    Regional2 --> Branch4

    User1[User: John<br/>Office: Branch A<br/>Can Access: Branch A only]
    User2[User: Jane<br/>Office: Regional North<br/>Can Access: Regional + Branch A, B]
    User3[User: Admin<br/>Office: Head Office<br/>Can Access: All Offices]

    Branch1 -.->|Scoped to| User1
    Regional1 -.->|Scoped to| User2
    Head -.->|Scoped to| User3

    style Head fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style Regional1 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style Regional2 fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    style Branch1 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Branch2 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Branch3 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    style Branch4 fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
```

## 5. Permission Grouping & Hierarchy

```mermaid
graph TB
    subgraph "Portfolio Permissions"
        LoanPerms[Loan Permissions<br/>- READ_LOAN<br/>- CREATE_LOAN<br/>- APPROVE_LOAN<br/>- DISBURSE_LOAN<br/>- REPAYMENT_LOAN<br/>- WRITEOFF_LOAN]

        SavingsPerms[Savings Permissions<br/>- READ_SAVINGSACCOUNT<br/>- CREATE_SAVINGSACCOUNT<br/>- APPROVE_SAVINGSACCOUNT<br/>- ACTIVATE_SAVINGSACCOUNT<br/>- DEPOSIT_SAVINGSACCOUNT<br/>- WITHDRAWAL_SAVINGSACCOUNT]

        ClientPerms[Client Permissions<br/>- READ_CLIENT<br/>- CREATE_CLIENT<br/>- UPDATE_CLIENT<br/>- DELETE_CLIENT<br/>- ACTIVATE_CLIENT<br/>- CLOSE_CLIENT]
    end

    subgraph "Organisation Permissions"
        OfficePerms[Office Permissions<br/>- READ_OFFICE<br/>- CREATE_OFFICE<br/>- UPDATE_OFFICE]

        UserPerms[User Permissions<br/>- READ_USER<br/>- CREATE_USER<br/>- UPDATE_USER<br/>- DELETE_USER]

        RolePerms[Role Permissions<br/>- READ_ROLE<br/>- CREATE_ROLE<br/>- UPDATE_ROLE<br/>- DELETE_ROLE]
    end

    subgraph "Accounting Permissions"
        GLPerms[GL Permissions<br/>- READ_GLACCOUNT<br/>- CREATE_GLACCOUNT<br/>- UPDATE_GLACCOUNT<br/>- DELETE_GLACCOUNT]

        JournalPerms[Journal Permissions<br/>- READ_JOURNALENTRY<br/>- CREATE_JOURNALENTRY<br/>- REVERSE_JOURNALENTRY]

        ReportPerms[Report Permissions<br/>- READ_RUNREPORT<br/>- READ_ACCOUNTING]
    end

    subgraph "System Permissions"
        ConfigPerms[Configuration<br/>- READ_CONFIGURATION<br/>- UPDATE_CONFIGURATION]

        CodePerms[Code/Value<br/>- READ_CODE<br/>- CREATE_CODE<br/>- UPDATE_CODE]

        HookPerms[Hooks<br/>- READ_HOOK<br/>- CREATE_HOOK<br/>- UPDATE_HOOK]
    end

    subgraph "Special Permissions"
        MakerPerms[Maker Permissions<br/>- *_MAKER suffix<br/>Requires checker approval]

        CheckerPerms[Checker Permissions<br/>- *_CHECKER suffix<br/>Can approve maker requests]

        SuperPerms[Super User<br/>- ALL_FUNCTIONS<br/>- ALL_FUNCTIONS_READ]
    end

    LoanPerms --> MakerPerms
    SavingsPerms --> MakerPerms
    ClientPerms --> MakerPerms

    MakerPerms -.->|Approved by| CheckerPerms

    style SuperPerms fill:#ffccbc,stroke:#bf360c,stroke-width:3px
    style MakerPerms fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style CheckerPerms fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
```

## 6. Self-Service User Flow (Mobile/Web Portal)

```mermaid
sequenceDiagram
    participant Customer
    participant Portal as Customer Portal
    participant API as Public API
    participant AuthService as Auth Service
    participant DB as Database
    participant OTPService as OTP Service

    Customer->>Portal: Register Account
    Portal->>API: POST /self/registration<br/>- phone_number<br/>- email<br/>- national_id

    API->>DB: Check if Client Exists<br/>WHERE national_id = ?

    alt Client Not Found
        API-->>Portal: Error: No client record found
    else Client Found
        DB-->>API: Client Record

        API->>OTPService: Generate OTP
        OTPService-->>API: OTP: 123456

        API->>DB: Store OTP<br/>- client_id<br/>- otp_hash<br/>- expires_at (5 min)

        API->>OTPService: Send OTP via SMS/Email
        OTPService->>Customer: SMS: "Your OTP is 123456"

        API-->>Portal: OTP Sent

        Portal-->>Customer: Enter OTP

        Customer->>Portal: Submit OTP: 123456
        Portal->>API: POST /self/registration/verify<br/>- otp: 123456

        API->>DB: Verify OTP<br/>WHERE client_id = ?<br/>AND otp_hash = ?<br/>AND expires_at > NOW()

        alt OTP Valid
            API->>DB: Create Self-Service User<br/>- client_id<br/>- username<br/>- password_hash<br/>- is_self_service_user = 1

            API->>DB: Assign Self-Service Role<br/>INSERT INTO m_appuser_role

            API-->>Portal: Registration Complete

            Portal-->>Customer: Account Created<br/>You can now login

            Customer->>Portal: Login<br/>- username<br/>- password

            Portal->>API: POST /self/authentication

            API->>AuthService: Authenticate Self-Service User

            AuthService->>DB: Validate Credentials

            AuthService->>AuthService: Generate JWT Token<br/>with limited permissions

            AuthService-->>Portal: JWT Token

            Portal->>API: GET /self/loans<br/>Authorization: Bearer {token}

            API->>API: Validate Token & Permissions

            API->>DB: SELECT loans<br/>WHERE client_id = {from_token}

            DB-->>API: Customer's Loans

            API-->>Portal: Loan Details

            Portal-->>Customer: Show:<br/>- Loan Balance<br/>- Next Payment Due<br/>- Transaction History
        else OTP Invalid
            API-->>Portal: Error: Invalid or expired OTP
        end
    end
```

## 7. API Key Authentication (Third-Party Integration)

```mermaid
sequenceDiagram
    participant ThirdParty as Third-party App
    participant Admin
    participant API as Fineract API
    participant KeyService as API Key Service
    participant DB as Database

    Admin->>API: POST /api-keys<br/>Create API Key for Third-party

    API->>KeyService: Generate API Key

    KeyService->>KeyService: Generate Secure Key:<br/>- 32 random bytes<br/>- Base64 encoded

    KeyService->>DB: INSERT INTO m_api_keys<br/>- key_hash (SHA-256)<br/>- name<br/>- scopes (permissions)<br/>- rate_limit<br/>- expires_at

    KeyService-->>API: API Key: fineract_live_abc123...

    API-->>Admin: API Key Created<br/>Key: fineract_live_abc123...

    Admin->>ThirdParty: Provide API Key<br/>(Secure channel)

    Note over ThirdParty: Third-party integrates<br/>Fineract into their app

    ThirdParty->>API: GET /loans<br/>Headers:<br/>- X-API-Key: fineract_live_abc123...<br/>- Fineract-Platform-TenantId: acme

    API->>KeyService: Validate API Key

    KeyService->>KeyService: Hash Provided Key<br/>SHA-256

    KeyService->>DB: SELECT * FROM m_api_keys<br/>WHERE key_hash = ?<br/>AND expires_at > NOW()

    alt Key Not Found or Expired
        DB-->>KeyService: No Results
        KeyService-->>API: Invalid Key
        API-->>ThirdParty: HTTP 401 Unauthorized
    else Key Valid
        DB-->>KeyService: API Key Record:<br/>- scopes<br/>- rate_limit

        KeyService->>KeyService: Check Rate Limit<br/>Redis: GET api_key_rate:{key}

        alt Rate Limit Exceeded
            KeyService-->>API: Rate Limit Exceeded
            API-->>ThirdParty: HTTP 429 Too Many Requests
        else Within Limit
            KeyService->>KeyService: Increment Rate Counter<br/>Redis: INCR api_key_rate:{key}

            KeyService-->>API: Key Valid<br/>Scopes: [READ_LOAN, READ_CLIENT]

            API->>API: Check Permission for Endpoint<br/>Required: READ_LOAN<br/>Available: [READ_LOAN, READ_CLIENT]

            alt Permission Granted
                API->>DB: Execute Query

                DB-->>API: Loan Records

                API-->>ThirdParty: HTTP 200 OK<br/>Loan Data
            else Permission Denied
                API-->>ThirdParty: HTTP 403 Forbidden
            end
        end
    end
```

## 8. Two-Factor Authentication (2FA) Flow

```mermaid
flowchart TB
    Start([User Login<br/>username + password]) --> ValidateCreds{Valid<br/>Credentials?}

    ValidateCreds -->|No| Failed[HTTP 401<br/>Unauthorized]
    ValidateCreds -->|Yes| Check2FA{2FA<br/>Enabled?}

    Check2FA -->|No| GenerateToken[Generate JWT Token]
    Check2FA -->|Yes| CheckMethod{2FA Method?}

    CheckMethod -->|TOTP| GenerateTOTP
    CheckMethod -->|SMS| GenerateSMSOTP
    CheckMethod -->|Email| GenerateEmailOTP

    GenerateTOTP[User Opens<br/>Authenticator App] --> EnterTOTP[User Enters<br/>6-digit TOTP]
    GenerateSMSOTP[Send OTP via SMS] --> EnterSMS[User Enters<br/>SMS OTP]
    GenerateEmailOTP[Send OTP via Email] --> EnterEmail[User Enters<br/>Email OTP]

    EnterTOTP --> ValidateTOTP{TOTP<br/>Valid?}
    EnterSMS --> ValidateSMS{SMS OTP<br/>Valid?}
    EnterEmail --> ValidateEmail{Email OTP<br/>Valid?}

    ValidateTOTP -->|No| Failed2FA[HTTP 401<br/>Invalid 2FA Code]
    ValidateTOTP -->|Yes| GenerateToken

    ValidateSMS -->|No| Failed2FA
    ValidateSMS -->|Yes| GenerateToken

    ValidateEmail -->|No| Failed2FA
    ValidateEmail -->|Yes| GenerateToken

    GenerateToken --> SetSession[Set Security Context]
    SetSession --> Success[HTTP 200 OK<br/>Login Success]

    Failed --> End([End])
    Failed2FA --> End
    Success --> End

    style Start fill:#e1f5ff
    style Success fill:#c8e6c9
    style Failed fill:#ffcdd2
    style Failed2FA fill:#ffcdd2
```

## 9. Security Configuration & Hardening

```mermaid
graph TB
    subgraph "Password Security"
        PasswordPolicy[Password Policy<br/>- Min length: 12<br/>- Complexity: Upper+Lower+Number+Special<br/>- History: Last 5 passwords<br/>- Expiry: 90 days]

        PasswordHashing[Password Hashing<br/>- Algorithm: BCrypt<br/>- Work factor: 12<br/>- Salt: Per-password]

        AccountLockout[Account Lockout<br/>- Failed attempts: 5<br/>- Lockout duration: 30 min<br/>- Admin unlock required]
    end

    subgraph "Session Security"
        JWTConfig[JWT Configuration<br/>- Algorithm: RS256<br/>- Expiry: 24 hours<br/>- Refresh: 7 days<br/>- Revocation: Blacklist]

        SessionTimeout[Session Timeout<br/>- Idle timeout: 30 min<br/>- Absolute timeout: 8 hours<br/>- Extend on activity]

        ConcurrentSession[Concurrent Sessions<br/>- Max sessions: 1<br/>- Kill oldest on new login]
    end

    subgraph "Network Security"
        HTTPS[HTTPS Enforcement<br/>- TLS 1.2+<br/>- Strong cipher suites<br/>- HSTS enabled]

        RateLimit[Rate Limiting<br/>- Login: 5 attempts/min<br/>- API: 100 requests/min<br/>- By IP + User]

        CORS[CORS Policy<br/>- Allowed origins<br/>- Allowed methods<br/>- Credentials: true]

        IPWhitelist[IP Whitelist<br/>- Admin IPs<br/>- API Key IPs<br/>- Office network ranges]
    end

    subgraph "Data Security"
        Encryption[Encryption at Rest<br/>- Database: AES-256<br/>- Backups: Encrypted<br/>- Logs: Sensitive data masked]

        DataMasking[Data Masking<br/>- API responses<br/>- Logs<br/>- Audit trails]

        PII[PII Protection<br/>- GDPR compliance<br/>- Data retention policies<br/>- Right to deletion]
    end

    subgraph "Audit & Monitoring"
        AuditLog[Audit Logging<br/>- All API calls<br/>- Authentication events<br/>- Data changes<br/>- Permission changes]

        AlertSystem[Alert System<br/>- Failed logins<br/>- Permission violations<br/>- Suspicious activity<br/>- Data exports]

        IntrusionDetect[Intrusion Detection<br/>- SQL injection<br/>- XSS attempts<br/>- CSRF tokens<br/>- Anomaly detection]
    end

    PasswordPolicy --> PasswordHashing
    PasswordHashing --> AccountLockout

    JWTConfig --> SessionTimeout
    SessionTimeout --> ConcurrentSession

    HTTPS --> RateLimit
    RateLimit --> CORS
    CORS --> IPWhitelist

    Encryption --> DataMasking
    DataMasking --> PII

    AuditLog --> AlertSystem
    AlertSystem --> IntrusionDetect

    style HTTPS fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style Encryption fill:#c8e6c9,stroke:#2e7d32,stroke-width:2px
    style AuditLog fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

## Key Security Features

### Multi-Tenant Security Isolation

- **Database-per-tenant**: Each tenant has separate database schema
- **Tenant resolution**: Extracted from HTTP header `Fineract-Platform-TenantId`
- **Connection pooling**: Separate connection pools per tenant
- **Data isolation**: No cross-tenant data access possible
- **Schema validation**: Tenant must exist in master DB before access

### Authentication Methods

1. **Basic Authentication**: Username + Password (primary)
2. **OAuth2**: Support for external identity providers
3. **API Keys**: For third-party integrations
4. **Self-Service**: Customer portal with OTP verification
5. **Two-Factor Authentication**: TOTP, SMS, Email OTP

### Authorization Model

- **RBAC**: Role-based access control with granular permissions
- **Office hierarchy**: Users can only access data within their office tree
- **Permission groups**: Organized by domain (portfolio, organisation, accounting, system)
- **Maker-Checker**: Dual control for sensitive operations
- **Self-service scoping**: Customers can only see their own data

### Password Security

- **Hashing**: BCrypt with work factor 12
- **Salting**: Unique salt per password
- **Complexity**: Configurable password policy
- **History**: Prevents password reuse
- **Expiry**: Force periodic password changes
- **Account lockout**: After failed login attempts

### Session Management

- **JWT tokens**: Stateless authentication
- **Token expiry**: 24 hours (configurable)
- **Refresh tokens**: 7 days (configurable)
- **Token revocation**: Blacklist for immediate logout
- **Concurrent sessions**: Limit active sessions per user

### API Security

- **HTTPS only**: All API traffic encrypted
- **CORS**: Configurable cross-origin policy
- **Rate limiting**: Prevent abuse and DoS
- **Input validation**: Prevent injection attacks
- **Output encoding**: Prevent XSS
- **CSRF protection**: Token-based for state-changing operations

### Audit & Compliance

- **Audit trail**: All API calls logged with user, timestamp, changes
- **Change tracking**: Before/after values for data modifications
- **Login tracking**: Successful and failed login attempts
- **Permission tracking**: Changes to roles and permissions
- **Data export tracking**: Log all data exports for compliance
- **Retention**: Configurable audit log retention period

### Data Protection

- **Encryption at rest**: Database encryption (TDE)
- **Encryption in transit**: TLS 1.2+
- **PII masking**: Sensitive data masked in logs
- **GDPR compliance**: Support for data subject rights
- **Data retention**: Configurable retention policies
- **Backup security**: Encrypted backups with access controls
