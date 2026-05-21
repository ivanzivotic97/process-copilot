# Process Copilot
**AI-Powered Deviation Management for Pharmaceutical Manufacturing**

## Overview

Process Copilot is an intelligent deviation management system designed for pharmaceutical manufacturing quality operations. Built entirely on Microsoft Power Platform, it automates deviation classification, risk assessment, and CAPA generation using AI Builder (GPT-4o mini), reducing manual review time by 99% while ensuring zero missed high-severity escalations.

## Business Problem

Pharmaceutical manufacturers must manage hundreds of manufacturing deviations monthly under strict GMP regulatory requirements (ICH Q10, FDA 21 CFR 211, EU GMP Annex 1). Traditional manual processes create critical operational bottlenecks:

- **Classification delays**: Quality specialists spend 2-4 hours per deviation determining severity and escalation requirements
- **Escalation risk**: High-severity deviations (sterility breaches, contamination events) take 24-48 hours to reach leadership, creating patient safety and compliance exposure
- **CAPA bottleneck**: Root cause analysis and corrective/preventive action planning requires 1-2 days per deviation, delaying closure and repeat-risk mitigation
- **Resource strain**: A site processing 200 deviations/month consumes approximately 400-500 hours of quality specialist time monthly (~2.5-3 FTEs)

## Solution Overview

### Before: Manual Process
```
Deviation logged → Quality specialist reviews (2-4 hrs) → Manual severity assignment
→ Email escalation if critical (24-48 hrs) → Subject matter expert assigned
→ Investigation and CAPA drafting (1-2 days) → Management review → Closure
```

### After: AI-Powered Process
```
Deviation logged → AI classification (10 sec) → Automatic severity assignment
→ Instant escalation if critical (<60 sec) → AI-generated CAPA draft (10 sec)
→ Quality specialist validation (15-30 min) → Management review → Closure
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER INTERACTION LAYER                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────────────┐              ┌──────────────────────┐    │
│  │   Power Apps         │              │  Copilot Studio      │    │
│  │   (Model-Driven)     │              │  (Conversational AI) │    │
│  │                      │              │                      │    │
│  │  - Deviation Entry   │              │  - Natural Language  │    │
│  │  - Case Management   │              │    Queries           │    │
│  │  - Review Interface  │              │  - "Show high        │    │
│  │                      │              │     severity items"  │    │
│  └──────────┬───────────┘              └──────────┬───────────┘    │
│             │                                     │                 │
└─────────────┼─────────────────────────────────────┼─────────────────┘
              │                                     │
              ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│                            DATA LAYER                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                    ┌───────────────────────┐                         │
│                    │  Microsoft Dataverse  │                         │
│                    │                       │                         │
│                    │  cr4ea_deviations     │                         │
│                    │  ├─ Title             │                         │
│                    │  ├─ Description       │                         │
│                    │  ├─ Severity          │                         │
│                    │  ├─ Status            │                         │
│                    │  ├─ AI_Summary        │                         │
│                    │  └─ AI_CAPA           │                         │
│                    └───────────┬───────────┘                         │
│                                │                                     │
└────────────────────────────────┼─────────────────────────────────────┘
                                 │ (Trigger on create)
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AUTOMATION & AI LAYER                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Power Automate Flow                        │  │
│  │                                                                │  │
│  │  Step 1: Trigger (New Deviation Record)                       │  │
│  │          │                                                     │  │
│  │          ▼                                                     │  │
│  │  Step 2: AI Prompt 1 - Classify Severity                      │  │
│  │          │  Input: Title + Description                        │  │
│  │          │  Output: severity, justification, confidence       │  │
│  │          │  (AI Builder - GPT-4o mini)                        │  │
│  │          ▼                                                     │  │
│  │  Step 3: AI Prompt 2 - Summarize Deviation                    │  │
│  │          │  Input: Title + Description                        │  │
│  │          │  Output: what_happened, potential_impact,          │  │
│  │          │          immediate_containment, info_needed        │  │
│  │          ▼                                                     │  │
│  │  Step 4: AI Prompt 3 - Suggest CAPA                           │  │
│  │          │  Input: Title + Description + Severity             │  │
│  │          │  Output: root_cause_category,                      │  │
│  │          │          immediate_corrections,                    │  │
│  │          │          corrective_actions,                       │  │
│  │          │          preventive_actions,                       │  │
│  │          │          effectiveness_check                       │  │
│  │          ▼                                                     │  │
│  │  Step 5: Update Dataverse Record                              │  │
│  │          │  Write AI outputs to record fields                 │  │
│  │          ▼                                                     │  │
│  │  Step 6: Conditional - If Severity = High                     │  │
│  │          │                                                     │  │
│  │          ▼                                                     │  │
│  │  Step 7: Send Email Alert (Outlook)                           │  │
│  │          To: Quality Leadership                               │  │
│  │          Subject: HIGH SEVERITY DEVIATION - {Title}           │  │
│  │                                                                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## Key Features

- **Automated Severity Classification**: AI analyzes deviation description and assigns severity (Low/Medium/High) based on GMP risk criteria including patient safety, sterility, contamination, and regulatory impact
- **Instant Escalation**: High-severity deviations trigger automatic email alerts to quality leadership within 60 seconds of logging
- **AI-Generated CAPA**: Automated root cause categorization and generation of immediate corrections, corrective actions, preventive actions, and effectiveness checks
- **Regulatory-Aligned Logic**: Classification rules embedded in AI prompts align with ICH Q10, FDA 21 CFR 211, and EU GMP Annex 1 requirements
- **Natural Language Interface**: Copilot Studio agent enables conversational queries like "Show me all high-severity deviations from this month" or "Summarize the sterility breach deviation"
- **Structured JSON Output**: All AI responses return structured data for programmatic parsing and integration
- **No-Code Implementation**: Entire solution built using Microsoft Power Platform configuration without custom code development
- **Audit Trail**: All AI classifications and suggestions stored in Dataverse with timestamps for regulatory compliance

## AI Design

### Classification Logic

The severity classification prompt uses explicit GMP risk criteria:

**High Severity**:
- Direct patient safety risk (sterility breach, contamination, product recall potential)
- Critical quality events (cross-contamination, data integrity fraud, undetected product defects)
- Regulatory reportable events (EU GMP Annex 1 Grade A/B deviations, FDA Form 483 findings)

**Medium Severity**:
- Product quality impact without immediate patient risk (OOS lab results, equipment failure, process excursions)
- Containable deviations requiring investigation (batch segregation, reprocessing evaluation)
- Procedural non-conformances with quality impact potential

**Low Severity**:
- No product or patient impact (documentation errors, cosmetic defects caught before release, administrative issues)
- Post-verification findings with effective containment

**Ambiguity Handling**:
- Vague descriptions default to Medium (triggers investigation)
- Uncertain patient impact elevates to High (fail-safe approach)
- Boundary cases select higher severity (conservative risk posture)

### Prompt Engineering Approach

All three AI prompts follow consistent design principles:

1. **Role Definition**: Each prompt begins with explicit role assignment ("You are a pharmaceutical quality expert with 15+ years of GMP experience")
2. **Regulatory Context**: Prompts reference specific regulations (ICH Q10, FDA 21 CFR 211, EU GMP Annex 1) to ground AI reasoning
3. **Structured Output**: JSON schema specified in prompt instructions to enable programmatic parsing
4. **Few-Shot Examples**: Classification prompt includes 3-5 example deviations per severity level
5. **Constraint Guardrails**: Explicit instructions on handling edge cases, missing information, and ambiguity

### Safeguards

- **Human Validation Required**: All AI classifications and CAPA suggestions require quality specialist approval before investigation closure
- **Confidence Scoring**: Classification prompt returns confidence level (0-100%) to flag borderline cases for manual review
- **Escalation Bias**: Ambiguous cases default to higher severity to prevent under-escalation
- **Audit Logging**: All AI inputs and outputs stored in Dataverse for regulatory inspection and continuous model improvement
- **Versioned Prompts**: Prompt instructions maintained in version control with change history for validation documentation

## Example End-to-End Execution

### User Input (Power Apps)
```
Title: "Aseptic filling line pause due to visible particle in vial"
Description: "During routine visual inspection of vials on Line 3, operator observed a 
dark particle approximately 2mm in size floating in one vial. Line was immediately 
stopped. Batch #2024-05-001 (500 units produced, 450 units still in quarantine, 
50 units released to distribution). Particle type unknown. No other vials visually 
inspected yet."
```

### AI Output 1: Classification
```json
{
  "severity": "High",
  "justification": "Visible particulate contamination in a sterile injectable product 
  represents a direct patient safety risk. Partial batch release to distribution creates 
  potential for patient exposure. Meets criteria for High severity under EU GMP Annex 1 
  (particulate contamination in sterile products) and FDA 21 CFR 211.165 (testing and 
  release requirements).",
  "confidence": 95
}
```

### AI Output 2: Summary
```json
{
  "what_happened": "A 2mm dark particle was visually identified in one vial during 
  aseptic filling on Line 3. The operator stopped production immediately after discovery. 
  Batch #2024-05-001 had 500 units produced with 50 units already released to distribution.",
  
  "potential_impact": "Patient safety risk from injectable particulate contamination. 
  Potential product recall of 50 distributed units. Regulatory reportable event under 
  EU GMP and FDA regulations. Possible sterility or bioburden concerns if particle is 
  biological in origin.",
  
  "immediate_containment": "All 450 quarantined units must remain on hold pending full 
  investigation. Initiate immediate field alert and customer notification for 50 released 
  units. Quarantine all Line 3 components and raw materials. Conduct expanded visual 
  inspection of all units in batch and preceding batches from same line.",
  
  "additional_info_needed": "yes"
}
```

### AI Output 3: CAPA Suggestion
```json
{
  "root_cause_category": "Equipment Failure",
  
  "immediate_corrections": [
    "Within 4 hours: Complete visual inspection of all 500 units in Batch #2024-05-001 
    using magnifying inspection (100% sampling). Document results with photographic evidence.",
    "Within 8 hours: Conduct line clearance inspection of Line 3 including filling needles, 
    tubing, filters, and vial transport system. Collect particle for laboratory identification 
    (FTIR, SEM analysis).",
    "Within 24 hours: Initiate field alert notification to distributors of 50 released 
    units. Prepare voluntary recall plan if particle confirmed as quality defect."
  ],
  
  "corrective_actions": [
    "By Day 3: Perform root cause investigation including review of Line 3 maintenance 
    records, filter integrity test results, and environmental monitoring data for 7 days 
    prior. Responsible: Quality Manager + Engineering Lead.",
    "By Day 5: If particle identified as equipment-related (e.g., gasket material, 
    filter fiber), replace all suspect components and conduct 3-batch validation runs with 
    enhanced inspection. Responsible: Manufacturing Lead + Validation Engineer.",
    "By Day 7: Complete laboratory particle identification and sterility testing. 
    If sterility compromised, expand investigation to bioburden monitoring and cleaning 
    validation. Responsible: QC Laboratory Manager."
  ],
  
  "preventive_actions": [
    "By Day 14: Implement automated vision inspection system on Line 3 as secondary 
    verification after manual inspection (if not currently in place). Responsible: 
    Engineering Director.",
    "By Day 21: Revise Line 3 preventive maintenance schedule to increase filter and 
    gasket change frequency by 25%. Update line clearance checklist to include specific 
    particle contamination risk assessment. Responsible: Maintenance Manager.",
    "By Day 30: Conduct GMP training for all Line 3 operators on particulate contamination 
    recognition, immediate response procedures, and batch disposition requirements. 
    Responsible: Training Coordinator + Quality Assurance."
  ],
  
  "effectiveness_check": "Method: Review Line 3 production data for 90 days post-CAPA 
  implementation. Metrics: (1) Zero visible particle deviations, (2) 100% compliance with 
  revised maintenance schedule, (3) 100% operator training completion with assessment 
  scores >85%. Timeline: 90-day review meeting scheduled for August 20, 2024. Responsible: 
  Quality Director.",
  
  "assumptions": [
    "Particle is equipment-related (not raw material or environmental contamination)",
    "No systemic sterility breach has occurred (pending sterility test results)",
    "Line 3 has existing automated vision inspection (if not, preventive action timeline 
    extends to 60 days for procurement and qualification)",
    "Batch #2024-05-001 is the only affected batch (pending expanded inspection of 
    prior batches)"
  ]
}
```

### Power Automate Actions
1. Flow triggered by new Dataverse record creation
2. AI classification executes in 3-5 seconds, severity set to "High"
3. AI summary executes in 4-6 seconds, stored in AI_Summary field
4. AI CAPA executes in 6-8 seconds, stored in AI_CAPA field
5. Dataverse record updated with all AI outputs (total elapsed time: 10-15 seconds)
6. Conditional branch evaluates severity = High → triggers email action
7. Email sent to quality.leadership@company.com with subject "HIGH SEVERITY DEVIATION - Aseptic filling line pause due to visible particle in vial" within 60 seconds of initial logging

## Business Value

### Time Savings per Deviation

| Process Step | Manual Process | AI-Powered Process | Time Savings | Reduction |
|--------------|----------------|-------------------|--------------|-----------|
| Severity Classification | 2-4 hours | 10 seconds | ~3 hours | 99%+ |
| High-Severity Escalation | 24-48 hours | <60 seconds | ~36 hours | 99%+ |
| CAPA Draft Generation | 1-2 days | 10 seconds | ~1.5 days | 99%+ |
| Quality Specialist Validation | N/A (included above) | 15-30 minutes | N/A | N/A |
| **Total per Deviation** | **3-5 days** | **30-45 minutes** | **3-4 days** | **95%+** |

### Organizational Impact (200 deviations/month baseline)

**Current State**:
- Manual classification and CAPA: 400-500 hours/month (~2.5-3 FTEs dedicated to deviation processing)
- High-severity escalation delays: 12-18 critical events per month with 24-48 hour leadership notification lag
- Quality specialist bandwidth constraint limits deviation closure rate to 180-200/month, creating backlog during high-volume periods

**Future State with Process Copilot**:
- AI classification and CAPA draft: <1 hour/month (fully automated)
- Validation and approval: 50-100 hours/month (~0.6-0.8 FTEs)
- **Net time savings: 350-400 hours/month (equivalent to 2-2.5 FTE reallocation)**
- High-severity escalation: <60 seconds for all critical events (zero missed escalations, 99% reduction in notification time)
- Deviation closure capacity increased to 300+ per month with same headcount

**Financial Summary**:
- **Labor savings**: $200-250K annually (assuming $100K blended rate per quality FTE)
- **Risk mitigation value**: Elimination of delayed high-severity escalations reduces regulatory exposure and potential patient safety incidents (non-quantified but significant)
- **Throughput improvement**: 50% increase in deviation closure capacity enables faster batch release and reduced quarantine holding costs
- **Implementation cost**: ~40-60 hours for Power Platform configuration + AI Builder licensing (~$5-10/user/month incremental to Power Apps licensing)
- **Payback period**: <2 months

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Data Storage | Microsoft Dataverse | Deviation records, AI outputs, audit trail |
| Data Entry Interface | Power Apps (Model-Driven) | Structured deviation logging and case management |
| Automation Engine | Power Automate (Cloud Flow) | Orchestration of AI classification, CAPA generation, and escalation |
| AI Classification | AI Builder (GPT-4o mini) | Severity classification with regulatory logic |
| AI Summarization | AI Builder (GPT-4o mini) | Structured deviation summary generation |
| AI CAPA Generation | AI Builder (GPT-4o mini) | Root cause categorization and corrective/preventive action suggestions |
| Conversational Interface | Copilot Studio | Natural language queries and conversational analytics |
| Notifications | Outlook (Office 365) | High-severity email alerts |
| Security & Compliance | Azure Active Directory, Dataverse Security Roles | Authentication, authorization, field-level security |
| Licensing | Power Apps Premium + AI Builder | Per-user subscription model |

## Future Roadmap

### Phase 1: Foundation (Complete)
- Dataverse table design with deviation entity
- Power Automate flow with AI Builder integration
- Severity classification and CAPA generation
- High-severity email escalation
- Copilot Studio conversational interface

### Phase 2: Enhanced Analytics (3-6 months)
- Power BI dashboard for deviation trends and classification accuracy
- AI performance monitoring: track classification accuracy, confidence scores, and human override rates
- Trend analysis: identify recurring root causes and high-risk processes
- Regulatory reporting templates: automated deviation summary reports for FDA inspections and management reviews

### Phase 3: Expanded Scope (6-12 months)
- Document intelligence: auto-extract deviation details from PDF investigation reports using AI Builder Document Processing
- Regulatory knowledge integration: connect AI prompts to FDA Warning Letters and EudraLex database for real-time regulatory guidance
- Predictive analytics: ML model to predict likelihood of deviation occurrence based on process parameters and environmental conditions
- Multi-site deployment: extend to additional manufacturing sites with site-specific severity criteria and escalation workflows

### Phase 4: Advanced Capabilities (12+ months)
- Integration with MES/ERP: bi-directional data sync with manufacturing execution systems for automatic deviation triggering based on process alarms
- Advanced NLP: fine-tuned GPT model trained on historical deviation data for organization-specific classification and CAPA quality
- Validation automation: AI-assisted investigation workflow with automated evidence collection, timeline generation, and closure checklist
- Regulatory submission: auto-generate deviation sections for regulatory filings (annual product quality reviews, variation applications)

## Deployment Guide

### Prerequisites

**Licensing Requirements**:
- Microsoft 365 E3/E5 or equivalent (for Outlook integration)
- Power Apps Premium licenses (one per user)
- AI Builder capacity allocation (minimum 10K AI Builder credits for pilot)
- Copilot Studio licenses (if deploying conversational interface)

**Environment Setup**:
- Microsoft Dataverse environment (Production or Sandbox)
- System Administrator or System Customizer role in Power Platform
- Maker role in Copilot Studio (if deploying agent)

**Skills Required**:
- Power Apps model-driven app configuration (intermediate level)
- Power Automate cloud flow development (intermediate level)
- AI Builder prompt engineering (basic to intermediate level)
- Dataverse data modeling (basic level)

### Setup Steps

#### 1. Dataverse Table Configuration (30 minutes)

1. Navigate to Power Apps maker portal (make.powerapps.com)
2. Select target environment
3. Go to **Tables** > **New table** > **Set properties**:
   - Display name: `Deviations`
   - Plural name: `Deviations`
   - Primary column: `Title` (Text)
   - Enable options: Attachments, Notes, Activities
4. Create custom columns (see `/config/table-schema.json` for complete definition):
   - `Description` (Multiple Lines of Text)
   - `Severity` (Choice: Low, Medium, High)
   - `Status` (Choice: Open, Under Investigation, Closed)
   - `AI_Summary` (Multiple Lines of Text, 4000 char limit)
   - `AI_CAPA` (Multiple Lines of Text, 8000 char limit)
   - `Classification_Confidence` (Decimal Number, 0-100)
   - `Classified_Date` (Date and Time)
5. Configure choice field values:
   - Severity: Low (919860000), Medium (919860001), High (919860002)
   - Status: Open (919860000), Under Investigation (919860001), Closed (919860002)
6. Save and publish table

#### 2. AI Builder Prompt Configuration (60 minutes)

1. Navigate to AI Builder in Power Apps maker portal
2. Create three custom prompts (see `/docs/ai-prompts.md` for full prompt text):

**Prompt 1: Classify Deviation Severity**
   - Prompt name: `ClassifyDeviationSeverity`
   - Input: `Title` (Text), `Description` (Text)
   - Output: JSON (severity, justification, confidence)
   - Model: GPT-4o mini
   - Copy classification prompt instructions from `/docs/ai-prompts.md`
   - Test with sample deviation: "Batch record missing operator signature"
   - Expected output: `{"severity": "Low", "justification": "...", "confidence": 85}`

**Prompt 2: Summarize Deviation**
   - Prompt name: `SummarizeDeviation`
   - Input: `Title` (Text), `Description` (Text)
   - Output: JSON (what_happened, potential_impact, immediate_containment, additional_info_needed)
   - Model: GPT-4o mini
   - Copy summarization prompt instructions from `/docs/ai-prompts.md`
   - Test with sample deviation: "Temperature excursion in cold storage"

**Prompt 3: Suggest CAPA**
   - Prompt name: `SuggestCAPA`
   - Input: `Title` (Text), `Description` (Text), `Severity` (Text)
   - Output: JSON (root_cause_category, immediate_corrections, corrective_actions, preventive_actions, effectiveness_check, assumptions)
   - Model: GPT-4o mini
   - Copy CAPA prompt instructions from `/docs/ai-prompts.md`
   - Test with sample deviation: "Equipment failure on tablet press"

3. Publish all three prompts

#### 3. Power Automate Flow Development (90 minutes)

1. Navigate to Power Automate (make.powerautomate.com)
2. Create new **Automated cloud flow**:
   - Flow name: `Process Copilot - Deviation Classification and CAPA`
   - Trigger: **When a row is added, modified or deleted** (Dataverse)
     - Change type: `Added`
     - Table name: `Deviations`
     - Scope: `Organization`
3. Add action: **Create a prompt** (AI Builder)
   - Prompt: `ClassifyDeviationSeverity`
   - Title input: `Title` (from trigger)
   - Description input: `Description` (from trigger)
4. Add action: **Parse JSON** (Data Operations)
   - Content: `outputs from classification prompt`
   - Schema: (see `/config/flow-definition.md` for JSON schema)
5. Add action: **Create a prompt** (AI Builder)
   - Prompt: `SummarizeDeviation`
   - Title input: `Title` (from trigger)
   - Description input: `Description` (from trigger)
6. Add action: **Parse JSON** (Data Operations)
   - Content: `outputs from summarization prompt`
   - Schema: (see `/config/flow-definition.md`)
7. Add action: **Create a prompt** (AI Builder)
   - Prompt: `SuggestCAPA`
   - Title input: `Title` (from trigger)
   - Description input: `Description` (from trigger)
   - Severity input: `severity` (from classification parse JSON output)
8. Add action: **Parse JSON** (Data Operations)
   - Content: `outputs from CAPA prompt`
   - Schema: (see `/config/flow-definition.md`)
9. Add action: **Update a row** (Dataverse)
   - Table name: `Deviations`
   - Row ID: `Row ID` (from trigger)
   - Fields:
     - Severity: `severity` (from classification parse JSON)
     - AI_Summary: `concat all fields from summary parse JSON`
     - AI_CAPA: `concat all fields from CAPA parse JSON`
     - Classification_Confidence: `confidence` (from classification parse JSON)
     - Classified_Date: `utcNow()`
10. Add action: **Condition** (Control)
    - Condition: `severity` (from classification parse JSON) `is equal to` `High`
    - If yes:
      - Add action: **Send an email (V2)** (Office 365 Outlook)
        - To: `quality.leadership@yourcompany.com`
        - Subject: `HIGH SEVERITY DEVIATION - @{triggerOutputs()?['body/cr4ea_title']}`
        - Body: (see `/config/flow-definition.md` for email template)
11. Save flow
12. Test with sample deviation record

#### 4. Power Apps Model-Driven App (45 minutes)

1. In Power Apps maker portal, go to **Apps** > **New app** > **Model-driven**
2. Name: `Process Copilot`
3. Add **Deviations** table to sitemap:
   - Create area: `Deviations`
   - Create group: `Management`
   - Add subarea: `Deviations` table
4. Configure main form:
   - Add all columns: Title, Description, Severity, Status, AI_Summary, AI_CAPA, Classification_Confidence, Classified_Date
   - Create sections: Deviation Details, AI Analysis, Investigation
   - Make AI_Summary and AI_CAPA read-only (users validate but cannot edit AI outputs)
5. Configure views:
   - **My Open Deviations**: Filter Status = Open
   - **High Severity Deviations**: Filter Severity = High
   - **Deviations Requiring Review**: Filter Status = Under Investigation
6. Publish app
7. Share app with quality team users (assign appropriate security roles)

#### 5. Copilot Studio Agent (Optional, 60 minutes)

1. Navigate to Copilot Studio (copilotstudio.microsoft.com)
2. Create new copilot:
   - Name: `Process Copilot Assistant`
   - Language: English
3. Configure **Knowledge** > **Dataverse**:
   - Connect to Dataverse environment
   - Select `Deviations` table
   - Enable fields: Title, Description, Severity, Status, AI_Summary, AI_CAPA
4. Create sample topics:
   - **Show high severity deviations**: Trigger phrase "show high severity", action: query Dataverse where Severity = High
   - **Summarize deviation**: Trigger phrase "summarize deviation [Title]", action: query Dataverse by Title, return AI_Summary field
   - **CAPA for deviation**: Trigger phrase "CAPA for [Title]", action: query Dataverse by Title, return AI_CAPA field
5. Test in authoring canvas
6. Publish copilot
7. Deploy to Teams or embed in Power Apps

### Demo Walkthrough

**Scenario: Log a medium-severity deviation and observe AI automation**

1. Open Process Copilot app in Power Apps
2. Click **New** to create deviation record
3. Enter sample data:
   - Title: `Batch record missing operator signature on dispensing log`
   - Description: `During final batch record review for Batch #2024-05-015 (Product: Amoxicillin 500mg tablets), Quality Assurance identified that the operator signature is missing from the raw material dispensing log on page 12. The weighing data and material lot numbers are present and correct. The operator (John Smith, Employee ID: 12345) confirmed via email that he performed the weighing on May 15, 2024 at 10:30 AM and forgot to sign. No product quality impact suspected as all material quantities reconcile correctly.`
4. Click **Save** (do not fill in Severity or AI fields - these will be auto-populated)
5. Wait 10-15 seconds and refresh the record
6. Observe AI automation results:
   - **Severity**: Auto-set to `Low`
   - **AI_Summary**: "The operator forgot to sign the dispensing log for Batch #2024-05-015, though all weighing data and lot numbers are documented correctly... Potential impact: No direct product quality or patient safety risk as material traceability is maintained... Immediate containment: Obtain retroactive signature from operator with explanation memo..."
   - **AI_CAPA**: "root_cause_category: Human Error, immediate_corrections: ['Obtain signed affidavit from operator confirming material dispensing activities...'], corrective_actions: ['Implement secondary verification step for batch record review before QA approval...'], preventive_actions: ['Conduct refresher training on GMP documentation requirements...'], effectiveness_check: 'Audit next 50 batch records for signature completeness...'"
   - **Classification_Confidence**: `88`
7. Check email inbox: No email received (only High severity triggers alerts)
8. Update **Status** to `Under Investigation` and assign to quality specialist for validation
9. Open Copilot Studio agent (if deployed): Ask "Show me all low severity deviations" → observe Dataverse query results

**Expected Outcome**: Deviation classified, summarized, and CAPA-drafted in <15 seconds with no manual intervention. Quality specialist validates AI outputs and proceeds directly to investigation rather than spending 2-4 hours on manual analysis.

## Screenshots

| Screenshot | Description | File Path |
|------------|-------------|-----------|
| Power Apps - Deviation Entry Form | Model-driven app form showing Title, Description, and AI-populated fields (Severity, AI_Summary, AI_CAPA) | `/screenshots/01-deviation-entry-form.png` |
| Power Apps - High Severity View | Filtered view showing all High severity deviations with color-coded severity indicator | `/screenshots/02-high-severity-view.png` |
| Power Automate - Flow Overview | Flow canvas showing 10-step automation from Dataverse trigger to email alert | `/screenshots/03-flow-overview.png` |
| AI Builder - Classification Prompt | GPT-4o mini prompt configuration with input/output schema and test results | `/screenshots/04-classification-prompt.png` |
| AI Builder - CAPA Prompt | GPT-4o mini CAPA prompt showing structured JSON output with root cause and action items | `/screenshots/05-capa-prompt.png` |
| Email Alert - High Severity | Sample Outlook email alert sent when High severity deviation is detected | `/screenshots/06-email-alert.png` |
| Copilot Studio - Conversational Query | Copilot agent responding to "Show high severity deviations" with Dataverse query results | `/screenshots/07-copilot-query.png` |
| Dataverse - Table Schema | Deviations table schema showing all custom columns and choice field configurations | `/screenshots/08-dataverse-schema.png` |

**Note**: Screenshots are illustrative placeholders. Actual implementation screenshots should be captured during deployment and added to `/screenshots/` directory.

---

## About This Repository

This repository documents a proof-of-concept implementation of Process Copilot designed for portfolio demonstration and consulting discussions. It showcases Microsoft Power Platform architecture, AI Builder prompt engineering, and pharmaceutical domain expertise.

**Repository Contents**:
- `/docs/` - Detailed architecture, AI prompt specifications, and business case analysis
- `/config/` - Dataverse table schema and Power Automate flow definitions
- `/screenshots/` - Visual documentation of implementation (to be added during deployment)

**Contact**: For questions about this implementation approach or potential consulting engagements, please open an issue or reach out via [your professional contact method].

**Disclaimer**: This is a conceptual design and proof-of-concept documentation. Actual deployment in a GMP-regulated environment requires validation according to GAMP 5 guidelines, including Installation Qualification (IQ), Operational Qualification (OQ), Performance Qualification (PQ), and ongoing change control. AI outputs require human validation by qualified personnel before use in regulatory decision-making.
