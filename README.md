# 🍛 Spice Route Kitchen

Cloud kitchen website for Indian Breakfast & Chaat — ordering for pickup and catering.

## Features
- Full interactive menu with 18+ items
- Shopping cart with quantity controls & tax calculation
- Pickup & Catering order types
- Catering inquiry form with package options
- Responsive design (mobile + desktop)
- No backend required — static HTML

## Live Site
🔗 [Visit Website](https://umeshpjr747.github.io/spice-route-kitchen/)

## Deploy
This site is deployed via GitHub Pages from the `main` branch.
# Temp-practice


PBI 1: Initial Discussion with DevOps Engineer on Azure Resource Landscape
Description: Conduct a working session with the DevOps Engineer to understand the current Azure spoke deployment landscape, CI/CD pipelines, and resources touched during releases. Identify documentation gaps and confirm resource ownership.
Acceptance Criteria:

Meeting held with DevOps Engineer
Notes captured covering: pipelines in use, resources deployed/managed, known shared dependencies, current pain points
List of follow-up questions and stakeholders identified
Notes shared with team and stored in team documentation space

Dependencies: DevOps Engineer availability

PBI 2: Inventory All Azure Resources in Use
Description: Enumerate every Azure resource the team uses or depends on across all environments — ADF, Logic Apps, Function Apps, Event Hubs, Databricks workspaces, plus supporting resources (storage accounts, key vaults, networking, etc.).
Acceptance Criteria:

Complete list of resources captured with: resource name, type, subscription ID, resource group, region, full Azure Resource ID, environment (Prod/Non-Prod/Dev), shared vs dedicated classification
Cross-checked against Azure portal / Azure Resource Graph queries
Reviewed with DevOps Engineer for completeness

Dependencies: PBI 1 (initial DevOps discussion), Azure portal access

PBI 3: Build Inventory Spreadsheet (Source of Truth)
Description: Consolidate the inventory into a structured spreadsheet that serves as both internal source of truth and the CMDB submission artifact.
Acceptance Criteria:

Spreadsheet created with columns for all resource attributes (from PBI 2) plus owner, support group, business criticality, usage notes
Stored in team's shared location with version control
Reviewed and approved by team lead

Dependencies: PBI 2 (inventory complete)

PBI 4: Create Resource Dependency / Relationship Graph
Description: Produce a visual diagram showing how Azure resources interact and depend on each other. This will drive CI relationships in CMDB and support impact analysis during changes.
Acceptance Criteria:

Diagram produced (Visio, draw.io, or Mermaid) showing all in-scope resources and their dependencies
Direction of dependencies clearly indicated (e.g., Function App → Event Hub → ADF → Databricks)
Reviewed with DevOps Engineer for accuracy
Stored alongside the inventory spreadsheet

Dependencies: PBI 3 (inventory spreadsheet)

PBI 5: Discussions with Platform Engineering Team on Shared Resources
Description: Engage Platform Engineering to confirm ownership of shared Azure resources and agree on how our team is represented against them in CMDB (used-by vs supported-by relationship). Avoid duplicate CI creation.
Acceptance Criteria:

Meeting held with Platform Engineering
Shared resource ownership confirmed in writing (email or ticket)
Agreement reached on CI linkage approach for each shared resource
Existing CMDB entries for shared resources identified

Dependencies: PBI 3 (inventory), Platform Engineering availability

PBI 6: Align with CMDB / ServiceNow Team on Process and CI Classes
Description: Engage the CMDB / ServiceNow administration team to confirm the registration process, whether Azure resources are auto-discovered via Service Graph Connector, and correct CI class mappings for each resource type.
Acceptance Criteria:

Meeting held with CMDB team
Discovery mechanism confirmed (auto via connector vs manual)
CI class mapping documented for each Azure resource type in our inventory
Catalog request path identified for Service and CI registration
Required attributes/fields per CI class documented

Dependencies: PBI 3 (inventory), CMDB team availability

PBI 7: Define Application Service in ServiceNow
Description: Define the Application Service (or Business Service) that our Azure resources will roll up under in ServiceNow.
Acceptance Criteria:

Service name, description, and scope agreed
Business criticality tier assigned
Business owner and technical owner identified and confirmed
Support / assignment group identified
Sign-off obtained from team lead and business stakeholder

Dependencies: PBI 5, PBI 6

PBI 8: Confirm Support Groups and Ownership Chain
Description: Validate the AD groups mapped in ServiceNow that will be used for support assignment, ownership, and escalation against our CIs.
Acceptance Criteria:

AD group names confirmed for L1/L2/L3 support
Managed By and Owned By values agreed
Groups verified as existing and correctly mapped in ServiceNow
Documented in inventory spreadsheet

Dependencies: PBI 7

PBI 9: Define Change Model and Identify Standard Change Candidates
Description: Document which ServiceNow change types (Standard, Normal, Emergency) apply to our team's activities, and identify routine activities that qualify as Standard Change templates.
Acceptance Criteria:

Change type usage documented per activity (e.g., ADF pipeline release → Standard, infra changes → Normal)
Candidate activities for Standard Change templates identified
Approval flow / CAB requirements understood per change type
Documentation reviewed with DevOps Engineer

Dependencies: PBI 6

PBI 10: Submit Catalog Request(s) in ServiceNow
Description: Raise the formal catalog request(s) in ServiceNow for new Application Service registration and CI creation/linking, attaching the inventory and dependency graph.
Acceptance Criteria:

Catalog request(s) submitted with inventory spreadsheet and relationship graph attached
All required fields completed (service definition, CI list, ownership, support groups)
Request tracked through to fulfillment
Any clarifications from CMDB team addressed promptly

Dependencies: PBIs 3, 4, 7, 8

PBI 11: Validate End-to-End with Test Change Ticket
Description: Verify the Application Service and CIs are correctly configured in CMDB and the change workflow operates as expected by raising a test Change ticket.
Acceptance Criteria:

All CIs visible in CMDB with correct attributes
Relationships correctly reflected in CMDB
Test Change ticket raised against a CI
Approval flow, support group assignment, and notifications work as expected
Test ticket closed successfully
Final documentation published to team's shared location

Dependencies: PBI 10

# Prompt for Databricks Assistant / Genie

Copy everything below into the Databricks Assistant (or a new notebook prompt) to generate the job.

---

Build a Databricks job (PySpark, Python 3.10+, Unity Catalog enabled) that evaluates Azure AI-related security controls against live Azure resource metadata and writes the results to a Delta table. Follow this spec exactly.

## 1. Authentication

- Authenticate to Azure using a service principal (SPN) with Reader-only access at the subscription/management-group scope.
- Do not hardcode credentials. Read `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, and `AZURE_CLIENT_SECRET` from a Databricks secret scope named `azure-compliance-scope`, which is backed by an Azure Key Vault.
- Use `azure-identity`'s `ClientSecretCredential` to build the credential object, then pass it into the SDK clients below.
- Install/import: `azure-identity`, `azure-mgmt-resourcegraph`, `azure-mgmt-resource`, `requests` (for the Policy Insights REST call), `pyspark.sql`.

## 2. Resource enumeration

- Use `azure-mgmt-resourcegraph` (`ResourceGraphClient`) to run KQL queries against `Resources` at subscription scope, paginating through `$skipToken` until exhausted.
- Pull at minimum: `id`, `type`, `name`, `resourceGroup`, `subscriptionId`, `location`, `tags`, `kind` (where applicable), and the full `properties` blob for each resource type listed in the control mapping table (Section 4).
- Only query the resource types actually referenced by a control — don't do a blanket `Resources` pull with no type filter.

## 3. Policy compliance state

- For controls whose "Azure Data Source" is Policy Insights, call the Policy Insights REST API (`https://management.azure.com/{scope}/providers/Microsoft.PolicyInsights/policyStates/latest/queryResults?api-version=2019-10-01`) using the same SPN credential (get a bearer token via `credential.get_token("https://management.azure.com/.default")`).
- Parse `policyAssignmentId`, `policyDefinitionId`, `resourceId`, and `complianceState` per result.

## 4. Control mapping table

Load the control-to-metadata mapping from the attached reference (`AI_Control_Azure_Metadata_Mapping.xlsx`, sheet `AI_Control_Metadata_Mapping`) as the source of truth — do not hardcode control logic inline. Read it into a small Delta/config table with columns: `control_id`, `domain`, `control_name`, `resource_types`, `data_source` (`resource_graph` | `policy_insights` | `process_control`), `metadata_property`, `compliant_condition`.

For each control where `data_source = resource_graph`: write a small evaluator function that takes a resource's `properties` dict and the `metadata_property`/`compliant_condition` for that control, and returns `(compliant: bool, reason: str)`. Examples to implement first (these have unambiguous single-property checks):

- `F_25` — TLS: compliant if `properties.minimalTlsVersion >= "1.2"`.
- `F_27` — HTTPS-only: compliant if `properties.httpsOnly == true`.
- `F_38` / `F_41` — Zone redundancy: compliant if `properties.zoneRedundant == true`.
- `F_39` — Key Vault purge protection: compliant if `properties.enablePurgeProtection == true`.
- `F_44` / `F_45` / `F_46` — Event Hub network rules: compliant per the conditions in the mapping table.
- `F_03` / `F_13` — Local auth disabled: compliant if `properties.disableLocalAuth == true` or the Storage `allowSharedKeyAccess == false` equivalent (branch by resource type).

For controls where `data_source = policy_insights` (e.g. `F_22`, `F_48`, `F_49`): join the resource's `id` against the Policy Insights results from Section 3 and map `complianceState` directly to the output.

For controls where `data_source = process_control` (e.g. `F_28`, `F_29`, `F_31`, `F_32`, `F_34`): do not attempt automated evaluation. Write one row per resource-in-scope (or a single tenant-level row if there's no resource) with `compliant = NULL` and `reason = "process control — not evaluable from resource metadata, requires attestation"`.

## 5. Output schema

Write results to a Unity Catalog Delta table `main.ai_compliance.control_evaluations` (create if not exists) with this schema:

```
resource_id         STRING
resource_type       STRING
subscription_id     STRING
control_id          STRING
domain              STRING
control_name        STRING
data_source         STRING
compliant           BOOLEAN   -- nullable for process controls
reason              STRING
evaluated_at         TIMESTAMP
```

Use `MERGE INTO` keyed on `(resource_id, control_id)` so re-runs update existing rows rather than duplicating them.

## 6. Job structure

- Package as a Databricks Job with a single task running this notebook/script on a job cluster (not all-purpose, no autoscaling needed above 2 workers — this is metadata volume, not big data).
- Parameterize the list of target subscription IDs as a job parameter/widget, not hardcoded.
- Schedule: daily, off business hours.
- On failure, log the exception and the last successfully processed resource type, and fail the job (don't swallow errors silently) so job alerting fires.
- Add basic logging (`print`/`logging`) at the start/end of each phase: auth, resource graph pull, policy insights pull, evaluation, write.

## 7. What not to do

- Don't request or use any role beyond Reader for the SPN.
- Don't write secrets, tokens, or the raw `properties` blob (which can be large/sensitive) into notebook output or job logs — log only counts and resource IDs.
- Don't build a dashboard in this job — that's a separate Databricks SQL/Power BI task on top of the Delta table.

## 8. Deliverable

Output a single Python notebook (or `.py` job file) implementing the above, plus a short markdown comment block at the top summarizing the auth flow and the three data-source paths (Resource Graph / Policy Insights / process control).
