# BizMaker Payroll System

A comprehensive payroll management system designed for Philippine businesses, supporting ZKTeco biometric device integration, automated holiday pay calculations, loan tracking, shift scheduling, anomaly detection, and secured REST APIs.

## Project Overview

BizMaker Payroll is a full-stack application that synchronizes attendance data from biometric hardware to a centralized database, processes payroll according to Philippine labor laws (DOLE/BIR), and provides a modern web interface for administrative management.

## System Lifecycle Flowchart

```mermaid
flowchart TD
    subgraph Init["<b>Phase 1: Environment & Authentication</b>"]
        direction TB
        s1[ ] --- Start((Start))
        Start --> Env{Env Check}
        Env -- "Fail" --> Err1[/Log Error & Halt/]
        Env -- "Success" --> Auth{Device Set?}
        Auth -- No --> Setup[Setup Hardware]
        Auth -- Yes --> Login[Login/Auth]
        Setup --> Login
        Login --> Dashboard[Premium Dashboard]
    end

    subgraph Ops["<b>Phase 2: Smart Attendance Sync</b>"]
        direction TB
        s2[ ] --- Dashboard
        Trigger[Sync Trigger] --> Sync[Fetch Logs]
        Dashboard --> Trigger
        
        Sync --> Busy{Busy?}
        Busy -- No --> Hol{Holiday?}
        Hol -- Yes --> Resting[Mark Resting]
        Hol -- No --> Proc[Process Logs]
        
        Proc --> SmartSync{Smart Sync}
        SmartSync -->|Update Placeholder| Emp[Employee DB]
        Proc --> BreakTime[Break Subtraction]
        BreakTime --> Rec[Daily Record]
        Resting --> Rec
    end

    subgraph Final["<b>Phase 3: BIR/DOLE Compliance</b>"]
        direction TB
        s3[ ] --- Rec
        Rec --> Cycle{Period End}
        Cycle -- Yes --> Engine[PH Payroll Engine]
        Engine --> Tables{Deductions}
        Tables --> Calc[Net Pay]
        Calc --> Gen[Payslip]
        Gen --> Export([Export Reports])
        Export --> PDF[BIR 1601-C PDF]
        Export --> DAT[Alpha-List .DAT]
        Export --> CSV[Bank Transmittal]
        Export --> REM[SSS/PhilHealth/PagIBIG Remittance]
    end

    %% Styling
    classDef bizGold fill:#D4AF37,stroke:#333,stroke-width:2px,color:#fff
    classDef bizNavy stroke:#002060,stroke-width:2px,fill:none
    classDef process fill:#f8f9fa,stroke:#002060,stroke-width:1px
    classDef spacer fill:none,stroke:none,color:none
    
    class Start,Export,PDF,DAT,CSV bizGold
    class Init,Ops,Final bizNavy
    class Env,Auth,Busy,SmartSync,Cycle,Tables process
    class s1,s2,s3 spacer
```

## System Processes

### Attendance & Payroll Flow
```mermaid
flowchart LR
    HW[ZK Device] -->|Fetch Logs| LOG[Raw Attendance Log]
    LOG -->|Mapping| MAP{Machine ID Mapping}
    MAP -->|Process| DAY[Daily Attendance Record]
    
    FW[Fieldwork Requests] -->|Hybrid Merge| DAY
    HOL[Holiday Pay Config] -->|Multiplier Appl| DAY
    
    DAY -->|Input| PAY[Payroll Engine]
    PRO[Configuration Store] -->|Contribution Tables| PAY
    PAY -->|BIR Compliance| SLIP[Individual Payslips]
    PAY -->|Accounting| REP[Excel/PDF Reports]
    PAY -->|Loans| LOAN[Auto-Deduct Active Loans]
    PAY -->|Gov Files| REMIT[SSS R3 / PhilHealth RF-1 / PagIBIG MCRF]

    style HW fill:#002060,color:#fff
    style SLIP fill:#D4AF37,color:#fff
    style REP fill:#D4AF37,color:#fff
    style REMIT fill:#10b981,color:#fff
```

### Fieldwork & Physical Attendance Overlap
```mermaid
flowchart TD
    START([Employee Day]) --> FW{Approved Fieldwork?}
    FW -- No --> PHYS[Physical Logs Only]
    FW -- Yes --> LOGS{Physical Logs Exist?}
    
    LOGS -- No --> BASE[8.0 Standard Hours]
    LOGS -- Yes --> MERGE[Merge Effective Boundaries]
    
    MERGE --> EFF["effective_in = max(check_in, shift_start)<br/>effective_out = max(check_out, fw_end)"]
    EFF --> BREAKS[Subtract Unpaid Breaks ≥ 1h]
    BREAKS --> CALC{Net Hours > 8.0?}
    
    CALC -- Yes --> OT[Base 8h + Overtime]
    CALC -- No --> FLOOR[Guaranteed 8.0h Floor]
    
    OT --> ND{Check-out ≥ 10 PM?}
    FLOOR --> SAVE[Save DailyAttendance]
    ND -- Yes --> NIGHT[+ Night Differential]
    ND -- No --> SAVE
    NIGHT --> SAVE

    style START fill:#002060,color:#fff
    style BASE fill:#10b981,color:#fff
    style OT fill:#D4AF37,color:#fff
    style NIGHT fill:#6366f1,color:#fff
    style FLOOR fill:#10b981,color:#fff
```

### Attendance Analytics Pipeline
```mermaid
flowchart LR
    UI[Period Selector] -->|Day/Week/Month/Year/Custom| DATES[start_date & end_date]
    EMP[Employee Filter] -->|Multi-select + Position Search| IDS[employee_ids]
    BR[Branch Filter] -->|Company Selection| BID[branch_id]
    
    DATES --> API{API Endpoints}
    IDS --> API
    BID --> API
    
    API --> BIO[Biometric Logs]
    API --> ATT[Attendance Summary]
    API --> PER[Period Summary]
    
    PER --> METRICS["total_work_hours<br/>attendance_rate<br/>punctuality_rate<br/>overtime_hours<br/>late_count"]

    style UI fill:#002060,color:#fff
    style METRICS fill:#D4AF37,color:#fff
    style PER fill:#10b981,color:#fff
```

### Anomaly Detection Pipeline
```mermaid
flowchart LR
    PERIOD[Date Range] -->|start_date & end_date| RULES{Rule Engine}
    EMP[Employee Filter] -->|Optional employee_ids| RULES
    
    RULES --> R1["Ghost OT\n>3h OT, no fieldwork"]
    RULES --> R2["Missing Checkout\nClock-in, no clock-out"]
    RULES --> R3["Excessive Late\n5+ lates in period"]
    RULES --> R4["Rest Day Work\nNo approval"]
    
    R1 --> SEV{Severity}
    R2 --> SEV
    R3 --> SEV
    R4 --> SEV
    
    SEV --> HIGH[High 🔴]
    SEV --> MED[Medium 🟡]
    SEV --> LOW[Low 🔵]

    style RULES fill:#002060,color:#fff
    style HIGH fill:#ef4444,color:#fff
    style MED fill:#f59e0b,color:#fff
    style LOW fill:#6366f1,color:#fff
```

### Labor Cost Aggregation (Stacked Analysis)
```mermaid
flowchart TD
    DB[(Payslip DB)] --> ATTR[Branch Attribution]
    ATTR --> AAA[AAA and Co., CPAs]
    ATTR --> BIZ[Bizmaker Consultancy]
    ATTR --> GAM[Gamma Oracle Dimensions]
    
    AAA --> SUM[Annotate: Sum gross/net]
    BIZ --> SUM
    GAM --> SUM
    
    SUM --> TRUNC[TruncMonth: Group by Period]
    TRUNC --> CHART[Stacked Bar Chart]
    
    subgraph View["Dashboard Visualization"]
        CHART --> SEG1[AAA Segment]
        CHART --> SEG2[Bizmaker Segment]
        CHART --> SEG3[Gamma Segment]
    end

    style DB fill:#002060,color:#fff
    style SEG1 fill:#002060,color:#fff
    style SEG2 fill:#D4AF37,color:#fff
    style SEG3 fill:#C0392B,color:#fff
```

### Loan Lifecycle
```mermaid
flowchart LR
    CREATE[Create Loan] --> ACTIVE[Active]
    ACTIVE --> PAYROLL{Payroll Run}
    PAYROLL --> DEDUCT[Auto-Deduct from Net Pay]
    DEDUCT --> BAL{Balance = 0?}
    BAL -- Yes --> PAID[Fully Paid ✅]
    BAL -- No --> ACTIVE
    ACTIVE --> MANUAL[Manual Payment]
    MANUAL --> BAL
    ACTIVE --> CANCEL[Admin Cancel]

    style CREATE fill:#002060,color:#fff
    style PAID fill:#10b981,color:#fff
    style CANCEL fill:#ef4444,color:#fff
    style DEDUCT fill:#D4AF37,color:#fff
```

### Shift Scheduling Flow
```mermaid
flowchart TD
    SHIFT[Shift Templates] --> ASSIGN{Assignment}
    ASSIGN --> SINGLE[Click Cell → Quick Assign]
    ASSIGN --> BULK[Bulk Assign → Date Range + Employees]
    
    SINGLE --> ROSTER[Weekly Calendar Roster]
    BULK --> ROSTER
    
    ROSTER --> ATT{Attendance Check}
    ATT --> HAS["Employee has shift → Use shift times"]
    ATT --> NO["No shift → Use global config defaults"]

    style SHIFT fill:#002060,color:#fff
    style ROSTER fill:#D4AF37,color:#fff
    style HAS fill:#10b981,color:#fff
    style NO fill:#6366f1,color:#fff
```

## Database Schema (ERD)

```mermaid
erDiagram
    DEPARTMENT ||--o{ EMPLOYEE : manages
    
    EMPLOYEE ||--o{ ATTENDANCE_LOG : "generates punches"
    BIOMETRIC_DEVICE ||--o{ ATTENDANCE_LOG : records
    
    EMPLOYEE ||--o{ DAILY_ATTENDANCE : summarizes
    EMPLOYEE ||--o{ FIELDWORK_REQUEST : files
    
    PAYROLL_PERIOD ||--o{ PAYSLIP : contains
    EMPLOYEE ||--o{ PAYSLIP : receives
    
    PAYROLL_CONFIGURATION ||--o{ PAYROLL_PERIOD : controls
    SSS_TABLE ||--o{ PAYROLL_CONFIGURATION : references
    PH_TABLE ||--o{ PAYROLL_CONFIGURATION : references
    BIR_TAX_TABLE ||--o{ PAYROLL_CONFIGURATION : references

    EMPLOYEE {
        string employee_id PK
        int device_user_id UK "Unified Machine ID"
        int branch_id FK "Company Assignment"
        string position "Free-text field"
        string tin_no "BIR ID"
        decimal basic_salary "Monthly Base"
        decimal hourly_rate "Auto-calculated"
    }
    
    DAILY_ATTENDANCE {
        date date PK
        datetime check_in
        datetime check_out
        float total_hours
        float overtime_hours
        float night_diff_hours
        boolean is_fieldwork "OB Override"
        boolean is_late
        int late_minutes
    }
    
    FIELDWORK_REQUEST {
        date start_date
        date end_date
        string reason
        string status "PENDING/APPROVED/REJECTED"
    }
    
    PAYSLIP {
        decimal gross_pay "Inclusive of Holiday/OT"
        decimal net_pay "Take Home"
        decimal sss_deduction
        decimal philhealth_deduction
        decimal withholding_tax "BIR Computed"
    }

    PAYROLL_CONFIGURATION {
        decimal sss_fixed_rate
        decimal philhealth_fixed_rate
        string current_tax_table
        boolean auto_calculate_13th_month "Toggle 13th month accrual"
        boolean bonuses_available "Toggle all bonus processing"
    }

    EMPLOYEE ||--o{ LOAN : borrows
    LOAN ||--o{ LOAN_PAYMENT : tracks
    PAYSLIP ||--o{ LOAN_PAYMENT : "auto-deducts"
    
    SHIFT ||--o{ EMPLOYEE_SHIFT : assigned
    EMPLOYEE ||--o{ EMPLOYEE_SHIFT : scheduled

    LOAN {
        string loan_type "CASH_BOND/SALARY/SSS/PAGIBIG"
        decimal principal_amount
        decimal monthly_deduction
        decimal remaining_balance
        string status "ACTIVE/PAID/CANCELLED"
    }

    SHIFT {
        string name "Morning/Mid/Night"
        time start_time
        time end_time
        time break_start
        time break_end
        boolean is_night_shift
        string color "Hex for calendar"
    }

    EMPLOYEE_SHIFT {
        date date "Specific day"
        int employee_id FK
        int shift_id FK
    }
```

## Technical Stack

### Backend
- Framework: Django 5.2 (Python 3.12+)
- API: Django REST Framework (DRF)
- Authentication: JWT with Secure Blob Storage
- Reports: fpdf2 (Official PDF Generation)
- Database: PostgreSQL / SQLite
- Integration: PyZK for Biometric Devices

### Frontend
- Framework: Vue 3 (Composition API)
- UI Library: Element Plus (Premium Glassmorphism)
- Charts: ApexCharts (Vibrant Gradients)
- State Management: Pinia
- Icons: Element Plus Icons, Dicebear (Avatars)
- Utilities: Axios, Dayjs

## Core Features

### 1. Philippine Payroll Compliance
- **Semi-Monthly Processing**: Native support for 15-day pay cycles with period-specific financial summaries.
- **Official BIR Reporting**: 
  - **BIR Form 1601-C (PDF)**: Professional PDF summaries for monthly remittance.
  - **Annual Alpha-List (.DAT)**: Mandatory BIR-compliant file format for validation modules.
- **Bank Transmittal (CSV)**: Grouped salary disbursement files with period identifiers.
- **Holiday Pay Matrix**: Automated Regular (200%), Special (130%), and Rest Day premiums.
- **Government Tables**: Automated SSS, PhilHealth, and Pag-IBIG deduction rules.
- **Admin Configuration Toggles**:
  - **Auto-accrue 14th–16th Month Pay**: 13th month is always computed (mandatory under PD 851). When toggled ON, all employees automatically receive 14th–16th month accrual and per-employee switches are locked. When OFF, admins can manually enable 14th–16th month per employee.
  - **Enable Bonus Management**: When enabled, Perfect Attendance, Christmas, and manual bonuses are included in payroll processing. Disable to zero out all bonus calculations.

#### Admin Config Toggle Flow
```mermaid
flowchart TD
    CFG[Admin Settings Panel] --> A13{Auto-accrue 14–16th?}
    
    A13 -- ON --> LOCK["All employees: 14th/15th/16th = ON<br/>Per-employee toggles LOCKED"]
    A13 -- OFF --> MANUAL["Per-employee toggles UNLOCKED<br/>All set to OFF by default"]
    MANUAL --> ADMIN["Admin can manually enable<br/>14th/15th/16th per employee"]
    
    LOCK --> ENGINE[Payroll Engine]
    ADMIN --> ENGINE
    
    CFG --> BONUS{Enable Bonus Mgmt?}
    BONUS -- ON --> BPROC["Perfect Attendance + Christmas<br/>+ Manual bonuses processed"]
    BONUS -- OFF --> BSKIP["All bonus calculations = ₱0"]
    
    BPROC --> ENGINE
    BSKIP --> ENGINE
    
    ENGINE --> MANDATORY["13th Month: ALWAYS computed<br/>(PD 851 - Mandatory)"]
    ENGINE --> SLIP[Payslip Generated]

    style CFG fill:#002060,color:#fff
    style MANDATORY fill:#10b981,color:#fff
    style LOCK fill:#D4AF37,color:#fff
    style SLIP fill:#D4AF37,color:#fff
    style BSKIP fill:#ef4444,color:#fff
```

### 2. Smart Biometric Integration
- **"Smart Sync" Profiling**: Automatically updates "Unknown" employee profiles with data from biometric logs, enriching the database on the fly.
- **Break Time Manager**: Global break intervals (e.g., 12:00-13:00) automatically subtracted from work hours if overlapped.
- **Real-time Monitoring**: Instant dashboard updates as employees punch in/out.
- **Hardware Protection**: Prevents "Device Busy" errors when external software (ZKAccess) is connected.

### 3. Advanced Attendance Analytics
- **Unified Period Picker**: Single selector with Day, Week, Month, Year, and Custom Range options. All selections are converted into `start_date` and `end_date` for unified backend filtering.
- **Branch-Level Organization**: A new dedicated Branch Filter allows admins to "sort" all attendance data by company (`AAA and Co., CPAs`, `Bizmaker Consultancy, Inc.`, or `Gamma Oracle Dimensions Inc.`), instantly updating logs and summaries.
- **Multi-Select Employee Filter**: Searchable dropdown supporting filtering by employee name or position. Select multiple employees to generate custom group analytics.
- **Period Summary Dashboard**: Aggregated metrics including total work hours, attendance rate, punctuality rate, overtime, late counts, and undertime — all dynamically scoped to the selected period.
- **Real-Time Period Capping**: If the selected month or year hasn't ended, the backend caps expected workdays to today's date, preventing inflated absence counts. A brief notification informs the admin that figures are still updating.
- **Dynamic Dashboard KPIs**: The "Logs" stat card on the main dashboard dynamically updates its label and count based on the selected trend period (Today/This Week/This Month/This Year).

### 4. Fieldwork & Hybrid Attendance
- **Admin-Controlled Approval**: All fieldwork requests default to `PENDING` status and require explicit admin approval, even if the admin initiated the request.
- **Guaranteed 8-Hour Baseline**: Approved fieldwork guarantees a minimum of 8.0 standard work hours for the day.
- **Custom Request Durations**: Admins can specify custom start/end times per fieldwork request, overriding standard shift windows for granular project-based tracking.
- **Hybrid Overlap Processing**: If an employee clocks into the biometric device on a fieldwork day, the engine merges the two timelines by unioning the shift boundaries (`max(check_in, shift_start)` to `max(check_out, fw_end)`), preventing double-counting while capturing all extended work.
- **Overtime on Extended Days**: Physical presence beyond the standard 8-hour threshold on a fieldwork day correctly generates overtime hours.
- **Night Differential Preservation**: Physical clock-out timestamps past 10 PM on fieldwork days still trigger night differential calculations.
- **Late Excusal**: Employees on approved fieldwork are automatically excused from late penalties.

### 5. Dynamic Attendance Logic
- **Early Punch Capping**: Biometric check-ins before the scheduled shift start (e.g., punching in at 7:00 AM for an 8:30 AM shift) are automatically capped to the shift start time. This prevents early arrivals from artificially inflating `total_hours` or triggering unintended overtime.
- **Dynamic Shift End Projection**: The system no longer assumes a fixed 9-hour gross shift. It dynamically calculates the shift end as `Shift Start + 8.0 (net hours) + Sum of all active unpaid breaks (≥ 1 hour)`.
- **Automated Break Subtraction**: Any active break defined in settings with a duration of 1 hour or more is automatically deducted from total hours if the employee's work interval overlaps with the break window.

### 6. Security & Reliability
- **Authenticated Exports**: All reports secured behind JWT, preventing unauthorized data access.
- **Encrypted Comm Keys**: AES-256 encryption for hardware communication keys stored in the database.
- **Defensive Downloads**: Blob-based download logic with race-condition protection.
- **Secure Password Change**: Verified identity check requiring current password for admin password resets.
- **Production Validation**: Enforced environment safety checks ensuring all encryption keys are present and valid before the system starts.
- **API Throttling**: Protection against brute-force and scraping.
- **Secure Headers**: HSTS, XSS Filter, and Content-Type Sniffing protection.

### 7. Cash Bond & Loan Tracker
- **Multiple Loan Types**: Cash Bond, Salary Loan, SSS Loan, Pag-IBIG Loan, Company Loan.
- **Auto-Deduction**: Active loans are automatically deducted from net pay during payroll processing (split by 2 for semi-monthly).
- **Payment Tracking**: Full payment history per loan — auto-deducted payments linked to payslips, manual payments with notes.
- **Auto-Close**: Loans automatically marked as "Fully Paid" when remaining balance hits zero.
- **Dashboard Summary**: Total active loans, outstanding balance, monthly deduction totals.

### 8. Shift Scheduling & Roster
- **Shift Templates**: Create reusable shifts (Morning, Mid, Night, Custom) with start/end times, break windows, and color coding.
- **Weekly Calendar Roster**: Visual grid showing all employees × 7 days. Click any cell to quick-assign a shift.
- **Bulk Assignment**: Assign a shift to multiple employees across a date range, with automatic rest-day skipping.
- **Global Fallback**: If no shift is assigned, the system falls back to the global `standard_shift_start/end` from PayrollConfiguration.

### 9. Anomaly Detection
  - **Zero AI / Zero RAM Overhead**: Pure SQL queries and threshold rules — no machine learning libraries.
  - **Unified Filtering**: Optimized to support the same date range and multi-employee filters as the rest of the analytics suite.
  - **Ghost OT Detection**: Flags overtime > 3 hours on days with no approved fieldwork.
  - **Missing Checkout**: Flags clock-in records with no corresponding clock-out.
  - **Excessive Lateness**: Flags employees late 5+ times in a single pay period.
  - **Unapproved Rest Day Work**: Flags work logged on rest days without fieldwork approval.
  - **Severity Badges**: High 🔴, Medium 🟡, Low 🔵 — sorted by priority.

### 10. Payroll Simulation Mode
- **Dry-Run Engine**: Preview payroll results without saving — no data is committed.
- **Salary Adjustments**: Apply percentage-based salary changes to see their impact.
- **Bonus Overrides**: Test bonus amounts before committing.
- **Comparison View**: Side-by-side simulated payslips with totals.

### 11. Real-Time Cost Dashboard
- **Privacy Guard**: Sensitive financial metrics (Branch Overview, Labor Cost, YTD Summary) are protected by a password-reveal lock.
  - **Branch Comparison**: Side-by-side cards comparing headcount, total gross, average salary, and overtime across branches.
  - **Monthly Labor Cost Chart**: Interactive **Stacked Bar Chart** showing payroll expenses segmented by branch (`AAA`, `Bizmaker`, and `Gamma`) for clear financial attribution.
  - **YTD Summary**: Gross, Net, Overtime, and Bonuses totals for the year.

### 12. Government Remittance Files
- **SSS R3 Report**: CSV export with EE/ER shares and EC contributions.
- **PhilHealth RF-1**: CSV export with premium splits per employee.
- **Pag-IBIG MCRF**: CSV export with contribution breakdowns.
- **One-Click Download**: Dropdown menu on processed payroll periods for instant generation.
- **Automatic Branch Cost Comparison**: Side-by-side analytics for `AAA and Co., CPAs`, `Bizmaker Consultancy, Inc.`, and `Gamma Oracle Dimensions Inc.`, including headcount, average salary, and budget utilization.
- **Monthly Total Aggregation**: Combines both semi-monthly cycles into a single remittance file, aggregating per employee.

### 13. Global Holiday Synchronization
- **Nager.Date API Integration**: Automatically synchronizes official Philippine public holidays for any given year into the local database.
- **Holy Week Readiness**: Native handling for Maundy Thursday, Good Friday, and Black Saturday.
- **"Resting" Logic**: Employees are automatically categorized as "Resting" instead of "Absent" on official holidays, preventing false-positive attendance reports.
- **One-Click Sync**: Admin-accessible sync button in settings to pull latest government-declared dates.

### 14. Premium Micro-Animations
- **Reactive Chart Re-renders**: Every analytics chart (Donut, Stacked Bar) utilizes a reactive-key strategy. When data loads, the chart performs a smooth "form-out" drawing animation rather than appearing statically.
- **Glassmorphic Feedback**: Smoother transitions and hover states for all interactive KPI cards.

## Getting Started

### Quick Start (Windows)
1. Ensure Python and Node.js are installed.
2. Run the startup script:
   - Double-click `Run_BizMaker.bat` in the root directory.
   - Alternatively, run `.\Run_BizMaker.ps1` in PowerShell.

### Required Dependencies

#### Backend (Python/Django)
- `django`, `djangorestframework`, `djangorestframework-simplejwt`
- `pyzk`: Critical for ZKTeco hardware communication.
- `Pillow`: Required for profile photo image processing.
- `django-cors-headers`: Enables frontend-backend cross-origin requests.
- `python-dotenv`: Management of environment variables.
- `openpyxl`, `pandas`: Powering the Excel and CSV export/import engines.

#### Frontend (Vue/Vite)
- `element-plus`: Core UI framework.
- `pinia`: Global state management.
- `apexcharts`: Interactive dashboard visualizations.
- `jspdf`, `jspdf-autotable`: PDF generation for payslips and reports.
- `xlsx`: Excel spreadsheet generation.

### Manual Setup

#### Backend
```bash
cd payroll_backend
# Set up virtual environment
python -m venv venv
.\venv\Scripts\activate
# Install dependencies
pip install -r requirements.txt
# Run migrations and start server
python manage.py migrate
python manage.py runserver
```

#### Frontend
```bash
cd payroll_frontend_new
npm install
npm run dev
```

## Security Configuration

The application uses environment variables for sensitive settings. Create a `.env` file in the root directory:

```env
SECRET_KEY=your_secure_key_here
DEBUG=True
DB_NAME=payroll_db
DB_USER=payroll_user
DB_PASSWORD=payroll_pass
DB_HOST=localhost
DB_PORT=5432
FIELD_ENCRYPTION_KEY=your_32_byte_base64_encoded_key # Mandatory for production
```

## Project Structure

- `payroll_backend/`: Django project housing logic for employees, attendance, payroll, loans, and shifts.
- `payroll_frontend_new/`: Vue 3 application for the administrative dashboard, scheduling, and reporting.
- `attendance_records/`: Local storage for raw biometric log backups.

## Code Quality & Reviews

This project uses **CodeRabbit AI** for automated pull request reviews. 

### Configuration
The review behavior is governed by the [.coderabbit.yaml](.coderabbit.yaml) file, which:
- Targets Django and Vue 3 best practices.
- Ignores irrelevant paths like `venv/`, `node_modules/`, and attendance log backups.
- Focuses on security and DOLE/BIR compliance.

### Enforcement Policy
To maintain high code quality, **every push must be reviewed by CodeRabbit before it is implemented (merged)** into the `main` branch. 
- While branch protection (blocking merge) is gated by GitHub Team/Enterprise for private repos, this project follows a strict **policy-based enforcement**.
- **Developers must not merge** until the CodeRabbit status check shows a "Success" or "Approved" state in the PR.
- All changes must go through a Pull Request for audit and AI review.

### How to use
When you open a Pull Request on GitHub, CodeRabbit will automatically analyze your changes and provide:
- High-level summaries of the PR.
- Line-by-line feedback and suggestions.
- Security and performance audits.

## License

Copyright (c) 2026 BizMaker. Private Repository.