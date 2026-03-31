# Technical Architecture

<cite>
**Referenced Files in This Document**
- [__manifest__.py](file://__manifest__.py)
- [abstract_report.py](file://report/abstract_report.py)
- [abstract_report_xlsx.py](file://report/abstract_report_xlsx.py)
- [abstract_wizard.py](file://wizard/abstract_wizard.py)
- [general_ledger.py](file://report/general_ledger.py)
- [general_ledger_wizard.py](file://wizard/general_ledger_wizard.py)
- [trial_balance.py](file://report/trial_balance.py)
- [trial_balance_wizard.py](file://wizard/trial_balance_wizard.py)
- [general_ledger.xml](file://report/templates/general_ledger.xml)
- [security.xml](file://security/security.xml)
- [ir.model.access.csv](file://security/ir.model.access.csv)
- [account.py](file://models/account.py)
- [account_move_line.py](file://models/account_move_line.py)
- [account_age_report_configuration.py](file://models/account_age_report_configuration.py)
- [res_config_settings.py](file://models/res_config_settings.py)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)

## Introduction
This document describes the technical architecture of the Account Financial Reports module. It explains the abstract report framework, the data processing pipeline, and the template rendering system. It documents the interactions among models, wizards, reports, and templates, details the inheritance hierarchy starting from abstract base classes, and traces the data flow from wizard configuration to final output generation. It also covers multi-currency support, date range handling, report caching mechanisms, and the security model with access controls and permissions.

## Project Structure
The module follows a layered structure:
- Wizard layer: user-configurable report parameters and actions
- Report layer: computation and aggregation logic, inheriting from abstract base classes
- Template layer: QWeb HTML templates and XLSX generation helpers
- Model layer: extensions to Odoo core accounting models and auxiliary configuration models
- Security layer: access rights and record rules

```mermaid
graph TB
subgraph "Wizard Layer"
GWZ["GeneralLedgerReportWizard"]
TWZ["TrialBalanceReportWizard"]
end
subgraph "Report Layer"
AR["AbstractReport (abstract_report.py)"]
GLR["GeneralLedgerReport (general_ledger.py)"]
TBR["TrialBalanceReport (trial_balance.py)"]
ARX["AbstractReportXslx (abstract_report_xlsx.py)"]
end
subgraph "Template Layer"
GLT["QWeb Template: general_ledger.xml"]
end
subgraph "Model Layer"
AA["AccountAccount (account.py)"]
AML["AccountMoveLine (account_move_line.py)"]
AARC["AccountAgeReportConfiguration* (account_age_report_configuration.py)"]
end
subgraph "Security Layer"
SECXML["security.xml"]
ACCESSCSV["ir.model.access.csv"]
end
GWZ --> GLR
TWZ --> TBR
GLR --> AR
TBR --> AR
GLR --> GLT
ARX -. "XLSX generation" .- GLR
AA --> GLR
AML --> GLR
AARC --> GWZ
SECXML --> GWZ
ACCESSCSV --> GWZ
```

**Diagram sources**
- [__manifest__.py:19-46](file://__manifest__.py#L19-L46)
- [abstract_report.py:7-165](file://report/abstract_report.py#L7-L165)
- [abstract_report_xlsx.py:8-698](file://report/abstract_report_xlsx.py#L8-L698)
- [abstract_wizard.py:7-52](file://wizard/abstract_wizard.py#L7-L52)
- [general_ledger.py:14-931](file://report/general_ledger.py#L14-L931)
- [trial_balance.py:12-981](file://report/trial_balance.py#L12-L981)
- [general_ledger_wizard.py:18-322](file://wizard/general_ledger_wizard.py#L18-L322)
- [trial_balance_wizard.py:12-285](file://wizard/trial_balance_wizard.py#L12-L285)
- [general_ledger.xml:1-789](file://report/templates/general_ledger.xml#L1-L789)
- [account.py:6-14](file://models/account.py#L6-L14)
- [account_move_line.py:9-71](file://models/account_move_line.py#L9-L71)
- [account_age_report_configuration.py:8-50](file://models/account_age_report_configuration.py#L8-L50)
- [security.xml:1-9](file://security/security.xml#L1-L9)
- [ir.model.access.csv:1-10](file://security/ir.model.access.csv#L1-L10)

**Section sources**
- [__manifest__.py:19-46](file://__manifest__.py#L19-L46)

## Core Components
- Abstract Report Base: Provides shared domain construction, move line recalculations, and common account/journal data retrieval.
- Abstract XLSX Report Base: Provides workbook creation, formatting, column definitions, and standardized XLSX writing helpers.
- Abstract Wizard Base: Provides shared wizard fields and export actions (HTML, PDF, XLSX).
- Concrete Reports: Implement report-specific domains, aggregations, and data shaping (e.g., General Ledger, Trial Balance).
- Templates: QWeb templates render HTML output; XLSX generation is handled via the XLSX abstract base.
- Models: Extend core models and introduce auxiliary configuration models for aging report intervals.
- Security: Access rights and record rules govern visibility and permissions.

**Section sources**
- [abstract_report.py:7-165](file://report/abstract_report.py#L7-L165)
- [abstract_report_xlsx.py:8-698](file://report/abstract_report_xlsx.py#L8-L698)
- [abstract_wizard.py:7-52](file://wizard/abstract_wizard.py#L7-L52)
- [general_ledger.py:14-931](file://report/general_ledger.py#L14-L931)
- [trial_balance.py:12-981](file://report/trial_balance.py#L12-L981)
- [general_ledger.xml:1-789](file://report/templates/general_ledger.xml#L1-L789)
- [account.py:6-14](file://models/account.py#L6-L14)
- [account_move_line.py:9-71](file://models/account_move_line.py#L9-L71)
- [account_age_report_configuration.py:8-50](file://models/account_age_report_configuration.py#L8-L50)
- [security.xml:1-9](file://security/security.xml#L1-L9)
- [ir.model.access.csv:1-10](file://security/ir.model.access.csv#L1-L10)

## Architecture Overview
The system is event-driven and data-centric:
- Wizard collects parameters and prepares a data payload.
- Report computes initial balances, aggregates move lines, and builds structured data.
- Templates render the final output (HTML or XLSX via report_xlsx).
- Security enforces access and visibility constraints.

```mermaid
sequenceDiagram
participant U as "User"
participant W as "Wizard (TransientModel)"
participant R as "Report (AbstractModel)"
participant DB as "Odoo ORM"
participant T as "QWeb/XLSX Template"
U->>W : Open wizard and set filters
W->>W : Validate and prepare report data
W->>R : Call report action with prepared data
R->>DB : Build domains and read-group/search-read
DB-->>R : Aggregated and raw data
R->>R : Transform and shape data (balances, lines, grouping)
R-->>W : Return report values
W->>T : Render HTML or generate XLSX
T-->>U : Deliver output
```

**Diagram sources**
- [general_ledger_wizard.py:274-322](file://wizard/general_ledger_wizard.py#L274-L322)
- [general_ledger.py:763-931](file://report/general_ledger.py#L763-L931)
- [abstract_report.py:21-165](file://report/abstract_report.py#L21-L165)
- [abstract_report_xlsx.py:18-698](file://report/abstract_report_xlsx.py#L18-L698)
- [general_ledger.xml:1-789](file://report/templates/general_ledger.xml#L1-L789)

## Detailed Component Analysis

### Abstract Report Framework
The abstract report base encapsulates common logic:
- Move line domain building for initial/fiscal/year domains
- Recalculation of residual amounts and currencies after reconciliations
- Retrieval of account and journal metadata
- Field sets for efficient read operations

```mermaid
classDiagram
class AbstractReport {
+COMMON_ML_FIELDS
+_get_move_lines_domain_not_reconciled(...)
+_get_new_move_lines_domain(...)
+_recalculate_move_lines(...)
+_get_accounts_data(...)
+_get_journals_data(...)
+_get_ml_fields()
}
class GeneralLedgerReport {
+_get_initial_balances_*_domain(...)
+_get_period_domain(...)
+_get_initial_balance_data(...)
+_get_period_ml_data(...)
+_create_general_ledger(...)
+_get_report_values(...)
}
class TrialBalanceReport {
+_get_initial_balances_*_domain(...)
+_get_period_ml_domain(...)
+_get_report_values(...)
}
GeneralLedgerReport --|> AbstractReport
TrialBalanceReport --|> AbstractReport
```

**Diagram sources**
- [abstract_report.py:7-165](file://report/abstract_report.py#L7-L165)
- [general_ledger.py:14-931](file://report/general_ledger.py#L14-L931)
- [trial_balance.py:12-981](file://report/trial_balance.py#L12-L981)

**Section sources**
- [abstract_report.py:7-165](file://report/abstract_report.py#L7-L165)

### Abstract XLSX Report Framework
The XLSX base provides:
- Workbook initialization and constant memory mode
- Standardized report scaffolding: title, filters, columns, content generation
- Formatting helpers for amounts, headers, and currencies
- Writing helpers for lines, initial/ending balances, and arrays

```mermaid
flowchart TD
Start(["XLSX Generation"]) --> Init["Init workbook and formats"]
Init --> Fetch["Fetch report name, filters, columns"]
Fetch --> Sheet["Create worksheet and set widths"]
Sheet --> Title["Write title and filters"]
Title --> Content["Generate report content"]
Content --> Footer["Write footer"]
Footer --> End(["Done"])
```

**Diagram sources**
- [abstract_report_xlsx.py:18-130](file://report/abstract_report_xlsx.py#L18-L130)
- [abstract_report_xlsx.py:605-698](file://report/abstract_report_xlsx.py#L605-L698)

**Section sources**
- [abstract_report_xlsx.py:8-698](file://report/abstract_report_xlsx.py#L8-L698)

### Wizard Layer and Data Preparation
Wizards collect parameters and prepare the data payload passed to reports:
- Date range handling and fiscal year start computation
- Domain composition and filtering by accounts, journals, partners, analytic accounts
- Export routing to HTML/PDF/XLSX via report actions

```mermaid
sequenceDiagram
participant W as "Wizard"
participant R as "Report"
participant A as "Action Report"
participant T as "Template"
W->>W : Compute fy_start_date, defaults
W->>W : Compose domain and filters
W->>R : Prepare data dict
R-->>W : Report values
W->>A : Search report by report_name and report_type
A->>T : Render template or XLSX
T-->>W : Output
```

**Diagram sources**
- [general_ledger_wizard.py:274-322](file://wizard/general_ledger_wizard.py#L274-L322)
- [trial_balance_wizard.py:12-285](file://wizard/trial_balance_wizard.py#L12-L285)

**Section sources**
- [general_ledger_wizard.py:18-322](file://wizard/general_ledger_wizard.py#L18-L322)
- [trial_balance_wizard.py:12-285](file://wizard/trial_balance_wizard.py#L12-L285)
- [abstract_wizard.py:7-52](file://wizard/abstract_wizard.py#L7-L52)

### Report Data Processing Pipeline (General Ledger)
The General Ledger report orchestrates:
- Initial balances computation across BS and PL accounts
- Period move lines aggregation with optional grouping by partners/taxes
- Cumulative balance calculation and reconciliation adjustments
- Centralized entries generation and final account assembly

```mermaid
flowchart TD
DFrom["date_from"] --> IB["Compute initial balances (BS/PL)"]
DTo["date_to"] --> PL["Compute PL initial balances (fy_start_date)"]
IB --> AccData["Load accounts/journals/taxes/analytic"]
PL --> AccData
AccData --> Period["Period domain and search-read"]
Period --> Group["Group by account/partner/tax as configured"]
Group --> Cumul["Recalculate cumulative balances"]
Cumul --> Centralize{"Centralize?"}
Centralize --> |Yes| Centralized["Build centralized lines"]
Centralize --> |No| Accounts["Assemble final accounts"]
Centralized --> Accounts
Accounts --> Out["Return report values"]
```

**Diagram sources**
- [general_ledger.py:258-800](file://report/general_ledger.py#L258-L800)
- [abstract_report.py:21-165](file://report/abstract_report.py#L21-L165)

**Section sources**
- [general_ledger.py:14-931](file://report/general_ledger.py#L14-L931)

### Template Rendering System (QWeb)
Templates define the HTML output structure:
- Filters display, account headers, move lines table, and ending balances
- Conditional rendering for foreign currency and analytic distributions
- Clickable links to underlying records via res-model attributes

```mermaid
graph TB
GLT["general_ledger.xml"]
Base["report_general_ledger_base"]
Filters["report_general_ledger_filters"]
Lines["report_general_ledger_lines"]
Ending["report_general_ledger_ending_cumul"]
GLT --> Base
Base --> Filters
Base --> Lines
Base --> Ending
```

**Diagram sources**
- [general_ledger.xml:1-789](file://report/templates/general_ledger.xml#L1-L789)

**Section sources**
- [general_ledger.xml:1-789](file://report/templates/general_ledger.xml#L1-L789)

### Multi-Currency Support
- Currency-aware fields are included in move line reads and rendered conditionally in templates.
- Amounts and cumulative balances can be shown in account currency when applicable.
- XLSX formatting supports per-currency number formats derived from currency decimals.

Key behaviors:
- Fields retrieved include residual and currency amounts.
- Templates check currency presence and adjust display accordingly.
- XLSX formats adapt to currency decimal places.

**Section sources**
- [abstract_report.py:154-165](file://report/abstract_report.py#L154-L165)
- [general_ledger.py:318-361](file://report/general_ledger.py#L318-L361)
- [general_ledger.xml:179-189](file://report/templates/general_ledger.xml#L179-L189)
- [general_ledger.xml:601-641](file://report/templates/general_ledger.xml#L601-L641)
- [abstract_report_xlsx.py:57-92](file://report/abstract_report_xlsx.py#L57-L92)
- [abstract_report_xlsx.py:526-568](file://report/abstract_report_xlsx.py#L526-L568)

### Date Range Handling
- Wizards compute fiscal year start date based on company settings.
- Domains restrict move lines to the selected date range and target moves (posted/all).
- Special handling for reclassified periods and reconciliations after the reporting period.

**Section sources**
- [general_ledger_wizard.py:131-142](file://wizard/general_ledger_wizard.py#L131-L142)
- [general_ledger.py:362-391](file://report/general_ledger.py#L362-L391)
- [general_ledger.py:403-417](file://report/general_ledger.py#L403-L417)

### Report Caching Mechanisms
- No explicit caching is implemented in the reviewed files. The system relies on efficient read-group and search-read operations with precomputed domains and indices.
- Index creation for move lines improves performance for large datasets.

**Section sources**
- [account_move_line.py:39-62](file://models/account_move_line.py#L39-L62)

### Security Model, Access Controls, and Permissions
- Access rights: CSV grants read/write/create/unlink permissions for wizard models to base.group_user.
- Record rules: Domain forces company scoping for configuration records.
- Module dependencies: Includes security-related modules via manifest.

```mermaid
graph TB
Access["ir.model.access.csv"]
Rule["security.xml (record rule)"]
WizardGL["GeneralLedgerReportWizard"]
WizardTB["TrialBalanceReportWizard"]
Config["AccountAgeReportConfiguration"]
Access --> WizardGL
Access --> WizardTB
Rule --> Config
```

**Diagram sources**
- [ir.model.access.csv:1-10](file://security/ir.model.access.csv#L1-L10)
- [security.xml:1-9](file://security/security.xml#L1-L9)
- [general_ledger_wizard.py:18-322](file://wizard/general_ledger_wizard.py#L18-L322)
- [trial_balance_wizard.py:12-285](file://wizard/trial_balance_wizard.py#L12-L285)
- [account_age_report_configuration.py:8-50](file://models/account_age_report_configuration.py#L8-L50)

**Section sources**
- [ir.model.access.csv:1-10](file://security/ir.model.access.csv#L1-L10)
- [security.xml:1-9](file://security/security.xml#L1-L9)
- [__manifest__.py:18-21](file://__manifest__.py#L18-L21)

### Auxiliary Models and Configuration
- Account extension adds centralized flag for General Ledger grouping.
- Aging report configuration models manage interval definitions and defaults via settings.

**Section sources**
- [account.py:6-14](file://models/account.py#L6-L14)
- [account_age_report_configuration.py:8-50](file://models/account_age_report_configuration.py#L8-L50)
- [res_config_settings.py:15-37](file://models/res_config_settings.py#L15-L37)

## Dependency Analysis
The module depends on core accounting and reporting modules, and integrates with XLSX generation and date ranges.

```mermaid
graph TB
Manifest["__manifest__.py"]
Account["account"]
DateRange["date_range"]
ReportXlsx["report_xlsx"]
Security["security/*.xml"]
Wizards["wizard/*_wizard.py"]
Reports["report/*_report.py"]
Templates["report/templates/*.xml"]
Manifest --> Account
Manifest --> DateRange
Manifest --> ReportXlsx
Manifest --> Security
Manifest --> Wizards
Manifest --> Reports
Manifest --> Templates
```

**Diagram sources**
- [__manifest__.py:18-46](file://__manifest__.py#L18-L46)

**Section sources**
- [__manifest__.py:18-46](file://__manifest__.py#L18-L46)

## Performance Considerations
- Efficient domain building reduces database load.
- Indices on move lines improve join performance for large datasets.
- Constant memory mode in XLSX workbook reduces memory footprint during generation.
- read_group and targeted field selection minimize data transfer.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and checks:
- Date range mismatch: Ensure wizard date range matches company fiscal year settings.
- Currency display: Verify multi-currency group membership and account currency setup.
- Reconciliation after reporting period: Adjustments mark reconciliations occurring after the reporting period.
- Analytic distribution: Confirm analytic account ids are populated and filtered correctly.

**Section sources**
- [general_ledger_wizard.py:218-232](file://wizard/general_ledger_wizard.py#L218-L232)
- [general_ledger.py:561-569](file://report/general_ledger.py#L561-L569)
- [account_move_line.py:16-38](file://models/account_move_line.py#L16-L38)

## Conclusion
The Account Financial Reports module implements a robust, extensible architecture centered on abstract report and wizard bases. It cleanly separates concerns across layers, leverages Odoo ORM efficiently, and provides flexible output formats (HTML and XLSX). Security is enforced through access rights and record rules. The design supports multi-currency, precise date range handling, and performance-conscious operations, enabling reliable financial reporting at scale.