# n8n_SKILL.md — n8n Build Rules for Claude Code

Read this file before building any n8n workflow. These rules encode hard-won lessons from production builds. Violating them produces workflows that import but fail at runtime.

---

## Trigger and Batch Processing

- Use a Manual Trigger + Google Sheets getAll node to read all rows at once when the workflow needs to process a list of items in a single run
- Never design for one-row-at-a-time trigger processing unless the use case genuinely requires it
- When deduplication is needed, read the log sheet separately and check in a Code node -- do not use the trigger to filter
- Google Sheets Trigger must use typeVersion 1 -- higher versions cause "Install this node" errors on import

---

## Node typeVersions

- Google Sheets Trigger: typeVersion 1
- Google Sheets (read/write): typeVersion 4
- Gmail: typeVersion 2
- HTTP Request: typeVersion 4.2
- Code: typeVersion 2
- IF: typeVersion 2
- LangChain chainLlm: typeVersion 1.4
- Always specify the n8n Cloud version in CLAUDE.md so typeVersions can be matched correctly

---

## Code Node Rules

- Always specify mode explicitly: runOnceForAllItems or runOnceForEachItem
- In runOnceForEachItem mode:
  - Use $json to access the current item -- never $input.first().json
  - Return a single object: return { json: {...} } -- never return [{ json: {...} }]
  - Cross-node references like $('NodeName').item.json work but lose context after HTTP Request nodes -- avoid where possible
- In runOnceForAllItems mode:
  - Use $input.all() to access all items
  - Return an array: return [{ json: {...} }, { json: {...} }]
- Never use $input.first() -- it only ever returns the first item regardless of how many items are flowing

---

## LLM Prompt Syntax

- All LLM prompt expressions must be wrapped in {{ }} -- this applies even when the prompt field is set to expression mode
- Never use JavaScript backtick template literals with ${ } -- n8n does not evaluate them
- String concatenation with + is valid inside {{ }} expressions
- Correct format:
  ```
  {{ 'Static text ' + $json.company + ' more static text ' + $json.summary }}
  ```
- Incorrect formats that will fail silently:
  ```
  `Static text ${ $json.company } more text`
  'Static text ' + $json.company + ' more text'   (without {{ }} wrapper)
  ```
- For multi-line prompts with \n, keep everything inside a single {{ }} expression using string concatenation

---

## Data Flow Between Nodes

- HTTP Request nodes and file processing nodes strip upstream item context
- After any HTTP Request node, the item only contains the response data -- all upstream fields are gone
- Always add a Code node or Edit Fields node after the last enrichment node to explicitly carry all fields forward onto the item
- Fields that must be carried forward: any ID, name, or contextual data needed by downstream nodes
- Downstream nodes must then use $json.fieldName -- not cross-node references -- to access this data
- Example: after scraping a website and fetching news, add a Code node that packages content, news, company, website, contactName, role, and email onto a single item before passing to the LLM

---

## Content Sanitization

Always sanitize raw scraped content before passing to any LLM. Use this pattern in a Code node:

```javascript
let cleaned = ($input.item.json.data || '').toString();
cleaned = cleaned.replace(/\[.*?\]\(https?:\/\/[^\)]+\)/g, '');
cleaned = cleaned.replace(/https?:\/\/\S+/g, '');
cleaned = cleaned.replace(/[\*#]+/g, '');
cleaned = cleaned.replace(/cookie|opt.out|preferences|privacy|GDPR|consent/gi, '');
cleaned = cleaned.replace(/\n+/g, ' ');
cleaned = cleaned.trim();
if (!cleaned || cleaned.length < 50) {
  cleaned = 'No meaningful content available for this company.';
}
cleaned = cleaned.substring(0, 3000);
```

- Cap content at 3000 characters to control token usage
- Always include the empty content fallback -- some pages return nothing useful after sanitization

---

## IF Node Conditions

- Boolean conditions must use expression mode on the left side: {{ $json.fieldName }}
- Use the 'is true' or 'is false' operator -- never 'equal to true' or 'equal to false' -- to avoid type mismatch errors
- IF node conditions do not always import correctly from JSON -- verify and fix manually after import
- The condition left side showing as a static value like "true" instead of an expression is a common import failure

---

## Gmail and Logging Pattern

- Gmail node does not pass data through -- it terminates the branch
- Never wire a logging node downstream of Gmail -- it will receive no data
- Correct pattern for workflows that send email AND log:
  - IF score routing True branch → Email Writer LLM → Gmail (terminates here)
  - IF score routing True branch → Log to Sheets (second wire from same IF output)
  - IF score routing False branch → Log to Sheets
- Both branches of the score IF node connect directly to the logging node
- The email branch is a side branch that terminates at Gmail

---

## Deduplication Pattern

- Read the log sheet with a getAll node triggered from the same manual trigger (fan-out)
- Use a Code node in runOnceForEachItem mode to check each company against the log
- Pass the log rows as a field on each item (summaryRows array) for downstream checking
- Dedup check example:
```javascript
const { company, summaryRows } = $json;
const cutoff = Date.now() - (30 * 24 * 60 * 60 * 1000);
let isRecent = false;
if (summaryRows && summaryRows.length > 0) {
  isRecent = summaryRows.some(row => {
    const rowCompany = (row.Company || '').trim();
    if (rowCompany.toLowerCase() !== company.toLowerCase()) return false;
    if (!row.Recency) return false;
    return new Date(row.Recency).getTime() > cutoff;
  });
}
return { json: { isRecent, company, ...otherFields } };
```
- Use $now.toISO() for timestamps written to the log sheet

---

## Structured Output from LLMs

- Do not use n8n Structured Output Parser nodes -- they inject conflicting schema instructions that cause model output failures
- Instead, instruct the LLM in the prompt to return raw JSON only with no markdown fences
- Add a Code node after the LLM to parse the JSON with a try/catch and regex fallback:
```javascript
const raw = ($json.text || '').trim();
let score = 5;
let rationale = 'Could not parse LLM output.';
try {
  const cleaned = raw.replace(/```(?:json)?\s*/gi, '').replace(/```\s*/g, '').trim();
  const parsed = JSON.parse(cleaned);
  score = Math.min(10, Math.max(1, parseInt(parsed.score, 10)));
  rationale = parsed.rationale || rationale;
} catch (e) {
  const scoreMatch = raw.match(/"score"\s*:\s*(\d+)/);
  const rationaleMatch = raw.match(/"rationale"\s*:\s*"([^"]+)"/);
  if (scoreMatch) score = Math.min(10, Math.max(1, parseInt(scoreMatch[1], 10)));
  if (rationaleMatch) rationale = rationaleMatch[1];
}
return { json: { score, rationale } };
```

---

## Browser Compatibility

- Use Chrome or Edge for n8n -- Firefox has websocket instability issues with the n8n editor
- Save frequently with Ctrl+S during active build and debug sessions

---

## Build Sequence for Token Management

- Break large workflow builds into phases to stay within Claude Code token limits
- Recommended phases:
  - Phase 1: Trigger, data reading, dedup logic
  - Phase 2: Enrichment (scraping, APIs, sanitization), first LLM
  - Phase 3: Scoring LLM, routing IF node
  - Phase 4: Output nodes (email, logging)
- End each phase prompt with an explicit instruction to stop and wait
- After each phase, retrieve the JSON from the GitHub branch before starting the next phase
