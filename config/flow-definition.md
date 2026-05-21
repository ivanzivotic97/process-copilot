# Power Automate Flow Definition

## Flow Overview

**Flow Name**: Process Copilot - Deviation Classification and CAPA  
**Flow Type**: Automated cloud flow  
**Trigger**: Dataverse - When a row is added, modified or deleted  
**Purpose**: Automatically classify deviation severity, generate summary, suggest CAPA, and send high-severity email alerts

**Performance Characteristics**:
- Average runtime: 10-15 seconds
- AI Builder calls: 3 per deviation (Classify, Summarize, CAPA)
- Success rate target: >95%
- Retry logic: 3 attempts per AI Builder call with exponential backoff

---

## Trigger Configuration

### Step 1: When a row is added (Dataverse)

**Connector**: Microsoft Dataverse  
**Action**: When a row is added, modified or deleted  

**Parameters**:
```json
{
  "changeType": "Added",
  "tableName": "cr4ea_deviations",
  "scope": "Organization",
  "filterExpression": ""
}
```

**Explanation**: Flow triggers when any user creates a new deviation record in Dataverse. `changeType: "Added"` ensures flow runs only on new records (not updates or deletes). `scope: "Organization"` monitors all records regardless of owner.

**Output Variables** (dynamic content available in subsequent steps):
- `cr4ea_deviationsid`: Unique identifier (GUID) of the new record
- `cr4ea_title`: Deviation title
- `cr4ea_description`: Deviation description
- `createdon`: Record creation timestamp
- `ownerid`: User who created the record

---

## Variable Initialization

### Step 2: Initialize variable - Title

**Action**: Initialize variable  
**Variable Name**: `var_title`  
**Type**: String  
**Value**: `@{triggerOutputs()?['body/cr4ea_title']}`

### Step 3: Initialize variable - Description

**Action**: Initialize variable  
**Variable Name**: `var_description`  
**Type**: String  
**Value**: `@{triggerOutputs()?['body/cr4ea_description']}`

### Step 4: Initialize variable - Record ID

**Action**: Initialize variable  
**Variable Name**: `var_record_id`  
**Type**: String  
**Value**: `@{triggerOutputs()?['body/cr4ea_deviationsid']}`

**Explanation**: Variables store trigger outputs for reuse in AI Builder calls and Dataverse update. This improves flow readability and simplifies debugging.

---

## AI Builder Integration

### Step 5: Create a prompt (AI Builder) - Classify Severity

**Connector**: AI Builder  
**Action**: Create a prompt  

**Parameters**:
```json
{
  "promptName": "ClassifyDeviationSeverity",
  "inputs": {
    "title": "@{variables('var_title')}",
    "description": "@{variables('var_description')}"
  },
  "temperature": 0.3,
  "maxTokens": 500
}
```

**Expected Output** (JSON string):
```json
{
  "severity": "Medium",
  "justification": "OOS result without immediate patient risk per ICH Q10 guidelines...",
  "confidence": 87
}
```

**Error Handling**: Configure retry policy with 3 attempts, exponential interval (minimum 10 seconds, maximum 60 seconds). If all retries fail, flow should continue to Step 6 with null classification (manual review required).

---

### Step 6: Parse JSON - Classification Output

**Action**: Parse JSON (Data Operations)  

**Content**: `@{outputs('Create_a_prompt_-_Classify_Severity')?['body/text']}`

**Schema**:
```json
{
  "type": "object",
  "properties": {
    "severity": {
      "type": "string",
      "enum": ["Low", "Medium", "High"]
    },
    "justification": {
      "type": "string"
    },
    "confidence": {
      "type": "number",
      "minimum": 0,
      "maximum": 100
    }
  },
  "required": ["severity", "justification", "confidence"]
}
```

**Output Variables**:
- `body('Parse_JSON_-_Classification_Output')?['severity']`: Severity value (Low/Medium/High)
- `body('Parse_JSON_-_Classification_Output')?['justification']`: Justification text
- `body('Parse_JSON_-_Classification_Output')?['confidence']`: Confidence score (0-100)

**Error Handling**: If JSON parsing fails (malformed AI response), flow should send error notification to system administrator and halt. Do not update Dataverse record with invalid data.

---

### Step 7: Create a prompt (AI Builder) - Summarize Deviation

**Connector**: AI Builder  
**Action**: Create a prompt  

**Parameters**:
```json
{
  "promptName": "SummarizeDeviation",
  "inputs": {
    "title": "@{variables('var_title')}",
    "description": "@{variables('var_description')}"
  },
  "temperature": 0.5,
  "maxTokens": 800
}
```

**Expected Output** (JSON string):
```json
{
  "what_happened": "Cold storage Room 5 experienced a temperature excursion reaching 10.2°C...",
  "potential_impact": "Potential degradation or loss of potency for temperature-sensitive API materials...",
  "immediate_containment": "Maintain quarantine status on all 12 affected API batches...",
  "additional_info_needed": "yes"
}
```

---

### Step 8: Parse JSON - Summary Output

**Action**: Parse JSON (Data Operations)  

**Content**: `@{outputs('Create_a_prompt_-_Summarize_Deviation')?['body/text']}`

**Schema**:
```json
{
  "type": "object",
  "properties": {
    "what_happened": {
      "type": "string"
    },
    "potential_impact": {
      "type": "string"
    },
    "immediate_containment": {
      "type": "string"
    },
    "additional_info_needed": {
      "type": "string",
      "enum": ["yes", "no"]
    }
  },
  "required": ["what_happened", "potential_impact", "immediate_containment", "additional_info_needed"]
}
```

**Output Variables**:
- `body('Parse_JSON_-_Summary_Output')?['what_happened']`
- `body('Parse_JSON_-_Summary_Output')?['potential_impact']`
- `body('Parse_JSON_-_Summary_Output')?['immediate_containment']`
- `body('Parse_JSON_-_Summary_Output')?['additional_info_needed']`

---

### Step 9: Create a prompt (AI Builder) - Suggest CAPA

**Connector**: AI Builder  
**Action**: Create a prompt  

**Parameters**:
```json
{
  "promptName": "SuggestCAPA",
  "inputs": {
    "title": "@{variables('var_title')}",
    "description": "@{variables('var_description')}",
    "severity": "@{body('Parse_JSON_-_Classification_Output')?['severity']}"
  },
  "temperature": 0.6,
  "maxTokens": 1500
}
```

**Expected Output** (JSON string):
```json
{
  "root_cause_category": "Equipment Failure",
  "immediate_corrections": [
    "Within 24 hours: Re-test dissolution using reserve sample units...",
    "Within 48 hours: Review all manufacturing batch record data..."
  ],
  "corrective_actions": [
    "By Day 5: Conduct full laboratory investigation...",
    "By Day 10: If root cause identified as process-related..."
  ],
  "preventive_actions": [
    "By Day 21: Revise dissolution method to include additional testing timepoints...",
    "By Day 30: Implement enhanced in-process controls..."
  ],
  "effectiveness_check": "Method: Monitor dissolution test results for next 10 production batches...",
  "assumptions": [
    "Assumes laboratory error has been ruled out",
    "Assumes process parameters were within validated ranges"
  ]
}
```

---

### Step 10: Parse JSON - CAPA Output

**Action**: Parse JSON (Data Operations)  

**Content**: `@{outputs('Create_a_prompt_-_Suggest_CAPA')?['body/text']}`

**Schema**:
```json
{
  "type": "object",
  "properties": {
    "root_cause_category": {
      "type": "string",
      "enum": ["Human Error", "Equipment Failure", "Process Design", "Material/Supplier", "Environmental", "Documentation", "Training Gap", "System/Software"]
    },
    "immediate_corrections": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "corrective_actions": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "preventive_actions": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "effectiveness_check": {
      "type": "string"
    },
    "assumptions": {
      "type": "array",
      "items": {
        "type": "string"
      }
    }
  },
  "required": ["root_cause_category", "immediate_corrections", "corrective_actions", "preventive_actions", "effectiveness_check", "assumptions"]
}
```

**Output Variables**:
- `body('Parse_JSON_-_CAPA_Output')?['root_cause_category']`
- `body('Parse_JSON_-_CAPA_Output')?['immediate_corrections']` (array)
- `body('Parse_JSON_-_CAPA_Output')?['corrective_actions']` (array)
- `body('Parse_JSON_-_CAPA_Output')?['preventive_actions']` (array)
- `body('Parse_JSON_-_CAPA_Output')?['effectiveness_check']`
- `body('Parse_JSON_-_CAPA_Output')?['assumptions']` (array)

---

## Data Transformation

### Step 11: Compose - Format AI Summary

**Action**: Compose (Data Operations)  

**Inputs**: 
```
What Happened:
@{body('Parse_JSON_-_Summary_Output')?['what_happened']}

Potential Impact:
@{body('Parse_JSON_-_Summary_Output')?['potential_impact']}

Immediate Containment:
@{body('Parse_JSON_-_Summary_Output')?['immediate_containment']}

Additional Information Needed: @{body('Parse_JSON_-_Summary_Output')?['additional_info_needed']}
```

**Output Variable**: `outputs('Compose_-_Format_AI_Summary')`

**Explanation**: Concatenates JSON fields into human-readable multi-line text suitable for Dataverse `cr4ea_ai_summary` field storage and Power Apps display.

---

### Step 12: Compose - Format AI CAPA

**Action**: Compose (Data Operations)  

**Inputs**: 
```
Root Cause Category: @{body('Parse_JSON_-_CAPA_Output')?['root_cause_category']}

Immediate Corrections (24-48 hours):
@{join(body('Parse_JSON_-_CAPA_Output')?['immediate_corrections'], '
- ')}

Corrective Actions:
@{join(body('Parse_JSON_-_CAPA_Output')?['corrective_actions'], '
- ')}

Preventive Actions:
@{join(body('Parse_JSON_-_CAPA_Output')?['preventive_actions'], '
- ')}

Effectiveness Check:
@{body('Parse_JSON_-_CAPA_Output')?['effectiveness_check']}

Assumptions:
@{join(body('Parse_JSON_-_CAPA_Output')?['assumptions'], '
- ')}
```

**Output Variable**: `outputs('Compose_-_Format_AI_CAPA')`

**Expression Notes**:
- `join(array, delimiter)`: Converts JSON array to multi-line string with newline + bullet separator
- Example: `["Action 1", "Action 2"]` becomes `"- Action 1\n- Action 2"`

---

## Dataverse Update

### Step 13: Update a row (Dataverse)

**Connector**: Microsoft Dataverse  
**Action**: Update a row  

**Parameters**:
```json
{
  "tableName": "cr4ea_deviations",
  "rowId": "@{variables('var_record_id')}",
  "item/cr4ea_severity": "@{body('Parse_JSON_-_Classification_Output')?['severity']}",
  "item/cr4ea_ai_summary": "@{outputs('Compose_-_Format_AI_Summary')}",
  "item/cr4ea_ai_capa": "@{outputs('Compose_-_Format_AI_CAPA')}",
  "item/cr4ea_confidence": "@{body('Parse_JSON_-_Classification_Output')?['confidence']}",
  "item/cr4ea_classified_date": "@{utcNow()}"
}
```

**Severity Mapping** (text to choice field value):

If using Dataverse choice field with numeric values instead of text, add a `Condition` action before the update to map text to choice values:

```
Switch: @{body('Parse_JSON_-_Classification_Output')?['severity']}

Case "Low": Set variable 'var_severity_value' to 919860000
Case "Medium": Set variable 'var_severity_value' to 919860001
Case "High": Set variable 'var_severity_value' to 919860002
Default: Set variable 'var_severity_value' to 919860001 (Medium as safe default)
```

Then use `@{variables('var_severity_value')}` in the Update a row action for `item/cr4ea_severity`.

**Explanation**: Updates the original deviation record with all AI outputs. `utcNow()` returns ISO 8601 timestamp (e.g., `2024-05-21T10:30:45Z`). Dataverse audit log automatically records this update with user ID and timestamp.

---

## Conditional Escalation

### Step 14: Condition - Check if Severity = High

**Action**: Condition (Control)  

**Condition**:
```json
{
  "and": [
    {
      "equals": [
        "@{body('Parse_JSON_-_Classification_Output')?['severity']}",
        "High"
      ]
    }
  ]
}
```

**If Yes** (proceed to Step 15):

---

### Step 15: Send an email (V2) - High Severity Alert

**Connector**: Office 365 Outlook  
**Action**: Send an email (V2)  

**Parameters**:
```json
{
  "to": "quality.leadership@yourcompany.com",
  "subject": "HIGH SEVERITY DEVIATION - @{variables('var_title')}",
  "body": "<HTML email body - see below>",
  "importance": "High"
}
```

**Email Body (HTML)**:
```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-size: 14px; color: #333; }
    .header { background-color: #D13438; color: white; padding: 15px; font-size: 18px; font-weight: bold; }
    .section { margin: 20px 0; }
    .label { font-weight: bold; color: #505050; }
    .value { margin-left: 10px; }
    .footer { margin-top: 30px; font-size: 12px; color: #888; border-top: 1px solid #ddd; padding-top: 10px; }
    .button { background-color: #0078D4; color: white; padding: 10px 20px; text-decoration: none; display: inline-block; margin-top: 15px; border-radius: 3px; }
  </style>
</head>
<body>
  <div class="header">
    🚨 HIGH SEVERITY DEVIATION ALERT
  </div>
  
  <div class="section">
    <p>A high-severity manufacturing deviation has been logged and requires immediate attention.</p>
  </div>
  
  <div class="section">
    <span class="label">Deviation ID:</span>
    <span class="value">@{variables('var_record_id')}</span>
  </div>
  
  <div class="section">
    <span class="label">Title:</span>
    <span class="value">@{variables('var_title')}</span>
  </div>
  
  <div class="section">
    <span class="label">Severity Classification:</span>
    <span class="value" style="color: #D13438; font-weight: bold;">HIGH</span>
  </div>
  
  <div class="section">
    <span class="label">AI Confidence:</span>
    <span class="value">@{body('Parse_JSON_-_Classification_Output')?['confidence']}%</span>
  </div>
  
  <div class="section">
    <span class="label">Justification:</span>
    <div class="value">@{body('Parse_JSON_-_Classification_Output')?['justification']}</div>
  </div>
  
  <div class="section">
    <span class="label">What Happened:</span>
    <div class="value">@{body('Parse_JSON_-_Summary_Output')?['what_happened']}</div>
  </div>
  
  <div class="section">
    <span class="label">Potential Impact:</span>
    <div class="value">@{body('Parse_JSON_-_Summary_Output')?['potential_impact']}</div>
  </div>
  
  <div class="section">
    <span class="label">Immediate Containment Required:</span>
    <div class="value">@{body('Parse_JSON_-_Summary_Output')?['immediate_containment']}</div>
  </div>
  
  <div class="section">
    <a href="https://make.powerapps.com/environments/{environment_id}/apps/{app_id}?recordId=@{variables('var_record_id')}" class="button">
      View Deviation in Process Copilot
    </a>
  </div>
  
  <div class="footer">
    This alert was generated automatically by Process Copilot AI-powered deviation management system.<br>
    Classification timestamp: @{utcNow()}<br>
    For questions, contact Quality Assurance at quality@yourcompany.com
  </div>
</body>
</html>
```

**Customization Notes**:
- Replace `quality.leadership@yourcompany.com` with actual distribution list
- Replace `{environment_id}` and `{app_id}` placeholders with actual Power Apps environment GUID and app GUID (obtain from Power Apps maker portal URL)
- Deep link format: `https://make.powerapps.com/environments/{env}/apps/{app}?recordId={record_id}` opens specific deviation record in Power Apps

**If No** (severity is Low or Medium):
- Skip email action
- Flow proceeds directly to completion

---

## Flow Completion and Monitoring

### Step 16: Flow Termination

**Action**: Terminate (Control) - Success  

**Status**: Succeeded  
**Code**: 200  
**Message**: `Deviation @{variables('var_record_id')} processed successfully. Severity: @{body('Parse_JSON_-_Classification_Output')?['severity']}. Classification confidence: @{body('Parse_JSON_-_Classification_Output')?['confidence']}%.`

**Explanation**: Explicit termination with success status provides clear run history log. Message includes key outputs for debugging.

---

## Error Handling Strategy

### Global Error Handling (Configure on Flow Settings)

**Configure run after settings** on critical steps to handle failures gracefully:

1. **AI Builder prompts (Steps 5, 7, 9)**: 
   - Configure retry policy: 3 attempts, exponential interval (10-60 seconds)
   - If all retries fail: Add parallel branch → Send email to system administrator with error details → Terminate with "Failed" status

2. **Parse JSON actions (Steps 6, 8, 10)**:
   - Configure "run after" to execute only if previous step succeeded
   - If parsing fails (malformed JSON): Add error notification → Terminate with "Failed" status
   - Do not update Dataverse with incomplete data if parsing fails

3. **Dataverse Update (Step 13)**:
   - If update fails due to permission or connection issue: Add retry (2 attempts, 5-second interval)
   - If all retries fail: Send email to system administrator → Terminate with "Failed" status
   - Do not send high-severity email if Dataverse update failed (avoid false escalation)

### Specific Error Scenarios

**Scenario 1: AI Builder returns non-JSON response**

Example: Model returns `"I cannot classify this deviation due to insufficient information."`

**Mitigation**:
- Parse JSON action will fail
- Flow sends error notification: "AI Builder returned invalid JSON for deviation [ID]. Manual classification required."
- Quality Manager receives notification and manually classifies deviation

**Scenario 2: AI Builder service unavailable (HTTP 503)**

**Mitigation**:
- Retry policy (3 attempts, exponential backoff) handles transient outages
- If service remains unavailable after 3 retries: Flow sends alert, deviation remains unclassified
- Quality specialists manually process backlog once service restored
- Flow can be manually re-triggered for backlog deviations via "Run flow" button in Power Apps

**Scenario 3: Network connectivity interruption between flow steps**

**Mitigation**:
- Power Automate built-in retry logic handles network transients automatically
- Flow run history shows retry attempts and eventual success/failure
- If flow fails mid-execution: Dataverse record remains in original state (null AI fields), no partial updates

---

## Performance Optimization

### Recommendations for High-Volume Environments

**Current Design**: Sequential execution (Classify → Summarize → CAPA → Update)  
**Total latency**: 10-15 seconds for 3 AI calls + parsing + Dataverse update

**Optimization 1: Parallel AI Calls** (reduces latency by 40-50%)

Add `Scope` actions to parallelize AI Builder prompts:
- Scope 1: Classify Severity + Parse JSON
- Scope 2: Summarize Deviation + Parse JSON
- Scope 3: Wait for Scope 1 completion → Suggest CAPA (requires severity input) + Parse JSON

Parallel execution reduces total latency to 6-8 seconds.

**Optimization 2: Batch Processing for Backlog**

If processing backlog of existing deviations (e.g., 100+ records at system go-live):
- Create child flow that accepts Deviation ID as input parameter
- Parent flow: List rows from Dataverse (filter: unclassified deviations) → Apply to each → Run child flow
- Throttling: Add 5-second delay between child flow invocations to avoid AI Builder rate limits (60 requests/minute)

**Optimization 3: AI Builder Credit Management**

Monitor AI Builder credit consumption:
- Classification: ~0.5 credits per call (500 tokens)
- Summarization: ~0.8 credits per call (800 tokens)
- CAPA: ~1.5 credits per call (1500 tokens)
- Total: ~2.8 credits per deviation

For 200 deviations/month: 560 credits/month (~$5.60 at $10/1000 credits)

**Note**: AI Builder pricing is subject to change; verify current rates at https://powerapps.microsoft.com/ai-builder/pricing/

---

## Testing and Validation

### Unit Testing (per step)

1. **Test Dataverse trigger**: Create sample deviation record, verify flow initiates
2. **Test AI classification**: Review classification output for accuracy against known deviation
3. **Test AI summarization**: Verify summary captures key facts from description
4. **Test AI CAPA**: Assess CAPA quality and completeness
5. **Test severity mapping**: Verify Low/Medium/High maps correctly to choice field values
6. **Test Dataverse update**: Confirm all AI outputs written to correct fields
7. **Test email alert**: Create High severity deviation, verify email received within 60 seconds

### Integration Testing (end-to-end)

**Test Case 1: High Severity Deviation with Complete Information**

Input:
- Title: "Sterility breach in aseptic filling"
- Description: "Grade A cleanroom pressure dropped to -2 Pa (spec: +15 Pa) for 12 minutes during filling of Batch #2024-0500. 200 vials produced during excursion. No vials released."

Expected Output:
- Severity: High
- AI Summary: Comprehensive summary with patient safety impact
- AI CAPA: Equipment Failure root cause, detailed immediate corrections
- Email Alert: Sent to quality.leadership@yourcompany.com within 60 seconds
- Dataverse Record: All fields populated correctly

**Test Case 2: Low Severity Deviation with Minimal Information**

Input:
- Title: "Typo in batch record"
- Description: "Operator wrote '500mL' instead of '500mg' in material description section of batch record. Correct unit confirmed via material label and lot number."

Expected Output:
- Severity: Low
- AI Summary: Brief summary with no quality impact noted
- AI CAPA: Documentation root cause, simple corrective actions (retroactive correction)
- Email Alert: Not sent (severity is Low)
- Dataverse Record: All fields populated correctly

**Test Case 3: Ambiguous Severity Deviation**

Input:
- Title: "Unusual odor in production area"
- Description: "Operator reported unusual chemical odor in tablet compression area during production. No product defects observed. Odor dissipated after 10 minutes. Root cause unknown."

Expected Output:
- Severity: Medium (conservative classification due to ambiguity)
- Confidence: <75% (flags for manual review)
- AI Summary: Acknowledges missing information
- AI CAPA: Environmental root cause (preliminary), recommends expanded investigation
- Email Alert: Not sent (severity is Medium, not High)

### Performance Qualification (PQ) Testing

**Objective**: Validate AI classification accuracy using historical deviation dataset

**Methodology**:
1. Select 100 historical deviations (30 Low, 50 Medium, 20 High) with confirmed severity assignments by human experts
2. Create test Dataverse records with historical titles and descriptions
3. Trigger flow execution for all 100 records
4. Compare AI classifications to human classifications
5. Calculate accuracy metrics:
   - Overall accuracy: (correct classifications / total) × 100%
   - Precision per severity level: (true positives / (true positives + false positives))
   - Recall per severity level: (true positives / (true positives + false negatives))
   - F1 score: harmonic mean of precision and recall

**Acceptance Criteria**:
- Overall accuracy: >85%
- High severity recall: >95% (minimize false negatives for patient safety)
- Low false positive rate for High severity: <10% (minimize alert fatigue)

**Remediation if criteria not met**:
- Review misclassified deviations for patterns
- Tune AI Builder prompts (add examples, clarify criteria, adjust temperature)
- Retest with additional 50-deviation sample
- Document prompt changes in validation file

---

## Flow Maintenance

### Version Control

All flow changes should follow change control process:
1. Export flow as solution ZIP file before making changes
2. Document change rationale in flow description field (visible in Power Automate)
3. Test changes in sandbox environment before deploying to production
4. Archive previous flow version (rename to "v1.0 - Archived" before publishing v2.0)

### Monitoring and Alerting

Configure flow analytics dashboard to track:
- **Success rate**: Target >95%
- **Average runtime**: Target <20 seconds
- **AI Builder call latency**: Track per-prompt performance
- **Email delivery rate**: Verify all High severity alerts delivered successfully
- **Human override rate**: Track how often quality specialists change AI classification (target: 10-15% indicating thoughtful validation)

### Continuous Improvement

Quarterly review:
- Analyze deviations with low confidence scores (<70%) to identify prompt tuning opportunities
- Review human overrides to understand classification disagreements
- Update prompt examples and criteria based on regulatory guideline changes
- Monitor AI Builder model updates from Microsoft and revalidate if needed

---

## Appendix: Common Expressions

### Useful Power Automate Expressions

**Current timestamp (ISO 8601)**:
```
utcNow()
```
Example output: `2024-05-21T10:30:45Z`

**Format timestamp for display**:
```
formatDateTime(utcNow(), 'yyyy-MM-dd HH:mm:ss')
```
Example output: `2024-05-21 10:30:45`

**Convert JSON array to multi-line string**:
```
join(body('Parse_JSON_Output')?['array_field'], '\n- ')
```
Example: `["Item 1", "Item 2"]` → `"- Item 1\n- Item 2"`

**Conditional value (if-then-else)**:
```
if(equals(variables('var_severity'), 'High'), 'URGENT', 'Standard')
```

**Null coalescing (default value if null)**:
```
coalesce(body('Parse_JSON_Output')?['optional_field'], 'No data provided')
```

**String concatenation**:
```
concat('Deviation ', variables('var_record_id'), ' classified as ', variables('var_severity'))
```

**Extract GUID from Dataverse lookup field**:
```
triggerOutputs()?['body/_ownerid_value']
```

### Dataverse Choice Field Value Mapping

If using choice fields with numeric values (instead of text labels):

| Severity | Display Label | Numeric Value |
|----------|---------------|---------------|
| Low | Low | 919860000 |
| Medium | Medium | 919860001 |
| High | High | 919860002 |

**Expression to map text to numeric value**:
```
switch(
  body('Parse_JSON_-_Classification_Output')?['severity'],
  'Low', 919860000,
  'Medium', 919860001,
  'High', 919860002,
  919860001
)
```
Last parameter (919860001) is default value if no match found (Medium as safe default).

---

## Document Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-05-21 | [Your Name] | Initial flow definition for Process Copilot v1.0 |

---

**For questions or support, contact**: [Your Professional Contact]
