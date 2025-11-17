# Apache Fineract ERD Diagrams - Usage Guide

This directory contains PlantUML Entity Relationship Diagrams (ERD) for the Apache Fineract codebase.

## Available Diagrams

### 1. Complete ERD (`fineract-erd-complete.puml`)
- **Entities:** All 234 entities across all domains
- **Coverage:** Organizational, Client/Group, Loan, Savings, Accounting, Infrastructure
- **Use Case:** Get a complete overview of the entire database schema

### 2. Loan Domain ERD (`fineract-erd-loan-only.puml`)
- **Entities:** Loan, LoanProduct, LoanTransaction, LoanRepaymentSchedule, LoanCharge, Collateral, Guarantor, and more
- **Features:** 43 transaction types, re-aging, re-amortization, reschedule workflows
- **Use Case:** Deep dive into loan management implementation

### 3. Savings Domain ERD (`fineract-erd-savings-only.puml`)
- **Entities:** SavingsAccount, SavingsProduct, FixedDeposit, RecurringDeposit, Interest Rate Charts
- **Features:** Interest calculation, dormancy tracking, account types
- **Use Case:** Understand savings and deposit account implementation

### 4. Accounting Domain ERD (`fineract-erd-accounting-only.puml`)
- **Entities:** GLAccount, JournalEntry, ProductToGLAccountMapping, AccountingRule, Provisioning
- **Features:** Chart of accounts, automatic journal posting, trial balance
- **Use Case:** Understand accounting integration and GL structure

## How to View the Diagrams

### Option 1: Online PlantUML Editor (Easiest)
1. Go to [PlantUML Online Editor](http://www.plantuml.com/plantuml/uml/)
2. Copy the content of any `.puml` file
3. Paste it into the editor
4. The diagram will render automatically
5. You can download as PNG, SVG, or other formats

### Option 2: VS Code Extension
1. Install the **PlantUML** extension in VS Code
   - Extension ID: `jebbs.plantuml`
2. Open any `.puml` file in VS Code
3. Press `Alt+D` to preview the diagram
4. Right-click and select "Export Current Diagram" to save as image

### Option 3: IntelliJ IDEA Plugin
1. Install the **PlantUML integration** plugin
   - Go to Settings → Plugins → Search "PlantUML"
2. Open any `.puml` file
3. The diagram will render in the preview pane
4. Right-click to export

### Option 4: Command Line (Local Installation)

**Install PlantUML:**
```bash
# On macOS with Homebrew
brew install plantuml

# On Ubuntu/Debian
sudo apt-get install plantuml

# Or download JAR from http://plantuml.com/download
```

**Generate PNG:**
```bash
plantuml fineract-erd-complete.puml
# Output: fineract-erd-complete.png

# Generate all diagrams
plantuml fineract-erd-*.puml
```

**Generate SVG (scalable):**
```bash
plantuml -tsvg fineract-erd-complete.puml
```

### Option 5: Docker
```bash
# Generate PNG
docker run --rm -v $(pwd):/data plantuml/plantuml:latest fineract-erd-complete.puml

# Generate SVG
docker run --rm -v $(pwd):/data plantuml/plantuml:latest -tsvg fineract-erd-complete.puml
```

## Diagram Features

### Entity Structure
Each entity box shows:
- **Entity Name** (header)
- **Primary Key** (marked with `* <<PK>>`)
- **Foreign Keys** (marked with `<<FK>>`)
- **Field Types** (BIGINT, VARCHAR, DECIMAL, etc.)
- **Constraints** (unique, not null)
- **Enums** (with possible values documented)
- **Derived Fields** (calculated/cached values)

### Relationships
- `||--||` : One-to-One
- `||--o{` : One-to-Many
- `}o--||` : Many-to-One
- `}o--o{` : Many-to-Many

### Reading the Diagrams

**Example Entity:**
```
entity "Loan" as loan {
  * id : BIGINT <<PK>>               ← Primary Key
  --
  * account_no : VARCHAR(20) <<unique>>  ← Required, Unique
  external_id : VARCHAR(100)          ← Optional
  * client_id : BIGINT <<FK>>         ← Foreign Key (Required)
  group_id : BIGINT <<FK>>            ← Foreign Key (Optional)
  --
  * loan_status_id : SMALLINT         ← Enum values documented
    (100=SUBMITTED, 200=APPROVED,
     300=ACTIVE, 600=CLOSED, ...)
  --
  principal_outstanding_derived : DECIMAL(19,6)  ← Calculated field
}
```

**Example Relationship:**
```
Client ||--o{ Loan : "borrows"
```
Reads as: "One Client can have Many Loans" (One-to-Many)

## Tips for Understanding

### Start Small
1. Begin with domain-specific diagrams (Loan, Savings, or Accounting)
2. Focus on core entities first (Loan, Client, SavingsAccount)
3. Understand the relationships before diving into fields
4. Reference the complete diagram for cross-domain relationships

### Key Entity Groups

**Loan Domain:**
- Loan → LoanProduct, LoanTransaction, LoanRepaymentSchedule, LoanCharge

**Savings Domain:**
- SavingsAccount → SavingsProduct, SavingsTransaction, SavingsCharge

**Accounting:**
- GLAccount → JournalEntry → ProductToGLAccountMapping

### Derived Fields
Fields ending in `_derived` are calculated and cached for performance:
- `principal_outstanding_derived` = principal - principal_repaid - principal_written_off
- `account_balance_derived` = deposits - withdrawals + interest - fees
- `running_balance_derived` = cumulative balance after transaction

## Customizing the Diagrams

### Change Layout Direction
Add at the top:
```plantuml
left to right direction
```

### Focus on Specific Entities
Comment out sections you don't need:
```plantuml
' ============================================
' COMMENT OUT ENTIRE SECTION
' ============================================
' entity "UnneededEntity" as entity {
'   ...
' }
```

### Change Colors
Add custom styling:
```plantuml
skinparam entity {
  BackgroundColor LightBlue
  BorderColor DarkBlue
}
```

### Simplify Relationships
Hide relationship labels:
```plantuml
hide empty methods
hide empty attributes
```

## Troubleshooting

**Diagram too large to render:**
- Use domain-specific diagrams instead of complete ERD
- Export as SVG for better scalability
- Increase memory for PlantUML: `java -Xmx2048m -jar plantuml.jar file.puml`

**Missing relationships:**
- Some relationships are intentionally simplified
- Check source code for full relationship details
- Refer to comprehensive guide: `APACHE_FINERACT_COMPLETE_GUIDE.md`

**Syntax errors:**
- Ensure you copied the entire file content
- Check for special characters in entity names
- Validate at plantuml.com before local rendering

## Additional Resources

- **PlantUML Documentation:** https://plantuml.com/
- **ERD Syntax:** https://plantuml.com/ie-diagram
- **Fineract Guide:** See `APACHE_FINERACT_COMPLETE_GUIDE.md` in this repository
- **Database Schema:** Check `/fineract-provider/src/main/resources/db/changelog/` for Liquibase migrations

## Integration with Development

### Use in Documentation
Export diagrams as images and include in your docs:
```bash
plantuml -tpng fineract-erd-loan-only.puml
# Include in README or wiki
```

### Database Design Reference
Use these diagrams when:
- Writing database migrations
- Understanding entity relationships
- Designing new features
- Debugging data integrity issues
- Onboarding new developers

### Code Generation
PlantUML diagrams can be:
- Converted to database DDL
- Used as basis for ORM mappings
- Imported into database modeling tools (MySQL Workbench, DbSchema, etc.)

## Contributing

To update diagrams:
1. Modify the `.puml` file
2. Test rendering locally
3. Commit with descriptive message
4. Include screenshot in PR description

## License

These diagrams are part of the Apache Fineract project and follow the same Apache 2.0 license.

---

**Happy Diagramming! 📊**

For questions or improvements, refer to the main Apache Fineract documentation or community channels.
