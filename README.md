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
