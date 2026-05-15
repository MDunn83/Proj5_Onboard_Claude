# n8n and Claude Code Lessons Learned

Mark Dunn | Automation & AI Orchestration Portfolio | May 2026

This is a living pattern library. Every entry is a transferable lesson -- something that burned time once and should never burn time again. Organized by topic so you can look up a problem type, not hunt through project history. The Build Summary doc is where project-specific narrative and benchmarks live.

---

# Section 1: n8n Patterns

---

## 1.1 Parallel Branches and Fan-In

**Problem: Parallel branches trigger downstream nodes multiple times**

When multiple LLM branches run simultaneously, each branch "pushes" its own execution downstream. A workflow with 4 parallel branches will execute every downstream node 4 times.

Resolution: Introduce a Merge node in Append mode with one input per parallel branch. This forces the workflow to wait until all branches check in before passing any data. Follow the Merge node with an Aggregate node set to Execute Once to collapse all items into a single JSON object with a `data` array.

**Problem: All parallel branch outputs have the same field name**

When parallel LLM chains all output a field called `text`, it is impossible to tell which output is which after merging.

Resolution: Add an Edit Fields (Set) node immediately after each LLM chain to rename the generic `text` output to a labeled key -- `summary`, `actions`, `decisions`, `qdb`, etc. The aggregated object then becomes a clearly labeled map that downstream nodes can reference predictably.

This same pattern applies to any parallel classifier branches. When two classifier branches each output a field called `classification`, only one survives the merge. Add an Edit Fields node after each branch's Code node to rename fields with a branch-specific prefix before the Merge. Example: `query_classification` and `response_classification` instead of `classification` on both. Zero field name overlap is the rule.

**Choosing the right Merge mode**

- Append mode: stacks all items from all inputs into a single list. Use when collecting outputs from parallel branches that each produce one item and you want to fan-in before aggregating. Note: 5 Jina items plus 5 news items in Append mode equals 10 items -- use Combine By Position instead when parallel branches process the same set of records.
- Combine By Position: zips items from multiple inputs by index order. Use when two parallel branches process the same companies or documents in the same order and you need one merged item per record. Enable "Include Any Unpaired Items" to prevent the Merge from dropping items when one branch produces fewer items than expected.
- Combine By Key: joins items using a shared unique identifier. Use when branches process multiple items at different speeds or produce different item counts. Always the safer choice for multi-document or multi-record pipelines. Requires a unique identifier (file_id, company_id, etc.) threaded through every node.

**Problem: Merge node parameter name mismatch on import**

The Merge node configured with `"combinationMode": "mergeByPosition"` in the JSON silently defaults to "Match Fields" mode on import, which fails because no matching fields are defined.

Resolution: The correct parameter is `"combineBy": "combineByPosition"` -- a different key name and a different value format. Always export a working workflow from the n8n UI and diff it against generated JSON. The UI export is the ground truth for parameter names.

**Pattern: Merge node as a synchronization gate, not a data aggregator**

A Merge node does not need to combine meaningful data from all its inputs. When the goal is to ensure all branches complete before a downstream node fires, the Merge node serves as a synchronization gate. The actual payload flows through a single input; the other inputs exist only to signal completion. This pattern ensures the log row is never written unless all branches succeed.

In Project 5, Merge1 has 4 inputs: both Gmail nodes, the Limit node (task creation confirmation), and Check_In (the only input carrying meaningful data). Three inputs are completion signals. One carries data. All four are necessary.

**Pattern: Limit node as a synchronization reducer**

When a multi-item branch needs to feed a synchronization Merge node, the Limit node set to maxItems: 1 is the cleanest reduction tool. It passes the first item and discards the rest, converting N items into 1 without aggregating or transforming data.

In Project 5, Create a task produced 5 items (one per action item). Wiring it directly into Merge1 caused 5 log rows per hire. A Limit node reduced the output to 1 item.

Use Limit over Aggregate when the downstream node only needs a completion signal, not the actual data. Use Aggregate when the collapsed data is needed downstream.

Any branch that produces multiple items and feeds a synchronization Merge node needs a Limit node. Add this to the mandatory post-build checklist.


---

## 1.2 Data Threading and Identifier Management

**Problem: File processing nodes strip all upstream metadata**

The Extract from File node and Download File node output only the file content fields. All upstream metadata -- file_id, filename, and any other fields -- is lost after these nodes.

Resolution: Add an Edit Fields (Set) node immediately after any file processing node. Use $('Node Name').item.json.fieldName syntax to reach back upstream and reattach critical identifiers alongside the extracted content. Treat file_id like a thread that must be stitched through every node from start to finish.

**Problem: Cross-node references lose context after HTTP Request nodes**

After Scrape Website or Fetch News HTTP Request nodes run, fields from upstream nodes (company name, contact, website URL) are no longer reliably on the current item. Cross-node references stop resolving correctly.

Resolution: Add an Edit Fields or Code node after the last enrichment node to explicitly carry all needed fields forward onto the current item. Downstream nodes should reference $json.fieldName, not cross-node expressions, for any field that needs to survive an HTTP Request node. The carry-forward node should explicitly place all needed context fields -- userId, query, startTime, usage metadata -- onto its output item. All downstream nodes reference $json.* instead of cross-node refs.

**Problem: Reaching back to trigger data from a downstream branch**

When a re-engagement or alternate branch fires, the item flowing through that branch may contain log sheet data rather than the original trigger data. Fields like Website URL that only exist in the trigger are not on the current item.

Resolution: Use $('Trigger Node Name').item.json.FieldName to reach back to the trigger directly. Trigger data is always available upstream regardless of which branch is currently executing.

---

## 1.3 Structured Output and JSON Parsing

**Pattern: Prompt Tug-of-War**

Symptom: "Model output doesn't fit required format" error.

Root cause: The system prompt contains JSON formatting instructions while a Structured Output Parser is also sending its own schema contract to the model. Both are telling the model how to format output -- they conflict.

Resolution: Strip all JSON formatting instructions, format examples, and structure directives from the system prompt entirely. The system prompt tells the model WHAT to extract. The Structured Output Parser exclusively owns HOW to format it. Never both.

**Problem: LLM output is a string even when it looks like JSON**

LLMs always output strings. Even when the output looks like a JSON array, n8n treats it as a string. Nodes like Split Out that require a true typed array will fail or output the entire block as one item.

Resolution (Option A -- preferred for looping): Use a Structured Output Parser node with a defined schema to force the LLM to output machine-readable typed data natively.

Resolution (Option B -- preferred for direct field access): Use JSON.parse($json.text) in a downstream Code node or expression. More resilient than the parser when output goes straight to Gmail, Sheets, or any node that just needs text fields. Always treat LLM output as dirty -- use .trim() before parsing and implement a regex fallback to isolate the JSON block from any conversational preamble.

**When to use Structured Output Parser vs inline JSON.parse**

- Use the Structured Output Parser when a downstream Split Out or loop requires a true typed array.
- Skip the parser and use JSON.parse in a Code node when output goes straight to Gmail, Sheets, or a node that only needs text fields. The parser adds fragility when it is not strictly necessary.

**Problem: Parser schema rejects null values for optional fields**

When the LLM returns null for a field it cannot find in the document, the parser rejects it because null does not match type: "string".

Resolution: Define optional fields in the schema as type: ["string", "null"] instead of type: "string". Required fields stay as type: "string". Schema design should reflect real-world document variability -- not every document has every field.

**Problem: Parser Schema Type set to wrong mode**

The "Generate From JSON Example" mode makes all properties required and does not support null types, causing immediate format errors on any document with missing fields.

Resolution: Always use "Define using JSON Schema" for the Structured Output Parser. "Generate From JSON Example" is a shortcut that creates rigid schemas incompatible with real-world document variability.

**Problem: Token limit causes empty parser response**

Symptom: "The AI model returned an empty response to the Structured Output Parser." The model runs out of tokens before completing structured output, often caused by a free-text field attempting to reproduce lengthy content verbatim.

Resolution: Add explicit word limits to any free-text summary fields in the prompt (e.g., "20 words or less"). Switch to a larger model for complex structured extraction -- small fast models are better for simple classification tasks. Reference model pairing: llama-3.1-8b-instant for classification, llama-3.3-70b-versatile for structured extraction.

**Problem: Split Out node field name vs field value confusion**

Symptom: "split is not a function" or Split Out outputs "1 item" instead of multiple items.

Root cause: The "Field to Split Out" box expects a field name (a string label), not an expression passing the field value.

Resolution: Toggle off Expression mode in the Split Out node and enter the simple string label of the target array (e.g., action_items).

**Problem: Dirty LLM output breaks JSON parsing**

LLMs frequently include invisible characters, newlines, or conversational prefixes (e.g., "Sure, here is your JSON:") before the actual data begins. Standard JSON parsers fail if the first character is not a bracket.

Resolution: Always use .trim() before parsing to strip leading and trailing whitespace. Implement a regex-based extraction to isolate the JSON block from any conversational filler. Treat all LLM output as dirty data by default.

**Problem: `response_format: json_object` breaks with reasoning models and low max_tokens**

When classifier nodes use `response_format: { type: "json_object" }` and a low max_tokens value (e.g., 60), reasoning models consume their entire token budget on internal thinking, leaving nothing for the JSON response. The result is `failed_generation: ""`.

Resolution: Remove `response_format` entirely. Increase max_tokens to at least 200 for classifier nodes. Instruct the model in the prompt to return raw JSON only. Add a Code node with try/catch and regex fallback to parse the output. `response_format: json_object` and reasoning models do not mix safely.

**Problem: LLMs can emit literal backslash-n strings instead of newlines**

Some models return responses containing the two-character sequence `\n` as literal text rather than as an actual newline character, causing downstream formatting and parsing issues.

Resolution: Add this replacement in any Code node that processes LLM output:
```javascript
text = text.replace(/\\n/g, ' ');
```
In a JSON workflow file this must be written as `\\\\n` to survive double-escaping. Never assume LLM output is clean. Always run it through a sanitization Code node before passing downstream.

**Pattern: Use explicit delimiters instead of newlines for LLM list extraction**

When an LLM prompt requests a newline-separated list, models frequently ignore the instruction and produce a single paragraph. Newlines are invisible and ambiguous.

Resolution: Instruct the model to separate items with a concrete delimiter on its own line:

```
Separate each item with this exact text on its own line: ---
```

Then split in the downstream Code node:

```javascript
const items = text
  .split("---")
  .map(item => item.trim())
  .filter(item => item.length > 0);
```

This resolved the action item blob problem in Project 5 on the first attempt after newline separation had failed. Use `---` as the default list extraction delimiter for any workflow that needs a split-able LLM output array.

**Problem: Variable referenced before being pulled off $json**

Symptom: `ReferenceError: text_clean is not defined`

Root cause: Code node references a field by its bare variable name before assigning it from `$json`.

Resolution: Every Code node must open by explicitly assigning all needed fields:

```javascript
let text_clean = $json.text_clean || "";
```

Never reference a field name directly in a Code node without first pulling it off `$json`.


---

## 1.4 LLM Prompt Behavior

**Problem: Email or Sheets output renders as raw JSON**

When the LLM prompt requests a JSON object, the JSON is dumped directly into the email body or sheet cell, producing unreadable output full of curly braces and quotes.

Resolution: Rewrite the prompt to request plain-text bullet points instead of JSON for any node whose output goes directly to Gmail or Sheets. Reserve JSON output for nodes feeding a parser or a loop.

**Problem: Gmail output renders as a wall of text**

Despite HTML mode being enabled, the Gmail node ignores LLM newlines (\n), producing a single unreadable block of text.

Resolution: Apply a JavaScript regex in the Gmail expression to convert newlines to HTML line breaks: {{ $json.fieldName.replace(/\n/g, '<br>') }}. This must be applied to every field in the email body.

**Problem: Paragraph formatting not produced by the LLM**

Post-processing with .replace(/\n/g, '<br>') only works if the LLM actually produces \n between paragraphs. If the model writes one continuous paragraph, no downstream expression fixes it.

Resolution: Switch models before spending time on prompt iteration. The right model produces the correct format naturally. Switching to a better model (openai/gpt-oss-20b in Project 6) resolved paragraph formatting without any explicit prompt instruction.

**Problem: Partial data extraction -- truncation at comma**

Using .split(',') on a string that contains multiple commas stops at the first comma, returning only a partial value (e.g., returning only the day of the week instead of the full date).

Resolution: Use existing labels in the text as anchors for splitting. Split by the next label (e.g., .split('Participants:')). Select index [0] for the preceding data or [1] for the following data. Use .replace() to clean the label and .trim() for whitespace.

**Problem: Redundant metadata appearing in the main body**

After extracting Date and Participants from the top of an LLM output string, those fields still appear at the top of the Summary field because the original string was not trimmed.

Resolution: Use .slice() to remove the header lines from the body text. Split on \n to get an array of lines, use .slice(1) or .slice(2) to drop the header lines, then .join('\n') to reassemble. Example: {{ $json.data[0].summary.split('\n').slice(1).join('\n').trim() }}

**Problem: LLM hallucinating passthrough field values**

When a prompt instructs the LLM to return a field like file_id or company_name but that field is not included in the input, the model invents placeholder values like "not provided", "unknown", or "N/A".

Resolution: Any field the LLM is expected to return must be explicitly included in the user message input. System prompt = instructions. User message = data. Both must be correct for the output to be correct. Hallucinated placeholders are always a signal that the model was asked to return something it was never given.

**Problem: Output format example interpreted as output format**

If the classifier prompt instructs "Output format: classification: SENSITIVE / domain: LEGAL" without the word "example," the model may interpret this as an instruction to always output those exact values regardless of the actual input.

Resolution: Always include the word "example" in the format instruction. "Output format example: classification: SENSITIVE / domain: LEGAL" makes clear this is illustrative, not prescriptive. One word difference, significant behavioral difference.

**Problem: Reasoning model outputs reasoning trace in response**

Models with a thinking or reasoning mode (e.g., qwen/qwen3-32b) output extended reasoning wrapped in `<think>...</think>` tags before their actual answer. These blocks appear in the cleaned response and in classifier output, breaking downstream parsing.

Resolution: Add `"reasoning_effort": "none"` to every API request body for any model with a reasoning mode. Also add a regex strip in the Code node as a safety net:
```javascript
text = text.replace(/<think>[\s\S]*?<\/think>/gi, '');
```
Any model with a reasoning or thinking mode must have that mode explicitly disabled unless the reasoning trace is specifically needed.

**Problem: User message fields concatenated with no separators**

When n8n expressions for multiple fields are placed adjacent to each other in the user message without labels or line breaks, all values concatenate into one unreadable blob. Example of what the LLM receives: `JordanMillsEngineering2026-06-01Priya Nair`

Resolution: Always add explicit field labels and line breaks. Every field gets its own labeled line:

```
First Name: {{ $json['First Name'] }}
Last Name: {{ $json['Last Name'] }}
Role: {{ $json.Role }}
Department: {{ $json.Department }}
Start Date: {{ $json['Start Date'] }}
Manager: {{ $json.Manager }}
```

Labels are not optional -- they are what make the data parseable by the model. This is the data half of the system prompt = instructions, user message = data rule.

**Pattern: Splitting a single LLM output into labeled sub-sections using regex anchors**

When an LLM generates a structured document with predictable section labels (e.g. a 30-60-90 day plan), regex anchors can split the output into separate named fields without requiring JSON output from the model:

```javascript
const day30Match = text.match(/30.day[\s\S]*?(?=60.day|$)/i);
const day60Match = text.match(/60.day[\s\S]*?(?=90.day|$)/i);
const day90Match = text.match(/90.day[\s\S]*/i);
```

The dot in `30.day` matches both "30-day" and "30 Day". The lookahead `(?=60.day|$)` stops the match at the next section label. If all three fields come back empty, the LLM's section labels do not match the regex anchors -- check `text_clean` and adjust the anchors to match.

**Problem: Dead code in Code nodes producing no output**

When a Code node computes variables but does not include them in the return statement, the values are silently discarded. The node runs without error but the fields are not available downstream.

Resolution: Every Code node's return statement must explicitly list every field downstream nodes will need. After any copy-paste or refactor, audit the return statement against what downstream nodes actually reference.


---

## 1.5 Model Selection

**Rule: Switch models before iterating on the prompt**

When structured output is failing or formatting is wrong, the instinct is to keep adjusting the prompt. This is usually the wrong move. Different models behave very differently with identical prompts and parser configurations. Prompt iteration on the wrong model is a losing strategy.

Resolution: Switch models first. If the new model produces correct output, the prompt was fine -- the model was the problem.

**Project 6 model evaluation (Groq, structured output)**

- llama-3.3-70b-versatile: reliable with original prompt, breaks immediately on any modification
- llama-3.1-8b-instant: too small for complex structured output, runs out of tokens
- llama-4-scout-17b-16e-instruct: handles modified prompts better but intermittently wraps output in markdown fences
- openai/gpt-oss-120b: excellent output quality, hits rate limits quickly on free tier
- openai/gpt-oss-20b: reliable structured output, clean paragraph breaks, stays within rate limits -- recommended for outreach generation

**Project 4 model evaluation (governance and sensitive query workflows)**

- Mistral (free tier): response quality good, mostly plain text with some residual markdown. Classifier nodes hung indefinitely without error on every attempt. Root cause unknown. Unsuitable for this workflow.
- Groq openai/gpt-oss-20b (free tier): hallucinated specific financial figures, fabricated product launch timelines with vendor names, invented security vault entry names, and described specific employee activities on sensitive queries. Hit free tier rate limits before completing 30 questions. Hit rate limits immediately on classifier nodes. Unsuitable for governance workflows at any meaningful volume.
- Gemini 2.0 Flash (free tier): appropriate refusals on all sensitive queries, clean plain text output, completed 90 API calls across three nodes on a 30 question run without rate limit issues. Selected model for v1.
- qwen/qwen3-32b (Groq free tier, 60 RPM): produced working output after disabling reasoning mode. Hallucinated financial data on sensitive queries similar to Groq's gpt-oss-20b. Suitable for classification tasks; not suitable for response generation in sensitive domains.

**Model selection has governance implications, not just performance implications**

The same system prompt and the same question set produced radically different risk profiles across models. Groq hallucinated confidential-looking financial data on the exact queries a governance tool is supposed to catch. Gemini refused them appropriately. A governance framework is only as reliable as the model running inside it. Model evaluation against a defined sensitive question set should be a standard pre-deployment step for any AI governance pipeline.

**Groq free-tier model availability changes without notice**

Free-tier models on Groq turn over frequently. gemma2-9b-it was deprecated mid-build with no warning during Project 4. Check the current Groq model list before starting any build. Do not assume a model available last week is still available today.

**Model pairing by task type**

- Classification (simple, fast): llama-3.1-8b-instant
- Complex structured extraction: llama-3.3-70b-versatile
- Summary, scoring, outreach generation: openai/gpt-oss-20b via Groq or Gemini 2.5 Flash
- Governance and sensitive query workflows: Gemini 2.5 Flash (free tier, appropriate refusals, no hallucination on sensitive content)
- Temperature tuning: 0.3 for summary, 0.1 for scoring, 0.7 for outreach email generation

**Plain text instructions are not universally respected -- model first, prompt second**

Mistral and several Groq models produced markdown formatting including bold markers and numbered lists despite explicit system prompt instructions to use plain text only. Iterating on prompt wording did not resolve it. The correct move is to switch models before adjusting the prompt. For maximum specificity when formatting compliance matters, name the specific elements to avoid: "Do not use asterisks, hashes, bullet points, numbered lists, bold text, or any formatting characters. Write in plain sentences only." This is more reliable than "no markdown" as a standalone instruction. If a model ignores a formatting instruction, it will likely continue to ignore variations of it.

**Project 5 model evaluation (Groq, text generation)**

- qwen3-32b (Groq free tier): produced `<think>` reasoning traces in every response before the actual output. Broke downstream sanitization. Do not use for response generation nodes without disabling reasoning mode first.
- llama-3.3-70b-versatile (Groq free tier): clean output after switching from qwen3-32b. Followed delimiter instructions reliably. Produced literal `\n` strings requiring the double backslash regex fix but otherwise stable. Selected model for all 4 chains in Project 5.


---

## 1.6 Node-Specific Behaviors

**Problem: Static vs dynamic mapping -- all output items have the same value**

Symptom: Google Tasks created multiple items but all were named "List" (or any other static value).

Root cause: The Title field was set to static text. n8n creates one item per incoming record but uses the same static value for every one.

Resolution: Delete the static text and use a dynamic expression: {{ $json.task }}. This instructs the node to pull the specific value for each individual row.

**Problem: Get Rows node returns no data**

The Get Rows node runs once per input item by default. With multiple items flowing in, it attempts to run multiple times but returns no output because it has no data to pass through -- it only reads the sheet.

Resolution: Enable Execute Once in the node Settings tab so it runs a single time and returns the full sheet contents.

**Problem: Gmail node terminates the branch**

The Gmail node sends the email and stops. It does not pass data through to downstream nodes. Any logging node wired downstream of Gmail receives no data.

Resolution: Wire logging nodes directly from the routing IF node (or equivalent), not from the Gmail node. The email-sending branch terminates at Gmail. Treat Gmail as a dead end in the wiring diagram.

**Problem: Compare Datasets branch selection inverted**

Input A should be the new leads from the trigger. Input B should be the log sheet. "In A only Branch" outputs companies in the trigger that are not in the log -- these are new leads to process. Wiring inputs backwards sends new leads down the wrong branch.

Resolution: Confirm Trigger is Input A and the log sheet Get Rows is Input B. Wire the processing branch from "In A only Branch."

**Problem: IF node conditions do not survive import**

IF node conditions frequently do not import correctly -- the left side may show as a static string instead of an expression, or Boolean conditions may have type mismatches.

Resolution: Treat IF node condition verification as a mandatory post-import check on every build. Manually inspect every IF node after import. For Boolean conditions, set the operator to "is false" and confirm expression mode is active on the left side. IF node conditions are the single most import-fragile part of any n8n workflow. Always check them first when a workflow runs but routes incorrectly.

**Problem: Google Drive Move node fails with 'My Drive' parent setting**

The Move file node returns a 404 "File not found" error even when the file exists. The "My Drive" dropdown in the Parent Drive setting is unreliable for files in nested folder structures.

Resolution: Change Parent Drive from the "My Drive" dropdown to "By ID" and enter the explicit folder ID of the source folder. Store all folder IDs at the start of a project build -- they are needed more than expected.

**Problem: Switch routing rule does not match LLM output**

A Switch branch never triggers even though documents of that type are flowing through. The Switch rule checks for one string but the LLM returns a slightly different string.

Resolution: The Switch routing rule must match LLM output exactly -- case sensitive, character for character. When a branch is not triggering, check whether the rule value matches the actual LLM output value, not the expected value. Define category names once in the classification prompt and build all Switch rules around those exact strings.

**Problem: Google Sheets node requires `__rl` format for references**

The Google Sheets node (typeVersion 4) requires spreadsheet and sheet references in the resource-locator format. Using a plain string value causes the node to show an error on import.

Resolution: Always use the `__rl` format for Google Sheets node references in typeVersion 4:
```json
"documentId": { "__rl": true, "value": "<id>", "mode": "id" },
"sheetName": { "__rl": true, "value": "<name>", "mode": "name" }
```

**Pattern: Static confirmation fields as a valid audit log approach**

When a workflow branches into multiple parallel paths, wiring every branch back to the logging node creates item count multiplication. Static fixed values are a valid alternative when workflow execution itself is the audit trail.

If any upstream branch fails, n8n halts and the log row is never written. A row in the log sheet is proof all branches completed. Static `Yes` values for confirmation fields are a logical guarantee based on workflow architecture, not a shortcut. The Merge synchronization gate pattern enforces this guarantee.

**Problem: Hardcoded email addresses in Gmail sendTo field**

Hardcoded personal email addresses in sendTo fields cause all emails to route to that address regardless of the actual recipient. The workflow appears to work during testing but is functionally broken for real use.

Resolution: Always use dynamic expressions pulling from trigger data:

```
{{ $('Google Sheets Trigger').item.json['Contact Email'] }}
{{ $('Google Sheets Trigger').item.json['Manager Email'] }}
```

A static email address in a sendTo field is always a bug in production context. Add hardcoded sendTo to the mandatory pre-export PII checklist.

**Problem: Limit node exported with empty parameters**

A Limit node added to the canvas may export with an empty parameters object `{}` if maxItems was not explicitly saved before export. On import, the node defaults to an unspecified limit that may vary by n8n version.

Resolution: Confirm maxItems is explicitly set and visible in the Parameters panel before exporting. Add Limit node parameter verification to the post-export checklist.


---

## 1.7 External APIs and Rate Limiting

**Problem: NewsAPI free tier blocks cloud-hosted requests**

NewsAPI free tier only returns results from localhost. Requests from cloud-hosted environments like n8n Cloud return totalResults: 0 with empty articles arrays. This is a deliberate restriction to push users to paid plans.

Resolution: Switch to GNews (gnews.io). Free tier works from cloud environments, 100 requests per day, compatible article structure. Use Send Query Parameters in the HTTP Request node instead of embedding parameters directly in the URL -- n8n does not support encodeURIComponent() in URL fields.

**Problem: Rate limiting on free API tiers**

Firing multiple simultaneous HTTP requests to GNews or similar free-tier APIs triggers 429 rate limit errors.

Resolution: Use the Batching option in the HTTP Request node. Set Items per Batch to 1 and Batch Interval to 2000ms. This spaces requests 2 seconds apart and stays within free tier limits.

**Problem: Batching must be applied to every HTTP Request node that calls a rate-limited API**

Batching added only to the first HTTP Request node in a pipeline leaves all subsequent nodes unthrottled. Classifier nodes or enrichment nodes hitting the same API will trigger rate limits immediately on the first test run.

Resolution: When rate limits apply to an API, every node that calls that API needs the same batching configuration -- not just the first one. For Groq free tier (60 RPM), set batchSize: 1 and batchInterval: 4000ms on all HTTP Request nodes hitting the Groq endpoint.

**Better alternative: Google News RSS**

Google News RSS is completely free, requires no API key, has no rate limits, and returns recent news reliably. Use it over GNews whenever Claude Code or a new build is starting fresh. GNews is only necessary when RSS is insufficient.

**Free tier rate limits are a hard operational constraint for governance workflows**

At any realistic enterprise query volume, free tier models are not viable for production governance pipelines. Two of three models tested in Project 4 could not complete a 30 question run on free tiers. For portfolio prototype work, either reduce the question set to stay within free tier limits, add Wait nodes between items to space out calls, or use providers with more generous free tiers. For serious production evaluation, budget $5 to $10 on a paid API tier to run a clean benchmark without rate limit interference.

---

## 1.8 Jina Web Scraping

**Problem: Jina content noise fills the character cap with useless content**

Jina pulls everything from the page including cookie banners, opt-out preferences, privacy policy text, navigation elements, and footer links. A 3,000 character cap fills with useless content instead of meaningful company information.

Resolution: Add a targeted regex sanitization Code node before the character cap. Strip common noise patterns (cookie text, privacy policy boilerplate, navigation elements). Add an empty content fallback for pages that return near-nothing after sanitization. This sanitization step must be explicitly specified -- Claude Code does not include it by default.

**Better alternative: No-credential Jina**

Jina can be accessed via a simple URL prefix (r.jina.ai/[url]) with no API credential required. This is simpler than setting up a Jina API credential and produces equivalent results for most use cases.

---

## 1.9 Deduplication and Re-engagement

**Pattern: Dedup vs re-engagement logic**

Hard deduplication via Compare Datasets alone permanently blocks previously processed companies or contacts, which is wrong for any real sales or outreach pipeline.

Resolution: Use Compare Datasets to identify existing records, then route them through an IF node that checks the Last Contacted timestamp. Records contacted within the suppression window are blocked. Records outside the window pass through for re-engagement. This gives the pipeline a memory without making it permanently blind to prior contacts.

---

## 1.10 Test Data Discipline

**Problem: Test data does not reflect the actual use case**

Testing a lead scoring pipeline with Google, Microsoft, or HubSpot produces all 10s -- every company scores as a perfect fit. IF routing never triggers the low-score branch. Pipeline problems are masked because the test data is wrong.

Resolution: Use test data that matches the actual target profile. For a consulting lead pipeline, that means small to mid-size businesses -- companies like Trainual, Jobber, Calendly, Copper CRM. Always include at least one genuinely poor-fit lead to confirm IF routing works end to end.

For governance pipelines, always include queries from every classification category -- STANDARD, SENSITIVE, and UNCERTAIN -- and at least one query from each escalation list category (legal, medical, HR, named individual). A question set that does not include UNCERTAIN or escalation-list queries cannot confirm that routing logic works for the cases that matter most.

---

## 1.11 Classification Prompt Design

**Problem: LLM routes ambiguous documents to the wrong category**

When a document contains elements of more than one category, the model defaults to the closest single match rather than recognizing ambiguity. Vague "unknown" category descriptions do not override the model's preference for a confident classification.

Resolution: Add explicit definitions for every category in the classification prompt -- not just the category name. Add an explicit trigger rule for the fallback/unknown category: "If the document contains elements of more than one category, return unknown." Define each named category narrowly enough that hybrid documents are excluded (e.g., "contract = services only, no investment component").

**Pattern: UNCERTAIN is a valid output, not a failure state**

When a query spans multiple sensitive categories, the model will force a single confident classification rather than flag the ambiguity unless explicitly instructed otherwise. A confident wrong classification is worse than a flagged uncertain one.

Resolution: Define UNCERTAIN explicitly in the classifier prompt as a valid and expected output with a clear trigger rule: "If the query or response spans multiple sensitive categories or intent cannot be confidently determined, output UNCERTAIN." Do not rely on the model to infer that ambiguity is acceptable. State it explicitly. UNCERTAIN always routes to human review and is never a reason to skip review.

**Pattern: Classify both query and response independently for governance workflows**

Classifying only the query misses responses that inadvertently contain sensitive information despite a benign-looking query. Classifying only the response misses sensitive intent even when the LLM appropriately deflects or refuses.

Resolution: Use two separate classifier nodes running in parallel. One receives the query text. One receives the response text. Each outputs an independent classification and domain label. The IF routing rule fires if EITHER classification is SENSITIVE or UNCERTAIN. This catches both the attempt (sensitive query) and the inadvertent leak (sensitive response) as distinct governance signals.

---

## 1.12 Environment and Editor

**Problem: n8n editor instability on Firefox**

The n8n editor repeatedly drops its websocket connection every few seconds on Firefox, making consistent work impossible.

Resolution: Use Chrome or Edge. n8n's frontend is Chromium-based and performs significantly better on Chromium browsers. Save frequently (Ctrl+S) as a habit regardless of browser.

---

## 1.13 Audit Log and Metadata Sourcing

**Pattern: Every audit log field must have an identified source before building**

Audit log schemas often include fields like timestamp, token counts, latency, and estimated cost. None of these can come from the LLM -- it has no clock, no token counter, and no awareness of its own response time. Asking the LLM to populate these fields produces hallucinated plausible-looking values.

Resolution: Before building any audit log workflow, map every column in the schema to its source:
- LLM fields: response text, classification label, domain label
- n8n fields: timestamp (DateTime node), routing destination (Edit Fields after IF node)
- API metadata fields: token counts (from provider response metadata if surfaced -- varies by provider)
- Calculated fields: estimated cost (Code node using token counts and model pricing)
- Latency: requires DateTime nodes before and after the LLM call with a Code node to calculate the delta

If a field cannot be sourced reliably, leave it blank in v1 and document it as a v2 addition. Do not ask the LLM to fill in fields it cannot know.

**Pattern: Pre-export PII and bug checklist for workflow JSON**

Before exporting any workflow JSON for public sharing or GitHub, check for the following:

- sendTo fields in Gmail nodes -- must be dynamic expressions, never hardcoded addresses
- Google Sheet IDs and URLs in documentId and sheetName fields
- Task List IDs in Google Tasks nodes
- OAuth2 credential IDs in all credentials blocks
- Webhook IDs in trigger and webhook nodes
- n8n instance ID in the meta block
- Workflow ID and versionId at the root level

Replace all literal values with clearly labeled placeholders (e.g. `YOUR_GOOGLE_SHEET_ID`). Run this check programmatically on every export. The Project 5 PII scan script is a reusable template for future builds.


---

# Section 2: Claude Code Patterns

---

## 2.1 Setup and Access

**Claude Code web version requires a GitHub connection**

The web version of Claude Code is not standalone. For a fully cloud-based workflow with no local install, connect via GitHub integration. The local CLI version (/web-setup) is required if not using GitHub.

**GitHub repo permissions are not automatic**

Repos created after the initial GitHub authorization are not automatically visible to Claude Code.

Resolution: Go to GitHub, Settings, Applications, find the Claude Code app, and manually add the new repo. Alternatively, click "Install the Claude GitHub app" in the Claude Code repo selector.

**Global CLAUDE.md requires local CLI install**

The global CLAUDE.md file (at ~/.claude/CLAUDE.md) only works with the local CLI install. When using the web version via GitHub, all context must live inside the repo itself.

Resolution: Always create CLAUDE.md inside the repo before starting a build session, not after.

---

## 2.2 CLAUDE.md -- The Most Important Habit

**CLAUDE.md is a forcing function**

The quality of Claude Code output is directly proportional to the quality of CLAUDE.md. Vague instructions produce architecturally correct but runtime-broken workflows. Specific instructions about batch processing, syntax rules, and wiring patterns produce workflows that import cleanly.

Every question Claude Code asks during a session costs tokens. CLAUDE.md eliminates the discovery phase entirely. Minimum content for any n8n project: credential names exactly as they appear in n8n, LLM provider and model, Google Sheet structure and column names, output targets, and lessons from prior builds.

**The most impactful CLAUDE.md additions define what the workflow is for**

The single biggest fix across Project 6 iterations was one sentence: "The workflow must process ALL rows in the input sheet in a single run. The dedup check is what prevents reprocessing -- not trigger filtering." This one sentence produced a fundamentally correct architecture on the first attempt in R2 after producing a fundamentally flawed architecture in R1 without it.

CLAUDE.md additions that define the purpose and batch behavior of the workflow matter more than additions that describe syntax rules -- though both are needed.

**Wiring instructions need to be diagram-level specific**

Abstract descriptions of wiring logic are not specific enough. Claude Code will repeat known wiring mistakes (like wiring Log to Summary from Gmail output) unless the instruction is explicit about exact node connections.

Resolution: Describe wiring as: "Route By Score True branch connects to BOTH Email Writer Chain AND Log to Summary. Gmail is a dead end -- nothing wires from it." Name the exact nodes and the exact connections.

**Keep n8n-specific rules in a dedicated n8n_SKILL.md**

Keeping n8n-specific syntax rules and node behavior in a separate skill file means every future project inherits all lessons automatically. CLAUDE.md stays lean and project-specific. n8n_SKILL.md grows with each build and compounds in value across the entire portfolio.

---

## 2.3 Token Limits and Output Management

**32,000 output token limit on Pro plan**

Claude Code on Pro hits a 32,000 output token ceiling. n8n workflow JSON is verbose -- even moderate workflows exceed this in a single response.

Resolution: Break large workflow builds into explicit parts and instruct Claude Code to stop and wait after each one. Example chunking: Part 1 is trigger and parallel LLM chains. Part 2 is Merge, Aggregate, and Edit Fields. Part 3 is output branches. Each part stays within the limit and Claude Code commits incrementally to the repo.

**Do not retry a failed token-limit response blindly**

A failed attempt still consumes tokens against the session limit. Chunk first, then build.

**Keep prompts lean**

Instructions like "double and triple check your work" dramatically inflate output size without improving quality. Keep prompts literal and specific. Save review requests for a dedicated follow-up prompt after the build is complete.

---

## 2.4 n8n-Specific Syntax Claude Code Gets Wrong

**Backtick template literals are not evaluated by n8n**

Claude Code writes LLM prompts using JavaScript backtick template literals with ${ } syntax. n8n does not evaluate these -- they are sent to the LLM as literal strings including the expression syntax itself.

Resolution: All LLM prompt expressions must use n8n {{ }} syntax. Never use backtick template literals with ${ } in n8n prompt fields.

**The {{ }} wrapper is required even in expression mode**

Even after switching from backtick syntax to string concatenation, prompts are still sent as literal strings if the entire expression is not wrapped in {{ }}.

Resolution: The entire prompt string must be wrapped in {{ }} in expression mode. Example: {{ 'static text ' + $json.company + ' more text' }}. This applies to string concatenation as well as direct field references.

**$input.first() only returns the first item**

Claude Code frequently uses $input.first().json in Code nodes, which always grabs the first item regardless of how many items are flowing. This is correct in runOnceForAllItems mode but wrong in runOnceForEachItem mode.

Resolution: In runOnceForEachItem mode, use $json directly (not $input.first()). In runOnceForAllItems mode, use $input.all() to access all items.

**Return syntax differs by Code node mode**

- runOnceForAllItems: return [{ json: {...} }] -- returns an array
- runOnceForEachItem: return { json: {...} } -- returns a single object

Mixing these causes "A json property isn't an object" errors. Claude Code uses array syntax throughout -- when switching a node to runOnceForEachItem mode, the return syntax must be updated.

**typeVersion mismatches on import**

Claude Code may generate nodes with typeVersion values that do not match the n8n Cloud instance. Symptom: "Install this node to use it -- invalid structure" on import.

Resolution: Specify the n8n Cloud version in CLAUDE.md. For Google Sheets Trigger specifically: use typeVersion 1, not typeVersion 4. For any mismatch, either ask Claude Code to downgrade the typeVersion or delete the broken node and recreate it manually in n8n.

**Node name changes break all cross-node references**

If a node is deleted and recreated with a different name (e.g., trigger renamed from "New Lead Added" to "Google Sheets Trigger"), every Code node referencing the original name breaks immediately.

Resolution: Either keep the original node name exactly as Claude Code generated it, or do a find-and-replace in the JSON before importing. Node names in cross-node references are case-sensitive and must match exactly.

**Gemini model version**

Claude Code may specify deprecated Gemini model versions (e.g., models/gemini-1.5-flash) that return errors on import.

Resolution: Use models/gemini-2.5-flash for current builds. gemini-2.0-flash is deprecated as of June 2026. Add the correct model string to CLAUDE.md and n8n_SKILL.md for all future builds.

---

## 2.5 Architecture and Design

**Do design work before opening Claude Code**

Claude Code is a builder, not a designer. Discovery conversations consume tokens with no output. Use Claude chat to finalize architecture and then bring a clean spec to Claude Code. Claude Code has a planning mode -- but it works best with a solid starting brief, not when doing discovery from scratch.

**Claude Code's tool choices are often better than manual build choices**

Despite import issues, Claude Code independently selects superior tools relative to what emerges from manual debugging. Examples from Project 6: Jina via r.jina.ai with no API credential required, Google News RSS instead of GNews (free, no key, no rate limits), Gemini Flash over Groq Llama models, Code node JSON.parse with regex fallback instead of Structured Output Parser, temperature tuning per node, Guard Email Body defensive node, appendOrUpdate instead of append only on log sheets. Trust these choices. The friction is n8n syntax, not design quality.

**Claude Code uses branches, not direct commits to main**

Claude Code creates its own branch and commits there. The full workflow JSON accumulates across commits -- each part builds on the last. To get the final file: switch to Claude Code's branch in GitHub, open the JSON, click Raw.

**The trigger node is configuration, not logic**

Claude Code uses placeholder values (REPLACE_WITH_INPUT_SHEET_ID) for Sheet IDs, Task List IDs, and email addresses. "No input connected" on a trigger node is expected behavior -- triggers fire on events and do not have upstream connections. Fill in all configuration values after import as the final step.

**Mandatory post-import checklist**

After importing any Claude Code-generated workflow into n8n:
1. Verify every IF node condition -- check that the left side is in expression mode and the operator is correct
2. Verify all LLM prompt fields are wrapped in {{ }} and expressions are evaluating
3. Check that Gmail is not wired to any logging node
4. Confirm typeVersion on the trigger node matches the n8n instance
5. Fill in all placeholder Sheet IDs, Task List IDs, and email addresses
6. Confirm credential names match exactly what is in n8n's credential manager
7. Verify the Merge node is set to Combine By Position, not Matching Fields, and that Include Any Unpaired Items is enabled

**CLAUDE.md additions required for Project 5 Claude Code build**

Before running the Claude Code benchmark build of Project 5, add the following to CLAUDE.md:

1. "The workflow must process ALL rows in the input sheet in a single run."
2. "Use the Limit node (maxItems: 1) after any multi-item branch that feeds a synchronization Merge node."
3. "Separate action items using --- as a delimiter. The downstream Code node splits on ---."
4. "The sendTo field in all Gmail nodes must use dynamic expressions. Never hardcode email addresses."
5. "Every Code node must open by pulling all needed fields off $json before referencing them."
6. "The Clean and Clean1 nodes are sanitization only. Do not include 30/60/90 split logic in these nodes."


---

# Section 3: Cross-Tool Patterns

These patterns apply regardless of whether the build is manual n8n or Claude Code.

---

**Prompt Tug-of-War**

Never mix JSON formatting instructions in the system prompt with a Structured Output Parser. The parser owns formatting. The prompt owns instructions. This conflict appeared in Projects 1, 3, and 6 and is the single most recurring issue across the entire portfolio.

**Data Threading**

Pass all necessary fields explicitly through every node. Never assume upstream data survives HTTP Request nodes, file processing nodes, or mode changes. Reattach fields immediately after any node that strips them. Use a Code or Edit Fields node to carry all fields forward, then reference via $json.fieldName downstream.

**Model First**

When structured output is failing or formatting is wrong, switch models before iterating on the prompt. The right model solves the problem. The wrong model fights every change. This applies equally to formatting compliance -- if a model ignores a plain text instruction, switch models before adjusting the prompt wording.

**Schema-First Design**

Define the parser schema before writing the prompt. The schema is the contract between the prompt and the downstream node. The prompt must never duplicate the schema's formatting instructions.

**Test Data Discipline**

Test data must reflect the actual use case. Wrong test data produces misleading results and masks real pipeline problems. Always include at least one edge case or poor-fit record to confirm routing logic works end to end.

**Identifier Threading**

In any pipeline processing multiple records or documents, establish a unique identifier (file_id, company_id, row index) at the start and thread it through every single node. File processing nodes strip it. HTTP Request nodes may lose it. Reattach it immediately whenever it goes missing. Merge nodes need it for key-based joining.

**Category Definition Discipline**

Every classification category needs an explicit definition and an explicit trigger rule -- not just a name. The fallback/unknown category needs the clearest trigger rule of all. Never rely on the model to infer ambiguity -- state the rule explicitly.

**Governance vs Performance: Model Selection Is Both**

For workflows processing sensitive content, model selection is a governance decision as much as a performance decision. A model that halluccinates confidential-looking data on sensitive queries provides false assurance, not actual governance. Evaluate models against a representative sensitive question set before committing to any model for a governance pipeline.


**Limit Node as Synchronization Reducer**

When a multi-item branch feeds a synchronization Merge node, the Limit node set to maxItems: 1 is the correct tool. Cleaner and more direct than Aggregate when the downstream node needs only a completion signal, not the collapsed data.

**PII Export Discipline**

Every workflow export for public sharing must pass a PII checklist before GitHub commit. Hardcoded emails, credential IDs, sheet IDs, and instance IDs are all exposures in a public repository. The pre-export checklist is as mandatory as the post-import checklist.

**CLAUDE.md Quality Determines Output Quality**

The gap between a Claude Code build that imports cleanly and one that requires hours of post-import fixes is almost entirely determined by CLAUDE.md quality. Every post-import fix that is documented and added to CLAUDE.md or n8n_SKILL.md is a fix that will not need to be made again.
