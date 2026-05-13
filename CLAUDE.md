# New Hire Onboarding Workflow — Project Specification

## Overview

An n8n automation workflow that ingests new hire data from a Google Sheet, uses a Groq LLM to generate personalized onboarding content, sends HTML emails to the new hire and their manager, logs action items in Google Tasks, records the completed record in a Status Google Sheet, and removes the processed row from the input sheet.

---

## Platform & Output

- **Automation platform:** n8n
- **Output format:** Single `.json` file importable into n8n
- **LLM provider:** Groq (free tier) — model: `llama-3.3-70b-versatile`
- **Rate limiting:** 3-second wait node between each Groq API call to respect free-tier limits

---

## Google Sheets

Both sheets live in the same Google Sheets file on Google Drive.

### New Hire Sheet (Trigger / Input)

Columns (in order):

| Column | Description |
|---|---|
| First Name | New hire's first name |
| Last Name | New hire's last name |
| Role | Job title |
| Department | Department name |
| Start Date | Employment start date |
| Manager | Manager's full name |
| Manager Email | Manager's email address |
| Contact Email | New hire's email address (destination for welcome email) |
| Plan Tier | Employment type (e.g. Full Time, Part Time, Contract) |

- **Trigger:** n8n Google Sheets trigger fires on each new row added.
- **Post-processing:** After a row is fully processed and written to the Status sheet, the corresponding row is **deleted** from this sheet.

### Status Sheet (Output / Archive)

Columns (in order):

| Column | Description | Source |
|---|---|---|
| Timestamp | Date/time the workflow processed the record | Workflow-generated (now) |
| First Name | | From New Hire sheet |
| Last Name | | From New Hire sheet |
| Role | | From New Hire sheet |
| Department | | From New Hire sheet |
| Start Date | | From New Hire sheet |
| Manager | Manager's full name | From New Hire sheet |
| Contact Email | New hire's email address | From New Hire sheet |
| Welcome Email Sent | Was welcome email sent? | "Yes" / "No" |
| Manager Email Sent | Was manager email sent? | "Yes" / "No" |
| Tasks Created | Were Google Tasks created? | "Yes" / "No" |
| Scheduled Check-In Date | 30-day check-in date | Calculated: Start Date + 30 days |
| Onboarding Status | Current onboarding state | Set to "Initiate" on first write |
| Plan Tier | Employment type | From New Hire sheet |

---

## LLM Calls (Groq — 4 calls per new hire)

All calls use model `llama-3.3-70b-versatile`. A **3-second wait node** is inserted between each call.

### Call 1 — Welcome Message

**Purpose:** Personalized 3–5 sentence welcome email body for the new hire.

**Prompt template:**
```
You are an onboarding HR lead at a prestigious company that builds the most exquisite widgets in North America. Based on the following new hire data, personalize a 3 to 5 sentence welcome email to the new hire. Keep the tone warm, professional, and encouraging.

New Hire Name: {{First Name}} {{Last Name}}
Role: {{Role}}
Department: {{Department}}
Start Date: {{Start Date}}
Manager: {{Manager}}
Plan Tier: {{Plan Tier}}
```

---

### Call 2 — 30/60/90 Day Plan

**Purpose:** Role- and department-specific onboarding plan. Each phase is 2–3 sentences.

**Prompt template:**
```
You are an onboarding HR lead at a prestigious company that builds the most exquisite widgets in North America. Generate a personalized 30/60/90 day onboarding plan for the following new hire. Each phase (30, 60, and 90 days) should be 2 to 3 sentences and tailored specifically to their role and department.

New Hire Name: {{First Name}} {{Last Name}}
Role: {{Role}}
Department: {{Department}}
Start Date: {{Start Date}}
Manager: {{Manager}}
Plan Tier: {{Plan Tier}}

Format your response as:
30 Days: [text]
60 Days: [text]
90 Days: [text]
```

---

### Call 3 — Action Items

**Purpose:** 4–5 total action items derived from the 30/60/90 plan, all owned by the manager.

**Prompt template:**
```
Based on the following 30/60/90 day onboarding plan for {{First Name}} {{Last Name}} ({{Role}}, {{Department}}), generate 4 to 5 concise action items that the manager ({{Manager}}) must complete to support this new hire's onboarding. Each action item should reference the relevant phase (30, 60, or 90 days) and be specific and actionable.

30/60/90 Plan:
{{30_60_90_plan_output}}
```

---

### Call 4 — 30-Day Agenda

**Purpose:** A detailed 30-day onboarding agenda for the manager's reference.

**Prompt template:**
```
You are an onboarding HR lead at a prestigious company that builds the most exquisite widgets in North America. Based on the new hire information and their 30/60/90 day plan below, generate a concise 30-day onboarding agenda that the manager ({{Manager}}) can use to guide {{First Name}} {{Last Name}} through their first month. Keep it practical and role-specific.

New Hire Name: {{First Name}} {{Last Name}}
Role: {{Role}}
Department: {{Department}}
Start Date: {{Start Date}}
Plan Tier: {{Plan Tier}}

30/60/90 Plan:
{{30_60_90_plan_output}}
```

---

## Emails (Gmail — HTML format)

### Email 1 — New Hire Welcome Email

- **To:** `Contact Email`
- **From:** Configured Gmail OAuth2 account
- **Subject:** `Welcome to the team, {{First Name}}!`
- **Format:** HTML
- **Body includes:**
  - Welcome message (from LLM Call 1)
  - 30/60/90 Day Plan (from LLM Call 2)

---

### Email 2 — Manager Notification Email

- **To:** `Manager Email`
- **From:** Configured Gmail OAuth2 account
- **Subject:** `New Hire Onboarding Plan — {{First Name}} {{Last Name}}`
- **Format:** HTML
- **Body includes:**
  - 30-Day Agenda (from LLM Call 4)
  - Action Items (from LLM Call 3)

---

## Google Tasks

- **Task list name:** `Proj5 NewHires`
- **One task per action item** (4–5 tasks per new hire)
- **Task title format:** `[Manager Name] — [Action Item text]`
- **Task notes:** Include new hire name, role, and which phase (30/60/90) the item belongs to
- **Due dates:**
  - 30-day action items → Start Date + 30 days
  - 60-day action items → Start Date + 60 days
  - 90-day action items → Start Date + 90 days
- **Owner:** Manager (name included in task title; Google Tasks does not support cross-account assignment natively)

---

## Error Handling

- **Strategy:** Retry on failure
- **Retry attempts:** 3
- **Backoff:** Exponential (2s → 4s → 8s)
- **Error notifications:** None configured at this time

---

## n8n Credentials (as configured in n8n)

| Service | Credential Name in n8n |
|---|---|
| Gmail | `Gmail OAuth2 API` |
| Google Sheets | `Google Sheets OAuth2 API` |
| Google Tasks | `Google Tasks OAuth2 API` |
| Groq | `Groq account` |

---

## Workflow Node Sequence (High-Level)

```
Google Sheets Trigger (New Row)
  └─► Set Node (normalize/map fields)
        └─► Groq: Welcome Message
              └─► Wait (3s)
                    └─► Groq: 30/60/90 Plan
                          └─► Wait (3s)
                                └─► Groq: Action Items
                                      └─► Wait (3s)
                                            └─► Groq: 30-Day Agenda
                                                  └─► Gmail: Welcome Email → new hire
                                                        └─► Gmail: Manager Email
                                                              └─► Google Tasks: Create tasks (loop per action item)
                                                                    └─► Google Sheets: Append row to Status sheet
                                                                          └─► Google Sheets: Delete row from New Hire sheet
```

---

## Key Assumptions & Decisions

- `Scheduled Check-In Date` is calculated by the workflow as `Start Date + 30 days` (not pre-filled).
- `Onboarding Status` is set to `"Initiate"` on first write to Status sheet.
- `Welcome Email Sent`, `Manager Email Sent`, `Tasks Created` are written as `"Yes"` on success, `"No"` on failure.
- `Timestamp` on Status sheet is the moment the workflow processes the row.
- The New Hire row is only deleted after all steps complete successfully.
- Groq free tier: 3-second waits between LLM calls; 3 retries with exponential backoff on any node failure.
- Google Tasks does not support assigning tasks to other users natively; manager name is embedded in task title and notes.
