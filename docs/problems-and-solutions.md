# Problems & Solutions

Real issues from building and iterating the ShireenAI agent engine, each with its root cause and fix. Status: ✅ fixed · ⚠️ fixed with a caveat · 🔁 tried, then rolled back.

## A. Retrieval (RAG) and Author DNA

| # | Problem | Root cause | Fix | Status |
|---|---|---|---|---|
| A1 | Retrieval returned nothing even though data existed | `user_id` mismatch: the web app sent it in a nested metadata field, and some triggers had no user at all | Resolve `user_id` from every known payload location, inject it into the author profile that the retrieval sub-workflow reads, and fall back to an explicit "all" scope | ✅ |
| A2 | Style-chunk inserts failed, so retrieval silently ran on nothing | **Vector dimension mismatch**: column was `vector(1024)`, the embedding pipeline produced 768 | Altered the column to `vector(768)` and pinned every embedding call to 768 dimensions | ✅ |
| A3 | SQL inserts failed with no useful error | Hand-built SQL: empty embeddings became `[]`, keywords were unquoted, IDs could be empty | Safe SQL builder: empty embedding → `NULL`, quoted keywords, UUID fallback | ✅ |
| A4 | Writing profile never reached agent memory | The save step read the previous database node's output instead of the profile data | Read explicitly from the node that built the profile | ✅ |
| A5 | Questionnaire-based and upload-based style context got mixed up or lost | Both sent as one blob | Pass them as separate fields, with a flag computed from the original request | ✅ |
| A6 | Agent used a default profile even after the user uploaded samples | Wrong priority between profile sources | Prefer the profile returned by the style sub-workflow | ✅ |
| A7 | Old profiles were stored with placeholder values | Created before full extraction existed | Heuristic rebuild from stored style chunks (no LLM), written back once | ⚠️ disabled after moving to direct database access |
| A8 | Per-chunk agent output failed to parse | Model returned JSON with trailing commas or wrapped in markdown | Tolerant multi-pass parser | ✅ |
| A9 | Uploaded files sometimes unreadable | Inconsistent input: base64 with data-URI prefix vs binary under different field names | Normalisation step and router: strip prefix → convert → force a single binary field | ✅ |
| A10 | Upload requests timed out | Chunking, embedding, and analysis take minutes | Acknowledge immediately and continue processing asynchronously | ✅ |
| A11 | Merging style and reference results corrupted items | A merge node in the middle of the flow broke item pairing | Moved retrieval into sub-workflows with a single merge point before context assembly | ✅ |

## B. Cascading API errors (HTTP 422 / 503)

| # | Problem | Root cause | Fix | Status |
|---|---|---|---|---|
| B1 | Callbacks rejected with 422 because `book_id` was empty | The LLM agent node does not reliably forward every input field, so IDs were lost after the agent step | Every downstream step re-reads `book_id` and `job_id` from the original trigger (with layered try/catch fallbacks) instead of trusting the agent's output | ✅ |
| B2 | Web app could not match replies to requests | Callbacks had no `job_id` | Added `job_id` to every callback, including error replies | ✅ |
| B3 | 422 because `content` was empty | Backend validation rejects empty strings; a failed generation produced `""` | Guard before the callback: empty content is replaced with a clear "please try again" message | ✅ |
| B4 | Malformed chat-callback body | `JSON.stringify` inside a JSON-body expression produced a double-encoded string | Send a plain object / body parameters | ✅ |
| B5 | Intermittent 503s from model APIs | Provider overload and rate limits | Retries with backoff on HTTP steps; primary model with automatic fallback model | ✅ |
| B6 | One bad step broke every step after it | Invalid agent output flowed straight into the callbacks | **Validate output at the boundary**: invalid results go to an error handler that sends a user-facing fallback message | ✅ |

## C. Agent behaviour

| # | Problem | Root cause | Fix | Status |
|---|---|---|---|---|
| C1 | A "proactive" feature caused a wave of 422s | The prompt made a follow-up question **mandatory**, and the model broke its JSON structure to satisfy it | Downgraded it to a preference with an explicit rule (*JSON validity > follow-up question*); empty values allowed; post-processing wrapped in try/catch | ✅ |
| C2 | Research results shown raw with labels (`research_query:`, `definition:` …) | Agent returned the tool's object; post-processing flattened it into "Label: value" text | Prompt rule against returning raw tool fields, plus extraction of known text fields only | ✅ |
| C3 | Output format changed whenever the agent used a tool | No rule on integrating tool results | "Tool results are raw material; the final answer always follows the active action's JSON format" | ✅ |
| C4 | Agent jumped phases | No guard | Pre-action validation (valid action, phase fit, profile present, blueprint approved) and a planning step | ✅ |
| C5 | No path for "summarise my uploaded references" | No dedicated action | Experimental `SUMMARIZE_RAG` action with keyword detection | 🔁 rolled back to keep the stable baseline |
| C6 | Token usage ballooned | Raw profile and long knowledge sent in full | Trimmed the writing sample, capped knowledge length, shortened the memory window | 🔁 rolled back together with C5 |

## D. Workflow runtime

| # | Problem | Root cause | Fix | Status |
|---|---|---|---|---|
| D1 | Helper functions unavailable in code steps | New isolated task runner | Native `fetch()` with parallel `Promise.all`, later replaced by database nodes | ✅ |
| D2 | "Node hasn't been executed" errors | Item-pairing lineage breaks after a merge, so referencing earlier nodes throws | Wrapped cross-node references in try/catch | ✅ |
| D3 | Flow stopped silently when a query returned no rows | A node with no output ends its branch | "Always output data" on the chain from context assembly to the callbacks | ✅ |
| D4 | Agent sometimes received an almost-empty prompt | The long inline prompt expression throws on unexpected input, and the fallback sends only IDs | Hardened with optional chaining and a try/catch fallback that records the error | ⚠️ **open**: the fallback still triggers for some requests; next step is to move prompt building into a dedicated, testable step |
| D5 | Output validation took the wrong branch | Strict type validation | Loosened case and type validation | ✅ |

## E. Infrastructure and migration

| # | Problem | Root cause | Fix | Status |
|---|---|---|---|---|
| E1 | Model credentials failed after moving to a new server | Workflow exports don't carry credential secrets | Switched the model provider to OpenRouter, keeping the same Gemini models | ✅ |
| E2 | Embeddings via the new provider were assumed impossible | Outdated information | Verified the OpenAI-compatible embeddings endpoint returns exactly 768 dimensions | ✅ |
| E3 | Embeddings would have become silently empty after the switch | Consumers read the old response shape (`embedding.values`) | Dual-format parsing (`data[0].embedding` first, old shape as fallback) everywhere | ✅ |
| E4 | Research tool depended on a provider-specific search feature | Vendor-specific tool node | Replaced with an HTTP tool calling an online-search model | ✅ |
| E5 | Database password rejected | The public port belonged to the hosting platform's own Postgres, not the application's | Exposed the application database on a dedicated port | ✅ |
| E6 | TLS connection failed | Server does not support SSL | Configured the client with SSL disabled (internal network) | ✅ |
| E7 | Database service restarts kept timing out | Image pulls timed out after the build cache was cleared | Fresh initialisation of the (empty) dev database | ✅ |
| E8 | "Permission denied for table/schema" | Application role lacked INSERT/CREATE | Granted the required privileges on the application schema | ✅ |
| E9 | Code steps called the database REST API with a service key | Early design | Replaced with database nodes using managed credentials | ✅ |
| E10 | Sub-workflow calls pointed to old IDs after import | IDs change on every import | Re-linked them, and added the step to the import checklist | ✅ |
| E11 | Secrets hard-coded in node headers leaked into every export | Early shortcut | Moved to managed credentials; old keys flagged for rotation; exports sanitised before commit | ✅ |
