# Process Copilot Architecture

## Overview

Process Copilot is a serverless, event-driven architecture built entirely on Microsoft Power Platform. The system leverages Dataverse as the central data repository, Power Automate for orchestration, AI Builder for intelligent classification and generation, and Power Apps/Copilot Studio for user interaction.

## Component Architecture

```
┌───────────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION LAYER                                 │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────┐           ┌─────────────────────────┐      │
│   │   Power Apps            │           │   Copilot Studio        │      │
│   │   (Model-Driven)        │           │                         │      │
│   ├─────────────────────────┤           ├─────────────────────────┤      │
│   │ • Deviation Entry Form  │           │ • Natural Language      │      │
│   │ • Case Management UI    │           │   Processing            │      │
│   │ • Review Workflow       │           │ • Conversational        │      │
│   │ • Audit Trail Viewer    │           │   Queries               │      │
│   │ • Dashboard (Future)    │           │ • Dataverse Integration │      │
│   └────────────┬────────────┘           └────────────┬────────────┘      │
│                │                                      │                    │
│                │ CRUD Operations                      │ Query API          │
│                │                                      │                    │
└────────────────┼──────────────────────────────────────┼────────────────────┘
                 │                                      │
                 ▼                                      ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER                                        │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                     ┌────────────────────────────┐                         │
│                     │   Microsoft Dataverse      │                         │
│                     ├────────────────────────────┤                         │
│                     │  Table: cr4ea_deviations   │                         │
│                     │                            │                         │
│                     │  Core Fields:              │                         │
│                     │  ├─ cr4ea_deviationsid     │ (Primary Key, GUID)    │
│                     │  ├─ cr4ea_title            │ (Text, 200 char)       │
│                     │  ├─ cr4ea_description      │ (Multiline, 4000 char) │
│                     │  ├─ cr4ea_severity         │ (Choice: L/M/H)        │
│                     │  ├─ cr4ea_status           │ (Choice: Open/Inv/Cls) │
│                     │  ├─ createdon              │ (DateTime, System)     │
│                     │  ├─ modifiedon             │ (DateTime, System)     │
│                     │  └─ ownerid                │ (Lookup, User/Team)    │
│                     │                            │                         │
│                     │  AI Output Fields:         │                         │
│                     │  ├─ cr4ea_ai_summary       │ (Multiline, 4000 char) │
│                     │  ├─ cr4ea_ai_capa          │ (Multiline, 8000 char) │
│                     │  ├─ cr4ea_confidence       │ (Decimal, 0-100)       │
│                     │  └─ cr4ea_classified_date  │ (DateTime)             │
│                     │                            │                         │
│                     │  Business Features:        │                         │
│                     │  ├─ Attachments: Enabled   │                         │
│                     │  ├─ Notes: Enabled         │                         │
│                     │  ├─ Activities: Enabled    │                         │
│                     │  ├─ Audit: Enabled         │                         │
│                     │  └─ Change Tracking: On    │                         │
│                     └──────────┬─────────────────┘                         │
│                                │                                            │
│                                │ (OnCreate Trigger)                         │
│                                │                                            │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATION LAYER                                   │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │              Power Automate Cloud Flow                           │    │
│   │              "Process Copilot - Deviation Processor"             │    │
│   ├─────────────────────────────────────────────────────────────────┤    │
│   │                                                                   │    │
│   │  [1] Trigger: Dataverse Record Created                           │    │
│   │      ├─ Table: cr4ea_deviations                                  │    │
│   │      ├─ Change Type: Added                                       │    │
│   │      └─ Scope: Organization                                      │    │
│   │                                                                   │    │
│   │  [2] Initialize Variables                                        │    │
│   │      ├─ var_title = trigger.cr4ea_title                          │    │
│   │      ├─ var_description = trigger.cr4ea_description              │    │
│   │      └─ var_record_id = trigger.cr4ea_deviationsid               │    │
│   │                                                                   │    │
│   │  [3] AI Prompt: Classify Severity ────────┐                     │    │
│   │      ├─ Input: var_title, var_description  │                     │    │
│   │      └─ Output: JSON (severity,            │                     │    │
│   │                justification, confidence)  │                     │    │
│   │                                             ▼                     │    │
│   │  [4] Parse JSON: Classification Output ────┤                     │    │
│   │      ├─ var_severity = body.severity       │                     │    │
│   │      ├─ var_justification =                │                     │    │
│   │      │   body.justification                │                     │    │
│   │      └─ var_confidence = body.confidence   │                     │    │
│   │                                             │                     │    │
│   │  [5] AI Prompt: Summarize Deviation ───────┤                     │    │
│   │      ├─ Input: var_title, var_description  │                     │    │
│   │      └─ Output: JSON (what_happened,       │                     │    │
│   │                potential_impact, etc.)     │                     │    │
│   │                                             ▼                     │    │
│   │  [6] Parse JSON: Summary Output ───────────┤                     │    │
│   │      ├─ var_what_happened =                │                     │    │
│   │      │   body.what_happened                │                     │    │
│   │      ├─ var_potential_impact =             │                     │    │
│   │      │   body.potential_impact             │                     │    │
│   │      └─ var_containment =                  │                     │    │
│   │          body.immediate_containment        │                     │    │
│   │                                             │                     │    │
│   │  [7] AI Prompt: Suggest CAPA ──────────────┤                     │    │
│   │      ├─ Input: var_title,                  │                     │    │
│   │      │   var_description, var_severity     │                     │    │
│   │      └─ Output: JSON (root_cause_category, │                     │    │
│   │                corrections, capa, etc.)    │                     │    │
│   │                                             ▼                     │    │
│   │  [8] Parse JSON: CAPA Output ──────────────┤                     │    │
│   │      ├─ var_root_cause =                   │                     │    │
│   │      │   body.root_cause_category          │                     │    │
│   │      ├─ var_corrections =                  │                     │    │
│   │      │   body.immediate_corrections        │                     │    │
│   │      └─ var_corrective_actions =           │                     │    │
│   │          body.corrective_actions           │                     │    │
│   │                                             │                     │    │
│   │  [9] Compose: Format AI Outputs ───────────┤                     │    │
│   │      ├─ Summary Text =                     │                     │    │
│   │      │   concat(what_happened,             │                     │    │
│   │      │   potential_impact, containment)    │                     │    │
│   │      └─ CAPA Text =                        │                     │    │
│   │          concat(root_cause, corrections,   │                     │    │
│   │          corrective_actions, etc.)         │                     │    │
│   │                                             │                     │    │
│   │  [10] Update Dataverse Record ─────────────┤                     │    │
│   │       ├─ Record ID: var_record_id          │                     │    │
│   │       ├─ Severity: var_severity            │                     │    │
│   │       ├─ AI_Summary: Summary Text          │                     │    │
│   │       ├─ AI_CAPA: CAPA Text                │                     │    │
│   │       ├─ Confidence: var_confidence        │                     │    │
│   │       └─ Classified_Date: utcNow()         │                     │    │
│   │                                             │                     │    │
│   │  [11] Condition: Check Severity ───────────┤                     │    │
│   │       If var_severity == "High"            │                     │    │
│   │          ├─ Yes: Send Email Alert ─────────┘                     │    │
│   │          │    ├─ To: quality.leadership@   │                     │    │
│   │          │    │        company.com          │                     │    │
│   │          │    ├─ Subject: HIGH SEVERITY    │                     │    │
│   │          │    │   DEVIATION - {title}      │                     │    │
│   │          │    ├─ Body: Deviation details + │                     │    │
│   │          │    │   AI Summary + Justification│                     │    │
│   │          │    └─ Importance: High           │                     │    │
│   │          │                                   │                     │    │
│   │          └─ No: End flow                    │                     │    │
│   │                                              │                     │    │
│   │  [12] Flow Completion                       │                     │    │
│   │       ├─ Total runtime: 10-15 seconds       │                     │    │
│   │       └─ Success notification to flow owner │                     │    │
│   │                                              │                     │    │
│   └──────────────────────────────────────────────┘                    │    │
│                                                                         │    │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ (Calls to AI Builder)
                                   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         AI LAYER                                           │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │                   AI Builder Service                             │    │
│   ├─────────────────────────────────────────────────────────────────┤    │
│   │                                                                   │    │
│   │  ┌───────────────────────────────────────────┐                  │    │
│   │  │  Custom Prompt 1:                          │                  │    │
│   │  │  "Classify Deviation Severity"             │                  │    │
│   │  ├───────────────────────────────────────────┤                  │    │
│   │  │  Model: GPT-4o mini                        │                  │    │
│   │  │  Temperature: 0.3 (deterministic)          │                  │    │
│   │  │  Max Tokens: 500                           │                  │    │
│   │  │  Input Schema: title (text),               │                  │    │
│   │  │                description (text)          │                  │    │
│   │  │  Output Format: JSON                       │                  │    │
│   │  │  Prompt Engineering:                       │                  │    │
│   │  │    ├─ Role definition (pharma QA expert)  │                  │    │
│   │  │    ├─ Classification criteria (GMP rules) │                  │    │
│   │  │    ├─ Few-shot examples (5 per severity)  │                  │    │
│   │  │    └─ Edge case handling logic             │                  │    │
│   │  └───────────────────────────────────────────┘                  │    │
│   │                                                                   │    │
│   │  ┌───────────────────────────────────────────┐                  │    │
│   │  │  Custom Prompt 2:                          │                  │    │
│   │  │  "Summarize Deviation"                     │                  │    │
│   │  ├───────────────────────────────────────────┤                  │    │
│   │  │  Model: GPT-4o mini                        │                  │    │
│   │  │  Temperature: 0.5 (balanced)               │                  │    │
│   │  │  Max Tokens: 800                           │                  │    │
│   │  │  Input Schema: title (text),               │                  │    │
│   │  │                description (text)          │                  │    │
│   │  │  Output Format: JSON                       │                  │    │
│   │  │  Prompt Engineering:                       │                  │    │
│   │  │    ├─ Structured output (4 fields)        │                  │    │
│   │  │    ├─ Pharmaceutical context awareness    │                  │    │
│   │  │    └─ Containment action focus            │                  │    │
│   │  └───────────────────────────────────────────┘                  │    │
│   │                                                                   │    │
│   │  ┌───────────────────────────────────────────┐                  │    │
│   │  │  Custom Prompt 3:                          │                  │    │
│   │  │  "Suggest CAPA"                            │                  │    │
│   │  ├───────────────────────────────────────────┤                  │    │
│   │  │  Model: GPT-4o mini                        │                  │    │
│   │  │  Temperature: 0.6 (creative)               │                  │    │
│   │  │  Max Tokens: 1500                          │                  │    │
│   │  │  Input Schema: title (text),               │                  │    │
│   │  │                description (text),         │                  │    │
│   │  │                severity (text)             │                  │    │
│   │  │  Output Format: JSON                       │                  │    │
│   │  │  Prompt Engineering:                       │                  │    │
│   │  │    ├─ Root cause categorization (8 types) │                  │    │
│   │  │    ├─ ICH Q10 CAPA framework alignment    │                  │    │
│   │  │    ├─ Timeline and responsibility         │                  │    │
│   │  │    │   assignment                          │                  │    │
│   │  │    └─ Effectiveness check methodology     │                  │    │
│   │  └───────────────────────────────────────────┘                  │    │
│   │                                                                   │    │
│   │  Runtime Characteristics:                                        │    │
│   │  ├─ API Endpoint: Azure OpenAI Service (via AI Builder)         │    │
│   │  ├─ Authentication: Managed via Power Platform                  │    │
│   │  ├─ Rate Limits: 60 requests/minute per environment             │    │
│   │  ├─ Latency: 3-8 seconds per prompt call                        │    │
│   │  ├─ Retry Logic: 3 attempts with exponential backoff            │    │
│   │  └─ Error Handling: Falls back to manual classification         │    │
│   │                                                                   │    │
│   └───────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ (Email notification)
                                   ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                      NOTIFICATION LAYER                                    │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐    │
│   │                   Office 365 Outlook                             │    │
│   ├─────────────────────────────────────────────────────────────────┤    │
│   │                                                                   │    │
│   │  Email Alert Configuration:                                      │    │
│   │  ├─ Recipient: quality.leadership@company.com                   │    │
│   │  ├─ Subject: HIGH SEVERITY DEVIATION - {Title}                  │    │
│   │  ├─ Body (HTML formatted):                                      │    │
│   │  │   ├─ Deviation ID and Title                                  │    │
│   │  │   ├─ Classification: High (with confidence %)                │    │
│   │  │   ├─ AI Justification                                        │    │
│   │  │   ├─ Summary excerpt (what_happened + impact)                │    │
│   │  │   └─ Link to Power Apps record                               │    │
│   │  ├─ Importance: High (red exclamation in Outlook)               │    │
│   │  ├─ Delivery: Immediate (no delay)                              │    │
│   │  └─ Attachments: None (link to app instead)                     │    │
│   │                                                                   │    │
│   │  SLA Metrics:                                                    │    │
│   │  ├─ Trigger to email sent: <60 seconds                          │    │
│   │  ├─ Delivery to inbox: <10 seconds (Office 365)                 │    │
│   │  └─ Total escalation time: <70 seconds                          │    │
│   │                                                                   │    │
│   └───────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Description

### Primary Flow: Deviation Creation to AI Analysis

1. **User Interaction** (T+0 seconds)
   - Quality specialist opens Power Apps model-driven app
   - Navigates to Deviations entity
   - Clicks "New" to create record
   - Fills required fields: Title (single-line text), Description (multi-line text)
   - Leaves AI fields blank (Severity, AI_Summary, AI_CAPA)
   - Clicks "Save"

2. **Dataverse Record Creation** (T+0.5 seconds)
   - Power Apps creates new row in `cr4ea_deviations` table
   - System fields auto-populated: `cr4ea_deviationsid` (GUID), `createdon` (timestamp), `ownerid` (current user)
   - User-entered fields stored: `cr4ea_title`, `cr4ea_description`
   - AI fields remain null: `cr4ea_severity`, `cr4ea_ai_summary`, `cr4ea_ai_capa`
   - Dataverse audit log records creation event

3. **Power Automate Trigger** (T+1 second)
   - Dataverse connector fires "When a row is added" trigger
   - Flow instance initiated with trigger context containing new record ID and field values
   - Flow run ID generated for tracking and debugging

4. **Variable Initialization** (T+1.5 seconds)
   - Flow extracts `Title` and `Description` from trigger outputs
   - Stores record ID for subsequent update operation
   - All three AI prompt calls can now execute in pseudo-parallel (Power Automate processes sequentially but optimizes internally)

5. **AI Classification** (T+2-5 seconds)
   - Flow calls AI Builder custom prompt "Classify Deviation Severity"
   - Inputs passed: Title, Description
   - AI Builder constructs full prompt by interpolating inputs into prompt template
   - Request sent to Azure OpenAI Service (GPT-4o mini model)
   - Model analyzes text against embedded GMP classification rules
   - Returns JSON response: `{"severity": "Medium", "justification": "OOS result without immediate patient risk per ICH Q10 guidelines", "confidence": 87}`
   - Flow parses JSON using Parse JSON action

6. **AI Summarization** (T+5-9 seconds)
   - Flow calls AI Builder custom prompt "Summarize Deviation"
   - Inputs passed: Title, Description
   - Model generates structured summary covering: what happened, potential impact, immediate containment needs, missing information flags
   - Returns JSON response with 4 fields
   - Flow parses JSON output

7. **AI CAPA Generation** (T+9-15 seconds)
   - Flow calls AI Builder custom prompt "Suggest CAPA"
   - Inputs passed: Title, Description, Severity (from Step 5 output)
   - Model applies ICH Q10 CAPA framework to generate: root cause category, immediate corrections (24-hour timeline), corrective actions (with responsible parties), preventive actions (with responsible parties), effectiveness check methodology
   - Returns JSON response with 6 fields (including assumptions list)
   - Flow parses JSON output

8. **Data Transformation** (T+15 seconds)
   - Flow concatenates JSON fields into human-readable text for Dataverse storage
   - Summary text: `"What Happened: {what_happened}\n\nPotential Impact: {potential_impact}\n\nImmediate Containment: {immediate_containment}"`
   - CAPA text: `"Root Cause: {root_cause_category}\n\nImmediate Corrections:\n{corrections}\n\nCorrective Actions:\n{corrective_actions}\n\nPreventive Actions:\n{preventive_actions}\n\nEffectiveness Check:\n{effectiveness_check}"`

9. **Dataverse Update** (T+16 seconds)
   - Flow calls Dataverse "Update a row" action
   - Record ID from trigger used to locate original record
   - Fields updated:
     - `cr4ea_severity`: "Medium" (or mapped choice value 919860001)
     - `cr4ea_ai_summary`: Summary text (string)
     - `cr4ea_ai_capa`: CAPA text (string)
     - `cr4ea_confidence`: 87 (decimal)
     - `cr4ea_classified_date`: 2024-05-21T10:30:45Z (ISO 8601 timestamp)
   - Dataverse audit log records update event with old/new values

10. **Conditional Escalation** (T+17 seconds)
    - Flow evaluates condition: `severity == "High"`
    - If TRUE: Execute "Send an email (V2)" action
      - Recipient: `quality.leadership@company.com` (configured in flow)
      - Subject: `"HIGH SEVERITY DEVIATION - {Title}"`
      - Body: HTML-formatted email with AI classification justification, summary excerpt, and deep link to Power Apps record
      - Email sent via Office 365 Outlook connector
      - Delivery to inbox within 10 seconds
    - If FALSE: Skip email action, proceed to flow completion

11. **Flow Completion** (T+18 seconds)
    - Flow status set to "Succeeded"
    - Run history recorded in Power Automate with execution duration and action outputs
    - No user notification sent (silent background process)

12. **User Experience Return** (T+20 seconds)
    - Quality specialist refreshes Power Apps record (or record auto-refreshes if real-time updates enabled)
    - AI-populated fields now visible:
      - Severity badge displays "Medium" with amber color
      - AI Summary section shows structured text
      - AI CAPA section shows root cause and action items
    - Specialist reviews AI outputs, validates accuracy, and proceeds with investigation assignment

### Secondary Flow: Copilot Studio Query

1. **User Query** (Conversational Interface)
   - User types: "Show me all high severity deviations from this month"
   - Copilot Studio natural language understanding (NLU) layer parses intent: `query_deviations`
   - Entity extraction identifies: `severity_filter = High`, `date_filter = current_month`

2. **Dataverse Query Execution**
   - Copilot Studio constructs FetchXML or OData query:
     ```
     SELECT cr4ea_title, cr4ea_description, cr4ea_severity, createdon
     FROM cr4ea_deviations
     WHERE cr4ea_severity = 919860002 (High)
       AND createdon >= '2024-05-01T00:00:00Z'
       AND createdon < '2024-06-01T00:00:00Z'
     ORDER BY createdon DESC
     ```
   - Query sent to Dataverse via standard connector
   - Results returned (max 10 records by default to prevent token overflow)

3. **Response Generation**
   - Copilot Studio formats results into conversational response:
     ```
     I found 3 high severity deviations this month:
     
     1. Aseptic filling line pause due to visible particle (May 15)
     2. Sterility breach in Grade A cleanroom (May 8)
     3. Cross-contamination in tablet compression (May 3)
     
     Would you like details on any of these?
     ```
   - User can click/tap deviation title to open Power Apps record or ask follow-up: "Summarize the first one"

4. **Follow-Up Query**
   - User: "Summarize the first one"
   - Copilot retrieves `cr4ea_ai_summary` field for selected record
   - Returns summary text directly to user

## Design Decisions and Rationale

### 1. Model-Driven App vs. Canvas App

**Decision**: Use Power Apps model-driven app for deviation management interface.

**Rationale**:
- **Data-centric design**: Deviations are structured records with defined schema and relationships, ideal for model-driven approach
- **Built-in features**: Automatic audit trail, security roles, business process flows, and case management capabilities align with GMP requirements
- **Lower development effort**: Model-driven apps auto-generate forms, views, and dashboards from Dataverse schema with minimal customization
- **Long-term maintainability**: Schema changes in Dataverse automatically reflect in UI without manual form updates
- **Regulatory alignment**: Model-driven apps provide better traceability and validation documentation for 21 CFR Part 11 compliance

**Trade-off**: Less UI flexibility compared to canvas apps, but this is acceptable for a back-office quality management system where functionality outweighs aesthetics.

### 2. AI Builder (GPT-4o mini) vs. Azure OpenAI Direct Integration

**Decision**: Use AI Builder custom prompts with GPT-4o mini model rather than direct Azure OpenAI API calls.

**Rationale**:
- **No-code commitment**: Maintains zero custom code architecture, reducing validation burden for GMP system (GAMP 5 Category 4 instead of Category 5)
- **Native Power Platform integration**: AI Builder connectors available in Power Automate without custom connector development
- **License simplification**: AI Builder capacity included in Power Apps Premium licensing with predictable per-request pricing
- **Prompt versioning and management**: AI Builder provides UI-based prompt editor with built-in version control and testing
- **Faster time-to-value**: No need for Azure OpenAI service provisioning, API key management, or Azure subscription setup

**Trade-off**: Less control over model parameters (temperature, top_p, frequency_penalty) and limited to models exposed by AI Builder. GPT-4o mini is sufficient for classification/summarization tasks but may require upgrade to GPT-4 for more complex reasoning in future phases.

### 3. Three Separate AI Prompts vs. Single Compound Prompt

**Decision**: Implement three discrete AI Builder prompts (Classify, Summarize, CAPA) rather than one prompt that performs all tasks.

**Rationale**:
- **Separation of concerns**: Each prompt has distinct purpose, output schema, and temperature settings optimized for its task
- **Independent validation**: Quality specialists can validate classification accuracy separately from CAPA quality, enabling targeted prompt tuning
- **Reusability**: Summarize and CAPA prompts can be reused in other workflows (e.g., manual investigation review, regulatory report generation)
- **Prompt token optimization**: Shorter prompts with focused instructions reduce token consumption and latency per call
- **Failure isolation**: If CAPA prompt fails, classification and summary still succeed, allowing partial automation rather than complete failure

**Trade-off**: Higher total latency (3 sequential API calls vs. 1) and increased AI Builder credit consumption. However, total runtime remains <20 seconds which meets business requirements.

### 4. JSON Output Format vs. Natural Language Output

**Decision**: All AI prompts return structured JSON rather than natural language paragraphs.

**Rationale**:
- **Programmatic parsing**: Power Automate can extract specific fields (e.g., severity value, confidence score) using Parse JSON action without regex or text manipulation
- **Dataverse field mapping**: JSON structure aligns with Dataverse column types (choice fields, decimal numbers, multi-line text)
- **Downstream integration**: Structured data enables future Power BI reporting, Copilot Studio entity recognition, and API exports
- **Validation and error handling**: JSON schema validation in Parse JSON action catches malformed AI responses immediately
- **Consistency enforcement**: JSON schema constrains AI output to expected format, reducing variability in responses

**Trade-off**: Requires more detailed prompt engineering to specify JSON structure and enforce compliance. AI models occasionally return markdown-wrapped JSON which requires text trimming logic.

### 5. Synchronous Flow Execution vs. Asynchronous Processing

**Decision**: Power Automate flow executes synchronously (user waits during AI processing) rather than asynchronously with callback notification.

**Rationale**:
- **Simplicity**: Synchronous execution requires no queue management, job status tracking, or notification infrastructure
- **Fast enough**: Total flow runtime <20 seconds meets user expectation for real-time classification (users accustomed to 2-4 hour manual process)
- **Immediate validation**: Quality specialists can validate AI outputs immediately after saving record without waiting for background job completion
- **User experience**: Real-time feedback loop (save → refresh → see results) feels responsive and builds trust in AI accuracy

**Trade-off**: Flow failures or AI Builder service degradation result in visible delays to users. Asynchronous design would hide latency but add complexity. Mitigation: flow timeout set to 60 seconds with error notification to user if timeout occurs.

### 6. High Severity Email Alert vs. In-App Notification

**Decision**: Use email alert via Outlook for high severity deviations rather than in-app notification (e.g., Power Apps notification, Teams message).

**Rationale**:
- **Urgency alignment**: High severity deviations represent potential patient safety risk requiring immediate leadership attention; email alerts are universally monitored by quality leadership
- **Reach beyond app users**: Quality directors and regulatory affairs may not have Power Apps open but check email continuously
- **Audit trail**: Email provides durable record of escalation timestamp and recipient list for regulatory inspection
- **Existing workflows**: Pharmaceutical organizations have established email-based escalation procedures for critical events; no behavior change required

**Trade-off**: Email fatigue if classification accuracy is low (false positives on High severity). Mitigation: prompt engineering emphasizes conservative bias (prefer Medium over High when uncertain) and confidence score flagging for borderline cases.

### 7. Dataverse as Data Layer vs. SharePoint or SQL Database

**Decision**: Use Microsoft Dataverse as primary data repository rather than SharePoint lists or Azure SQL Database.

**Rationale**:
- **Native Power Platform integration**: Dataverse is the native data platform for Power Apps and Power Automate with zero configuration required
- **Security and compliance features**: Row-level security, field-level security, audit logging, and data loss prevention policies built-in
- **Relational data model**: Dataverse supports complex relationships (e.g., deviations linked to products, batch records, CAPA tasks) required for future phases
- **APIs and SDR**: Dataverse provides REST/OData APIs for external integrations and supports common data model (CDM) for data interoperability
- **No ETL required**: Real-time data availability for Power BI reporting without extraction or replication

**Trade-off**: Higher licensing cost compared to SharePoint (Power Apps Premium required vs. SharePoint included in M365). Dataverse storage limits (10GB per environment) may require capacity add-ons for large deviation volumes. Justifiable for GMP-regulated system requiring audit trail and security controls.

## Security and Compliance Considerations

### Authentication and Authorization

**Azure Active Directory (AAD) Integration**:
- All users authenticate via AAD single sign-on (SSO)
- Multi-factor authentication (MFA) enforced at organization level
- Conditional access policies can restrict access by device compliance, location, or risk score

**Dataverse Security Roles**:
- **Quality Specialist**: Create, read, update deviations owned by user or team; cannot delete or reassign after investigation start
- **Quality Manager**: Full CRUD permissions on all deviations; assign ownership; override AI classifications
- **Quality Director**: Read-only access to all deviations; export data for regulatory reporting; access audit logs
- **System Administrator**: Full administrative access; manage security roles; configure AI Builder prompts

**Field-Level Security**:
- AI output fields (`cr4ea_ai_summary`, `cr4ea_ai_capa`) set to read-only for Quality Specialists to prevent manual editing (enforces human validation workflow)
- `cr4ea_confidence` field visible only to Quality Managers and above to prevent user bias based on AI confidence scores

### Data Privacy and Protection

**Data Residency**:
- Dataverse environment deployed in organization's primary geography (e.g., US, EU) to comply with data residency requirements
- AI Builder calls route to Azure OpenAI service in same geography (EU GDPR compliance: data processed within EU region)

**Sensitive Data Handling**:
- Deviation descriptions may contain regulated data (patient identifiers, proprietary manufacturing processes)
- AI Builder prompts configured to NOT log input/output data in Azure OpenAI telemetry (privacy mode enabled)
- Dataverse database encryption at rest (SQL Server Transparent Data Encryption) and in transit (TLS 1.2+)

**Data Retention and Deletion**:
- Dataverse records retained per organization's quality record retention policy (typically 7-10 years for GMP records)
- Deletion of deviation records requires Quality Director approval and generates audit log entry
- No data retention in AI Builder beyond request/response lifecycle (no fine-tuning or model training on customer data)

### Audit Trail and 21 CFR Part 11 Compliance

**System Audit Logging** (Dataverse built-in):
- **Create**: User ID, timestamp, initial field values
- **Update**: User ID, timestamp, old value, new value for each field changed
- **Delete**: User ID, timestamp, all field values at time of deletion (soft delete, recoverable)
- **Read**: Dataverse does not log read operations by default; enable user access auditing if required for 21 CFR Part 11 compliance

**Power Automate Audit Trail**:
- Flow run history retained for 28 days (default) with all action inputs/outputs
- For long-term retention: configure flow to write AI inputs/outputs to Dataverse audit table (custom implementation)
- Flow modification history tracked: who changed flow logic, when, and what changes made

**AI Model Versioning**:
- AI Builder prompt versions tagged with version number and change description
- Prompt changes require Quality Manager approval (implement via change control process, not system-enforced)
- Historical prompt versions retained to reconstruct classification logic used at any point in time

**Electronic Signature** (future enhancement):
- Dataverse supports electronic signature add-ons (e.g., Adobe Sign, DocuSign) for investigation approval workflow
- Quality Manager "signs" deviation closure by updating Status field, generating audit log entry with user ID and timestamp

### Compliance Validation

**GAMP 5 Classification**:
- Process Copilot classifies as **GAMP 5 Category 4 (Configured Product)** with **Category 5 (Custom AI Prompts)** components
- Validation approach: Combined approach of vendor assessment (Microsoft Power Platform) + user validation testing (AI prompt accuracy)

**Validation Deliverables**:
- **User Requirements Specification (URS)**: Business requirements for deviation management and AI classification
- **Functional Requirements Specification (FRS)**: Detailed system behavior including AI prompt logic and classification rules
- **Design Specification (DS)**: Architecture diagram, data model, flow logic (this document serves as DS foundation)
- **Installation Qualification (IQ)**: Verify Power Platform environment provisioning, licensing, security roles, and Dataverse table schema
- **Operational Qualification (OQ)**: Test all flow steps with sample deviations, verify AI prompt outputs, confirm email alerts
- **Performance Qualification (PQ)**: Validate AI classification accuracy using historical deviation dataset (minimum 100 deviations across all severity levels, target >90% agreement with human classification)
- **Traceability Matrix**: Map URS requirements to FRS, test cases, and validation evidence

**AI Model Validation Considerations**:
- AI Builder uses non-deterministic models; exact reproducibility not achievable
- Validation approach: Statistical accuracy assessment (precision, recall, F1 score for classification) rather than deterministic test scripts
- Define acceptable accuracy threshold: e.g., "AI classification must agree with human expert classification in >85% of cases"
- Ongoing monitoring: Track human override rate and retrain/re-prompt if accuracy degrades below threshold

### Disaster Recovery and Business Continuity

**Backup and Recovery**:
- Dataverse includes automatic backup (daily backups retained for 7 days, manual backups retained per configuration)
- Power Automate flows and AI Builder prompts backed up via solution export (ZIP file containing all metadata)
- Recovery Time Objective (RTO): <4 hours to restore Dataverse environment from backup
- Recovery Point Objective (RPO): <24 hours (data loss limited to deviations created since last backup)

**Degraded Mode Operation**:
- If AI Builder service unavailable: Flow fails gracefully, deviation record saved with null AI fields, email alert sent to system administrator
- Quality specialists manually classify deviations during AI outage using traditional process (no blocking failure)
- Once AI service restored, backlog of unclassified deviations can be reprocessed by triggering flow manually

**Service Level Expectations**:
- Microsoft Power Platform SLA: 99.9% uptime (monthly)
- Azure OpenAI Service SLA: 99.9% uptime (monthly)
- Combined SLA: 99.8% uptime (approximately 87 minutes downtime per month)
- Acceptable for back-office quality system (not patient-facing critical application)

---

## Conclusion

Process Copilot's architecture prioritizes simplicity, maintainability, and regulatory compliance through a fully configured (no-code) Microsoft Power Platform solution. The event-driven design ensures real-time AI classification and escalation while maintaining clear audit trails and human validation checkpoints required for GMP operations.

Key architectural strengths:
- **Serverless and scalable**: No infrastructure management; Power Platform handles scaling automatically
- **Separation of concerns**: Data, orchestration, AI, and presentation layers cleanly separated for independent evolution
- **Regulatory-ready**: Audit logging, security controls, and validation documentation pathways built into platform
- **Fast time-to-value**: 40-60 hours implementation vs. months for custom development

This architecture serves as a reference design for AI-augmented quality operations in pharmaceutical manufacturing, demonstrating how modern low-code platforms can deliver enterprise-grade solutions with minimal technical debt.
