# Databricks Genie Prompt — Copilot Studio Data Pipeline via SPN

---

## PASTE THIS INTO DATABRICKS GENIE

---

Build a production-grade Databricks pipeline in Python that uses a Service Principal (SPN) with client credentials flow to pull all available Copilot Studio agent data across four API planes: Azure ARM (Application Insights), Microsoft Graph API, Dataverse Web API, and Power Platform REST API.

### ENVIRONMENT AND SECRETS

All secrets live in Databricks Secret Scope `"kv-nfcu-prod"`. Retrieve them using:
```python
dbutils.secrets.get(scope="kv-nfcu-prod", key="<key-name>")
```

Secret key names to use:
- `spn-tenant-id`          → Entra ID tenant ID
- `spn-client-id`          → App registration client ID
- `spn-client-secret`      → App registration client secret
- `dataverse-env-url`      → e.g. `https://nfcuorg.crm.dynamics.com`
- `appinsights-app-id`     → Application Insights application (instrumentation) ID
- `appinsights-resource-id`→ Full ARM resource ID for the App Insights resource

Install required packages at the top of the notebook:
```
%pip install msal requests
```

---

### AUTHENTICATION — MULTI-SCOPE TOKEN MANAGER

Build a reusable `TokenManager` class that acquires and caches tokens for each resource scope independently using MSAL ConfidentialClientApplication. Token scopes needed:

| Resource                     | Scope                                            |
|------------------------------|--------------------------------------------------|
| Azure ARM (App Insights KQL) | `https://management.azure.com/.default`          |
| Application Insights API     | `https://api.applicationinsights.io/.default`    |
| Microsoft Graph              | `https://graph.microsoft.com/.default`           |
| Power Platform Admin         | `https://api.powerplatform.com/.default`         |
| Dataverse (per env)          | `{dataverse_env_url}/.default`                   |

Cache tokens in a dict by scope and refresh if within 60 seconds of expiry. Log a warning (do not raise) if any individual token acquisition fails — the pipeline should continue with the remaining scopes.

---

### PIPELINE MODULE 1 — APPLICATION INSIGHTS RUNTIME LOGS

This is the PRIMARY source for runtime conversation telemetry. Query the App Insights REST API at:
`POST https://api.applicationinsights.io/v1/apps/{app_id}/query`

Use token scope: `https://api.applicationinsights.io/.default`

Run the following KQL queries in sequence. Wrap each in try/except — if a query fails, log the error and move on.

**Query 1 — All conversation events (last 7 days by default, parameterize lookback_days):**
```kusto
customEvents
| where timestamp > ago({lookback_days}d)
| extend
    conversation_id    = tostring(customDimensions["conversationId"]),
    activity_id        = tostring(customDimensions["activityId"]),
    channel_id         = tostring(customDimensions["channel"]),
    user_id            = tostring(customDimensions["userId"]),
    user_message       = tostring(customDimensions["text"]),
    topic_name         = tostring(customDimensions["TopicName"]),
    topic_id           = tostring(customDimensions["TopicId"]),
    bot_name           = tostring(customDimensions["BotName"]),
    bot_id             = tostring(customDimensions["BotId"]),
    design_mode        = tostring(customDimensions["DesignMode"]),
    locale             = tostring(customDimensions["locale"]),
    from_name          = tostring(customDimensions["fromName"]),
    error_code         = tostring(customDimensions["ErrorCode"]),
    error_message      = tostring(customDimensions["ErrorMessage"])
| project timestamp, name, conversation_id, activity_id, channel_id,
          user_id, user_message, topic_name, topic_id, bot_name, bot_id,
          design_mode, locale, from_name, error_code, error_message,
          session_Id, operation_Id, cloud_RoleInstance
```

**Query 2 — Generative answer events with model, tokens, sources (key for LLM observability):**
```kusto
customEvents
| where timestamp > ago({lookback_days}d)
| where name == "GenerativeAnswers"
| extend cd = todynamic(customDimensions)
| extend
    conversation_id    = tostring(cd["conversationId"]),
    bot_id             = tostring(cd["BotId"]),
    topic_name         = tostring(cd["TopicName"]),
    user_question      = tostring(cd["text"]),
    generative_answer  = tostring(cd["Result"]),
    sources_used       = tostring(cd["Sources"]),
    search_queries     = tostring(cd["SearchQueries"]),
    model_name         = tostring(cd["ModelName"]),
    model_deployment   = tostring(cd["ModelDeployment"]),
    prompt_tokens      = toint(cd["PromptTokens"]),
    completion_tokens  = toint(cd["CompletionTokens"]),
    total_tokens       = toint(cd["TotalTokens"]),
    latency_ms         = todouble(cd["LatencyMs"]),
    confidence_score   = todouble(cd["ConfidenceScore"]),
    answer_type        = tostring(cd["AnswerType"]),
    channel_id         = tostring(cd["channel"]),
    user_id            = tostring(cd["userId"]),
    design_mode        = tostring(cd["DesignMode"])
| project timestamp, conversation_id, bot_id, topic_name, channel_id, user_id,
          user_question, generative_answer, sources_used, search_queries,
          model_name, model_deployment, prompt_tokens, completion_tokens,
          total_tokens, latency_ms, confidence_score, answer_type, design_mode
```

**Query 3 — Bot request/response pairs (input → output per turn):**
```kusto
customEvents
| where timestamp > ago({lookback_days}d)
| where name in ("BotMessageReceived", "BotMessageSent")
| extend cd = todynamic(customDimensions)
| extend
    conversation_id = tostring(cd["conversationId"]),
    activity_id     = tostring(cd["activityId"]),
    message_text    = tostring(cd["text"]),
    channel         = tostring(cd["channel"]),
    user_id         = tostring(cd["userId"]),
    from_name       = tostring(cd["fromName"]),
    bot_id          = tostring(cd["BotId"]),
    bot_name        = tostring(cd["BotName"]),
    is_design_mode  = tobool(cd["DesignMode"])
| project timestamp, name, conversation_id, activity_id,
          message_text, channel, user_id, from_name,
          bot_id, bot_name, is_design_mode, session_Id
| order by conversation_id asc, timestamp asc
```

**Query 4 — Exceptions and errors:**
```kusto
exceptions
| where timestamp > ago({lookback_days}d)
| extend bot_id = tostring(customDimensions["BotId"])
| project timestamp, type, outerMessage, innermostMessage,
          bot_id, operation_Id, session_Id, cloud_RoleName
```

**Query 5 — Performance / dependency calls (external API calls made by agent):**
```kusto
dependencies
| where timestamp > ago({lookback_days}d)
| where type in ("HTTP", "Azure AI")
| project timestamp, name, target, data, duration,
          success, resultCode, operation_Id, session_Id
```

After pulling all query results, normalize each into a list of dicts and create a Spark DataFrame for each. Write each as a Delta table at:
- `catalog.copilot_studio.appinsights_events`
- `catalog.copilot_studio.appinsights_generative_answers`
- `catalog.copilot_studio.appinsights_messages`
- `catalog.copilot_studio.appinsights_exceptions`
- `catalog.copilot_studio.appinsights_dependencies`

Use `MERGE INTO` (Delta MERGE) on `(conversation_id, activity_id, timestamp)` so the pipeline is idempotent and can run incrementally.

---

### PIPELINE MODULE 2 — DATAVERSE WEB API (AGENT DESIGN + TRANSCRIPTS)

Base URL: `{dataverse_env_url}/api/data/v9.2/`  
Token scope: `{dataverse_env_url}/.default`  
Headers: `OData-MaxVersion: 4.0`, `OData-Version: 4.0`, `Prefer: odata.include-annotations=*`

Handle OData paging — follow `@odata.nextLink` until exhausted for every query. Implement exponential backoff (3 retries, base 2s) on 429 and 503.

**Dataverse Query 1 — All agents (bots table):**
```
GET /bots?$select=botid,name,schemaname,createdon,publishedon,statecode,
    statuscode,synchronizationstatus,componentstate,solutionid,
    authenticationmode,iconbase64&$orderby=createdon desc
```
Expand to get the Entra app registration linked to each agent:
```
$expand=createdby($select=systemuserid,fullname,internalemailaddress)
```
Extract `applicationId` from `synchronizationstatus` field (JSON blob) — this links each Copilot Studio agent to its Entra ID app registration.

Write to Delta: `catalog.copilot_studio.dv_agents`

**Dataverse Query 2 — Agent components (full design):**
For each bot_id found above, query:
```
GET /botcomponents?$filter=_parentbotid_value eq {bot_id}
    &$select=botcomponentid,name,componenttype,content,description,
             modifiedon,createdon,statecode,statuscode,data,schemaname
    &$orderby=componenttype asc
```

Parse the `content` field (JSON/YAML Adaptive Dialog format). Extract from each topic's content:
- `triggerPhrasesContent.items[]` — trigger phrases array
- `dialog.actions[]` — dialog steps/nodes
- Any `SendMessage` nodes (bot output text)
- Any `Question` nodes (bot prompts to user)
- Any `BeginDialog` / `CallSkill` references
- Any HTTP request action targets (external APIs called)

Map `componenttype` integer to human-readable name:
0=Topic, 1=Skill, 2=BotVariable, 3=Entity, 4=Dialog, 5=Trigger,
6=LanguageUnderstanding, 7=QnA, 8=Attachment, 14=Action(Flow),
15=ConnectionReference, 16=EnvironmentVariable

Write to Delta:
- `catalog.copilot_studio.dv_agent_components`
- `catalog.copilot_studio.dv_agent_topics` (flattened, one row per topic)
- `catalog.copilot_studio.dv_agent_trigger_phrases` (flattened, one row per phrase)

**Dataverse Query 3 — Conversation transcripts (CRITICAL for input/output detail):**
```
GET /conversationtranscripts?$select=conversationtranscriptid,name,
    content,createdon,conversationstarttime,conversationendtime,
    _bot_conversationtranscriptid_value,schematype
    &$filter=createdon ge {lookback_start}&$orderby=createdon desc
```

The `content` field is a JSON blob — parse it to extract per-turn structure:
```json
{
  "activities": [
    {
      "type": "message",
      "from": { "id": "...", "role": "user|bot" },
      "text": "...",
      "timestamp": "...",
      "channelData": { "properties": { ... } }
    }
  ]
}
```

From each activity, extract:
- `turn_index` (position in conversation)
- `role` (user or bot)
- `message_text` (the actual input or output)
- `timestamp`
- `channel_data` (any metadata)
- `bot_id` (from `_bot_conversationtranscriptid_value`)

Write flattened turns to: `catalog.copilot_studio.dv_transcript_turns`

**Dataverse Query 4 — Session analytics:**
```
GET /msdyn_botsessions?$select=msdyn_botsessionid,msdyn_sessionid,
    msdyn_starttime,msdyn_endtime,msdyn_outcome,msdyn_escalated,
    msdyn_sessionturns,msdyn_engagementid,
    _msdyn_bot_value&$orderby=msdyn_starttime desc
```
Write to: `catalog.copilot_studio.dv_sessions`

---

### PIPELINE MODULE 3 — GRAPH API AGENT CATALOG (TRY WITH FALLBACK)

Token scope: `https://graph.microsoft.com/.default`

**Important known limitation**: This endpoint only supports delegated (user context) permissions. Application-only (SPN) auth returns HTTP 403 "User context is required." Attempt it anyway and capture the error metadata — useful for governance tracking.

```python
def try_graph_catalog(token_manager):
    result = {"status": None, "agents": [], "error": None, "error_detail": None}
    try:
        token = token_manager.get_token("https://graph.microsoft.com/.default")
        url = "https://graph.microsoft.com/beta/copilot/admin/catalog/packages"
        params = {"$filter": "supportedHosts/any(h:h eq 'Copilot')"}
        resp = requests.get(url, headers={"Authorization": f"Bearer {token}"},
                            params=params)
        if resp.status_code == 200:
            result["status"] = "success"
            result["agents"] = resp.json().get("value", [])
        elif resp.status_code == 403:
            result["status"] = "blocked_delegated_only"
            result["error_detail"] = resp.json()
        else:
            result["status"] = f"http_{resp.status_code}"
            result["error_detail"] = resp.text
    except Exception as e:
        result["status"] = "exception"
        result["error"] = str(e)
    return result
```

Also attempt Graph Entra service principals to list agent app registrations. Every Copilot Studio agent automatically gets an Entra ID app registration — these ARE accessible via SPN:
```
GET https://graph.microsoft.com/v1.0/servicePrincipals
    ?$filter=tags/any(t:t eq 'WindowsAzureActiveDirectoryIntegratedApp')
    &$select=id,appId,displayName,tags,createdDateTime,
             notes,description,servicePrincipalType
```
Cross-reference these against the `applicationId` values extracted from Dataverse `bots.synchronizationstatus` to identify which service principals belong to Copilot Studio agents.

Write to: `catalog.copilot_studio.graph_agent_catalog` (even if only partial/error metadata)

---

### PIPELINE MODULE 4 — POWER PLATFORM ADMIN API

Token scope: `https://api.powerplatform.com/.default`
Base URL: `https://api.powerplatform.com/copilotstudio/`
API version: `?api-version=2022-03-01-preview`

**Note**: Use the `copilotstudio` namespace — `powervirtualagents` is deprecated.

**Query 1 — List environments (to discover all Dataverse orgs):**
```
GET https://api.powerplatform.com/appmanagement/environments
    ?api-version=2022-03-01-preview
```
Extract `environmentId`, `displayName`, `domainName`, `instanceUrl` for each environment.

**Query 2 — Bot quarantine status (for each bot_id from Dataverse):**
```python
for env_id, bot_id in bot_env_pairs:
    GET /copilotstudio/environments/{env_id}/bots/{bot_id}/quarantineStatus
        ?api-version=2022-03-01-preview
```
Extract quarantine state, reason, last updated. Write to `catalog.copilot_studio.pp_bot_quarantine_status`

---

### PIPELINE MODULE 5 — CONSOLIDATION AND ENRICHMENT

After all modules complete, build a unified conversation view by joining:

```python
# Pseudo-schema for the unified table
unified_schema = {
    "conversation_id":     "string — from App Insights or transcript",
    "session_id":          "string — from msdyn_botsession",
    "bot_id":              "string — from bots table",
    "bot_name":            "string",
    "channel":             "string",
    "user_id":             "string (hash/anonymize if required by policy)",
    "turn_index":          "int",
    "turn_timestamp":      "timestamp",
    "input_text":          "string — user message",
    "output_text":         "string — agent response",
    "topic_triggered":     "string",
    "model_name":          "string — from GenerativeAnswers event",
    "model_deployment":    "string",
    "prompt_tokens":       "int",
    "completion_tokens":   "int",
    "total_tokens":        "int",
    "latency_ms":          "double",
    "sources_used":        "string (JSON array)",
    "confidence_score":    "double",
    "escalated":           "boolean — from msdyn_botsession",
    "outcome":             "string — resolved/escalated/abandoned",
    "design_mode":         "boolean — test vs production",
    "data_source":         "string — appinsights|dataverse_transcript",
    "pipeline_run_ts":     "timestamp — when this pipeline run executed"
}
```

Join key: `conversation_id`. For token / model data that only comes from App Insights `GenerativeAnswers` events, left-join onto the transcript turns by `(conversation_id, closest timestamp)` within a 5-second window.

Write the unified table to: `catalog.copilot_studio.unified_conversation_turns`

Add a summary aggregation table at `catalog.copilot_studio.daily_agent_summary`:
- Rows: one per (bot_id, date, channel)
- Columns: total_conversations, total_turns, total_tokens, avg_latency_ms, escalation_rate, top_topics (array), error_count

---

### ORCHESTRATION AND SCHEDULING

Structure the notebook as a Class `CopilotStudioPipeline` with methods:
- `run_full()` — runs all 5 modules
- `run_incremental(lookback_hours=24)` — runs only App Insights + transcript delta
- `run_design_scan()` — runs only Dataverse agent design modules

Add a `PipelineRunLog` Delta table at `catalog.copilot_studio.pipeline_run_log` that captures:
- `run_id` (UUID), `run_type`, `start_ts`, `end_ts`, `status`, `rows_written` per table, `errors` (JSON array)

At the top of the pipeline, read Databricks widgets:
```python
dbutils.widgets.text("lookback_days", "7")
dbutils.widgets.text("run_type", "incremental")  # full | incremental | design_scan
dbutils.widgets.text("target_catalog", "catalog")
dbutils.widgets.text("target_schema", "copilot_studio")
```

---

### ERROR HANDLING REQUIREMENTS

1. Each API module must run independently — failure in one must not stop others
2. HTTP 429 (rate limit) → exponential backoff, max 3 retries, then log and skip
3. HTTP 403 on Graph catalog → log as expected, continue
4. Dataverse `content` parse failure → store raw JSON string, log a warning, continue
5. Missing fields in App Insights customDimensions → use `tostring(cd.get("field", ""))` with defaults
6. All exceptions must be caught, serialized to `pipeline_run_log.errors`, and surfaced as a final summary cell

---

### OUTPUT SUMMARY

At the end of the notebook, print a formatted summary showing:
- Tables written and row counts
- Token usage totals found (sum across all GenerativeAnswers events)
- Unique agents found across all sources
- Unique conversations in the window
- Any modules that failed with reason

Use `display()` and `spark.sql()` to show sample rows from each Delta table for validation.

---

### NOTES FOR GENIE

- Write all code as a single well-organized Databricks notebook with `# COMMAND ----------` cell separators
- Use `pyspark.sql.functions` for DataFrame transformations, not pandas (data could be large)
- Prefer Delta MERGE (upsert) over overwrite to support incremental runs
- Use `spark.conf.set("spark.sql.caseSensitive", "false")` to handle mixed-case OData column names
- Log all HTTP calls with URL, status code, and elapsed time using Python `logging` at INFO level
- Do not hardcode any credentials — all via `dbutils.secrets.get()`
- Mark any cell that does external HTTP calls with `# EXTERNAL CALL — requires network egress` comment
- The pipeline should be runnable both interactively and as a Databricks Job
