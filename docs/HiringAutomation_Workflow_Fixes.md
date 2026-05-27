# Hiring Automation Workflow Walkthrough

This document explains the full hiring automation system in `hiring-automation-workflows.json`. The export contains two logical workflows on one n8n canvas:

1. **Workflow 1 - Screening Pipeline**
2. **Workflow 2 - Communication Pipeline**

The system supports two roles:

- `SWE` - Software Engineer
- `BDM` - Business Development Manager

The core idea is simple: Workflow 1 reads new candidates, standardizes the data, screens resumes with AI, and updates the master sheet. Workflow 2 runs separately, reads screened candidates, sends the correct communication, creates calendar events for strong candidates, and updates the master sheet so nobody is contacted twice.

## Assignment Requirements Covered

- Pull applications from two separate Google Form response sheets.
- Handle different columns for SWE and BDM.
- Normalize messy form data into one master schema.
- Download resume files from Google Drive URLs.
- Extract PDF text.
- Use AI screening with role-specific evaluation criteria.
- Write score, classification, summary, strengths, concerns, and recommendation back to the master sheet.
- Run communication separately from screening.
- For strong candidates: create a Google Calendar event and contact the candidate.
- For average candidates: notify the hiring manager for manual review.
- For weak candidates: send a polite rejection email.
- Process multiple candidates per run.
- Avoid duplicate screening and duplicate communication.
- Exit gracefully when no new candidates exist.

## Files

- `hiring-automation-workflows.json`: final n8n workflow export.
- `HiringAutomation_Workflow_Fixes.md`: this explanation document.
- Test resumes: `CV_SWE_Average.pdf`, `CV_SWE_Weak.pdf`, `CV_BDM_Average.pdf`, `CV_BDM_Weak.pdf`, plus sample strong resumes.

## Master Sheet Columns

The master sheet is the central database for the automation. Current columns:

`Candidate ID`, `name`, `email`, `role`, `experience`, `status`, `ai_score`, `ai_classification`, `ai_summary`, `ai_strengths`, `ai_concerns`, `interview_recommended`, `contacted`, `calendar_event_link`, `extra_link`, `resume_url`, `drive_file_id`, `workflow_status`

Optional columns that are useful for debugging but not required:

- `source_sheet`
- `source_row_number`
- `error_note`
- `ingested_at`

These optional values are used internally by the workflow item data. If the columns exist in the sheet, they can help you debug; if they do not exist, the main workflow still works as long as the required columns above are present.

## Source Sheet Columns

### SWE Sheet

Expected columns:

- `Timestamp`
- `Email address`
- `Full Name`
- `Email Address`
- `Resume / CV (PDF)`
- `GitHub Profile Link`
- `Programming Languages`
- `processed`

### BDM Sheet

Expected columns:

- `Timestamp`
- `Email address`
- `Name`
- `Contact Email`
- `Resume`
- `LinkedIn Profile`
- `Years of Sales Experience`
- `processed`

## Workflow 1 - Screening Pipeline

### High-Level Flow

```text
Schedule Trigger
├── Master_Sheet
├── get SWE Applications -> ROLE-SWE
└── Get BDM Applications -> ROLE-BDM

ROLE-SWE + ROLE-BDM -> Merge
Merge + Master_Sheet -> Code
Code -> If
If true -> Append or update row in sheet
-> Download file
-> Extract from File
-> Basic LLM Chain
-> Code1
-> Updation in Master Sheet
-> Switch Source Sheet
-> Mark SWE Source Processed / Mark BDM Source Processed
```

### Why Master Sheet Is Read at the Start

The workflow reads `Master_Sheet` at the beginning so the `Code` node can check whether a candidate already exists. This prevents duplicates. The schedule trigger fans out to the master sheet and both source sheets. The `Master_Sheet` node also connects directly to `Code`, so the code can access existing master rows using:

```python
master_rows = _('Master_Sheet').all()
```

Recommended n8n setting:

```text
Master_Sheet -> Settings -> Always Output Data = ON
```

This helps first-time runs where the master sheet may be empty.

### Role Tagging Nodes

`ROLE-SWE` adds:

```text
role = SWE
source_sheet = SWE
```

`ROLE-BDM` adds:

```text
role = BDM
source_sheet = BDM
```

This is important because the two source sheets have different column names. The downstream `Code` node uses `role` to decide how to normalize each row.

### Merge Node

The `Merge` node combines the SWE and BDM streams so one normalization process can handle both roles.

## Code Node - Data Cleaning, Deduplication, and Normalization

The `Code` node is the most important data safety step in Workflow 1.

### Helper Functions

`clean(value)`:

- Converts `None`, `nan`, `none`, `-`, `n/a`, and `null` into empty string.
- Trims whitespace.
- Prevents bad placeholder values from entering the master sheet.

`truthy(value)`:

- Treats `true`, `yes`, `1`, `done`, and `processed` as true.
- Used for the source sheet `processed` column.

`find_field(data, exact_keys, keywords)`:

- First tries exact column names.
- If exact column names are not found, falls back to keyword matching.
- This makes the workflow resilient to small column-name changes.

`extract_drive_file_id(url)`:

Supports common Google Drive formats:

- `https://drive.google.com/open?id=FILE_ID`
- `https://drive.google.com/file/d/FILE_ID/view`
- URLs containing `/d/FILE_ID`
- Raw Drive file IDs

`candidate_id(email, role)`:

- Creates a deterministic ID from `email + role`.
- Format example: `SWE-092C4698F58C`
- Uses SHA-256 and the first 12 uppercase characters.

Why deterministic ID matters:

- Same candidate applying to the same role always gets the same ID.
- Google Sheets updates can safely match on `Candidate ID`.
- Re-running the workflow does not create accidental duplicates.

### Master Sheet Duplicate Map

The code builds:

```python
existing_by_key[f'{email}|{role}'] = row
existing_ids.add(candidate_id)
```

This means the workflow checks duplicates by both:

- candidate email + role
- Candidate ID

This is better than email alone because the same person could apply for both SWE and BDM.

### Retry Statuses

Retryable statuses:

```python
RETRY_STATUSES = {'pending', 'screening_error', 'resume_error'}
```

If a candidate exists in the master sheet with one of those statuses, Workflow 1 may retry screening. If the candidate is already `screened`, `interview_scheduled`, `manual_review_notified`, or `rejected`, the row is skipped.

### SWE Field Mapping

For SWE:

- `name`: `Full Name`
- `email`: `Email Address` or `Email address`
- `resume_url`: `Resume / CV (PDF)` or `Resume`
- `experience`: `Programming Languages`
- `extra_link`: `GitHub Profile Link`

### BDM Field Mapping

For BDM:

- `name`: `Name`
- `email`: `Contact Email`, `Email address`, or `Email Address`
- `resume_url`: `Resume`
- `experience`: `Years of Sales Experience`
- `extra_link`: `LinkedIn Profile`

### Required Fields

The candidate is skipped if any of these are missing:

- name
- email
- resume URL
- valid Drive file ID

Skipped rows are collected in `skipped_rows`, so the workflow exits cleanly instead of failing.

### Candidate Output to Master Sheet

For every valid new or retryable candidate, the code outputs a clean master row:

```text
Candidate ID
name
email
role
experience
status = pending
ai_score = empty
ai_classification = empty
ai_summary = empty
ai_strengths = empty
ai_concerns = empty
interview_recommended = empty
contacted = NO
calendar_event_link = empty
extra_link
resume_url
drive_file_id
workflow_status = NEW
source_sheet
source_row_number
error_note
ingested_at
```

If no valid candidates are found, the code returns:

```text
workflow_status = NO_NEW_CANDIDATES
message = No valid new candidates...
skipped_rows = first skipped rows
```

## If Node

The `If` node only allows rows where:

```text
workflow_status = NEW
```

If no new candidates exist, it routes to `Code2`, which logs a graceful stop message.

## Append or Update Master Sheet

This node writes the normalized candidate record to the master sheet using:

```text
matching column = Candidate ID
```

It uses `appendOrUpdate`, so:

- New candidates are inserted.
- Retryable candidates can be updated without creating duplicate rows.

## Resume Download and Extraction

`Download file`:

- Uses `drive_file_id`.
- Downloads the resume from Google Drive.

`Extract from File`:

- Extracts text from PDF.
- Keeps source data and text so the AI node still has candidate metadata.

Important limitation:

- If a PDF is scanned/image-only, normal text extraction may fail. OCR would be needed for fully production-grade handling.

## Basic LLM Chain - AI Screening

The LLM prompt is strict and role-specific.

Main rules:

- Evaluate only resume text and metadata.
- Do not invent facts.
- Do not browse.
- Do not include reasoning, markdown, code fences, or `<think>` tags.
- Return raw JSON only.
- Use consistent scoring.

### Score Bands

```text
75-100 = strong
50-74 = average
0-49 = weak
```

### SWE Rubric

SWE candidates are evaluated on:

- production engineering experience
- languages and frameworks
- APIs, databases, backend/frontend work
- system design, scale, architecture, ownership
- cloud/devops/infrastructure
- testing, debugging, reliability, performance
- projects/GitHub/portfolio
- measurable impact and leadership

### BDM Rubric

BDM candidates are evaluated on:

- sales experience
- quota/revenue ownership
- target achievement
- lead generation
- pipeline management
- client relationships
- negotiation and communication
- CRM usage
- proposal writing, account growth, industry fit

### Why the Prompt Uses Calibration Examples

The prompt includes examples like:

- 2-3 years Django/API without large-scale design should be average.
- Basic Python/web without production impact should be weak.
- 7+ years distributed systems and cloud architecture should be strong.
- Clear BDM revenue targets and CRM usage can be strong.

This reduces random scoring changes between runs.

## Code1 - AI Result Parser and Master Sheet Preparation

`Code1` is the second major safety layer. It exists because LLM output can vary even with a strict prompt.

It handles:

- Structured parser success: `output.score`
- Fields returned directly: `score`, `classification`, `summary`
- Raw text output
- Escaped JSON strings like `{\n "score": 65 }`
- Markdown code fences
- `<think>...</think>` reasoning blocks

### Important Functions in Code1

`clean_raw_text(text)`:

- Removes `<think>` blocks.
- Removes markdown code fences.
- Unquotes JSON strings if the model returns JSON as a string.
- Converts escaped `\n` and `\"` into normal JSON text.

`parse_ai_from_text(text)`:

- Tries direct `json.loads`.
- If direct parsing fails, searches for the first `{ ... }` JSON object inside the text.

`extract_ai_data(llm_json)`:

Checks output in this order:

1. `output` object from structured parser.
2. Direct fields from the LLM node.
3. Raw text/escaped JSON from `text`, `output`, `response`, or `result`.

### Classification Is Forced to Match Score

Even if the model says the wrong classification, `Code1` fixes it:

```text
score >= 75 -> strong
score 50-74 -> average
score < 50 -> weak
```

### Interview Recommendation Is Forced to Match Classification

```text
strong -> YES
average -> NO
weak -> NO
```

This prevents inconsistent data such as:

```text
classification = average
interview_recommended = YES
```

### Successful AI Output

If AI output is valid, `Code1` sets:

```text
ai_score
ai_classification
ai_summary
ai_strengths
ai_concerns
interview_recommended
status = screened
workflow_status = SCREENED
error_note = empty
```

### Failed AI Output

If no valid score/classification/summary is found, `Code1` sets:

```text
status = screening_error
workflow_status = ERROR
interview_recommended = NO
error_note = Missing or invalid LLM output object
```

This is important because Workflow 2 only communicates with rows where:

```text
status = screened
contacted = NO
```

So a broken AI response will not accidentally email a candidate.

## Updation in Master Sheet

This node updates the master sheet after AI screening.

Match key:

```text
Candidate ID
```

It writes the AI score, classification, summary, strengths, concerns, recommendation, status, and workflow status.

## Switch Source Sheet and Source Processed Updates

After successful master update, `Switch Source Sheet` routes by:

```text
source_sheet = SWE
source_sheet = BDM
```

Then:

- `Mark SWE Source Processed` updates the SWE response sheet.
- `Mark BDM Source Processed` updates the BDM response sheet.

Both set:

```text
processed = TRUE
```

This should happen only after screening succeeds, so source rows are not marked processed too early.

## Workflow 2 - Communication Pipeline

### High-Level Flow

```text
Schedule Trigger1
-> Master-Sheet
-> If1
-> Loop Over Items
-> Switch

strong:
Switch output 0
-> Create an event
-> Build Interview Email
-> Send Interview Email
-> Update row in sheet
-> Loop Over Items

weak:
Switch output 1
-> Send Rejection Email
-> Update contacted for avg and weak
-> Loop Over Items

average:
Switch output 2
-> Send Average Email
-> Update contacted for avg and weak
-> Loop Over Items
```

Important:

- `Loop Over Items` must connect to `Switch` from the `loop` output, not the `done` output.
- The final update nodes connect back to the `Loop Over Items` input so multiple candidates can be processed in one run.
- If you test manually, run the full workflow, not only the loop node. The loop node needs workflow context.

## If1 - Communication Eligibility

`If1` only allows candidates where:

```text
status = screened
contacted = NO
```

This prevents duplicate communication.

If a row is already:

```text
interview_scheduled
manual_review_notified
rejected
```

it will go to the false branch and will not be contacted again.

## Switch - Classification Routing

The switch normalizes classification:

```js
String($json['ai_classification'] || '').trim().toLowerCase()
```

Routes:

- `strong` -> calendar event path
- `weak` -> rejection email path
- `average` -> hiring manager review email path

Make sure the switch outputs are wired correctly:

```text
0 = strong -> Create an event
1 = weak -> Send Rejection Email
2 = average -> Send Average Email
```

## Strong Candidate Path

### Create an event

Creates a Google Calendar event and adds the candidate as attendee.

The assignment requires:

- strong candidates receive interview invitation
- Google Calendar event is created
- candidate is added as attendee

Because the candidate is an attendee, Google Calendar may send an automatic calendar invite email.

### Build Interview Email

This Code node builds the interview email body and extracts the internal calendar event link.

Important variables:

```js
const event = $input.first().json;
const candidate = $('Loop Over Items').item.json;
const calendarEventLink = event.htmlLink || event.hangoutLink || event.conferenceData?.entryPoints?.[0]?.uri || '';
```

It returns:

```text
output.subject
output.email_body
output.calendar_event_link
output.candidate_email
```

Current email behavior:

- The email tells the candidate that the calendar invitation has been sent.
- It does not directly include the calendar/Meet link.
- The calendar link is saved in the master sheet for internal tracking.

Note:

If you want to avoid duplicate candidate emails, remember that Google Calendar already sends an invite when the candidate is added as attendee. If you keep the Gmail interview email too, the candidate may receive both a calendar invite and a custom interview email.

### Update row in sheet

After the strong candidate path completes, this node updates:

```text
contacted = YES
status = interview_scheduled
calendar_event_link = output.calendar_event_link
```

Match key:

```text
Candidate ID
```

## Average Candidate Path

Average candidates do not get a candidate email directly. They go to hiring manager review.

`Send Average Email` sends a review email to:

```text
afnanshoukat35@gmial.com
```

Note: This is written exactly as configured. If it is a typo, change it to:

```text
afnanshoukat35@gmail.com
```

The email includes:

- candidate name
- role
- email
- AI score
- AI classification
- AI summary
- strengths
- concerns

Then `Update contacted for avg and weak` updates:

```text
contacted = YES
status = manual_review_notified
```

## Weak Candidate Path

Weak candidates receive a direct rejection email.

`Send Rejection Email` sends to:

```js
{{ $json["email"] }}
```

Then `Update contacted for avg and weak` updates:

```text
contacted = YES
status = rejected
```

## Status Lifecycle

The master sheet status moves through these stages:

```text
pending
screened
interview_scheduled
manual_review_notified
rejected
screening_error
```

Meaning:

- `pending`: candidate normalized and inserted, but AI screening not complete.
- `screened`: AI screening completed and candidate is ready for communication workflow.
- `interview_scheduled`: strong candidate calendar event/communication completed.
- `manual_review_notified`: average candidate was sent to hiring manager.
- `rejected`: weak candidate rejection was sent.
- `screening_error`: AI result was unusable; do not contact candidate yet.

## Contacted Flag

`contacted` starts as:

```text
NO
```

Workflow 2 sets it to:

```text
YES
```

only after the communication branch completes. This prevents the same candidate from receiving repeated emails.

## Important n8n Settings

Recommended settings:

- `Master_Sheet`: Always Output Data = ON.
- `Master-Sheet` in Workflow 2: Always Output Data can be ON, but no communication will happen if there are no rows.
- `Loop Over Items`: Batch Size = 1.
- `Basic LLM Chain`: use a non-reasoning model if possible.
- If using a model that returns escaped JSON or `<think>` text, keep `Code1` fallback parser.

## AI Model Notes

Some models, especially reasoning models, may return:

```text
<think>...</think>
```

or escaped JSON strings like:

```text
{\n "score": 65, ...}
```

The current `Code1` parser is designed to handle those cases. This is why the workflow is not fully dependent on the Structured Output Parser. If the parser fails but the LLM output still contains JSON, `Code1` can recover the result.

## Common Problems and Fixes

### Switch does not receive items in Workflow 2

Check:

- Is `If1` true branch producing items?
- Are candidate rows `status = screened` and `contacted = NO`?
- Is `Loop Over Items` connected to `Switch` from the `loop` output, not `done`?
- Are you running the full workflow rather than executing only the loop node?

### Candidate goes to false branch in If1

This is usually correct if:

```text
status != screened
```

or:

```text
contacted != NO
```

For testing, reset a row to:

```text
status = screened
contacted = NO
```

### Too many emails for strong candidates

Strong candidates can receive:

- a Google Calendar invite because they are added as attendee
- a custom Gmail interview email

If you want only one email per strong candidate, remove or bypass `Send Interview Email` and use only the Calendar invite.

### Average candidate marked rejected

Check the `Update contacted for avg and weak` status expression. It should normalize classification:

```js
String($('Loop Over Items').item.json['ai_classification'] || '').trim().toLowerCase() === 'average'
  ? 'manual_review_notified'
  : 'rejected'
```

### AI says strong one run and average another run

Use:

- strict prompt
- calibration examples
- low temperature
- `Code1` score/classification enforcement

The final sheet classification is forced from score:

```text
75+ strong
50-74 average
0-49 weak
```

### Resume download fails

Check:

- Google Drive credentials.
- File permissions.
- `drive_file_id` in master sheet.
- Whether the source resume URL is a valid Drive file link.

### Resume text is empty

Possible reasons:

- PDF is image-only/scanned.
- Extract From File cannot read the document.
- OCR is needed.

Expected outcome:

- AI should mark weak or `Code1` should mark `screening_error` if no usable AI output exists.

## Why This Architecture Is Safe

- Screening and communication are separate scheduled processes.
- Candidate updates match by `Candidate ID`, not email alone.
- Source rows are marked processed only after screening.
- Communication runs only for `screened` and `contacted = NO`.
- Average/weak branches update status after email.
- Strong branch stores calendar event link in the master sheet.
- AI parser has fallback logic so malformed model output does not silently corrupt rows.
- Multiple candidates can be processed in one run.

## Final Deliverable Notes

For the client or assignment walkthrough, explain the system like this:

1. Two source sheets feed into one normalized candidate schema.
2. A master sheet acts as the central database.
3. AI screening is role-aware and evidence-based.
4. Status fields control pipeline movement.
5. Communication runs separately to satisfy the assignment requirement.
6. Strong, average, and weak classifications each have a different path.
7. Data safety is handled with deterministic IDs, duplicate checks, retryable statuses, and contacted flags.

