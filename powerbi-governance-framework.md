# Power BI / Fabric workspace governance framework

**Baseline v0.2 — expandable.** Each section is written to be self-contained so it can be deepened independently without rewriting the rest.

| Annex | Covers | File |
|---|---|---|
| A | Legacy remediation — the 4,000 existing workspaces | `A-legacy-remediation-playbook.md` |
| B | Automation architecture — service principal, APIs, notebooks | `B-automation-architecture.md` |

Markers used throughout: `[VERIFY]` = must be confirmed with the customer or against current Microsoft docs. `[DECIDE]` = an open decision the customer must make.

---

## 0. Structure of the proposal *(requirement 8)*

The proposal to the customer should be assembled in this order. The order matters — it moves from evidence to decision to plan, so nothing is proposed before it is justified.

| # | Section | Purpose | Audience |
|---|---|---|---|
| 1 | Executive summary | The problem in one page, the ask, the outcome | Sponsor |
| 2 | Current state and findings | Evidence from the inventory. Numbers, not opinions. | Sponsor + IT |
| 3 | Risk assessment | What is exposed today and what it would cost | Sponsor + Security |
| 4 | Target operating model | Domains, archetypes, nomenclature, roles | IT + stewards |
| 5 | Administration model | Fabric admin, domain admin, workspace admin, delegation | IT |
| 6 | Legacy remediation plan | How the 4,000 get classified and resolved | Everyone |
| 7 | Intake and lifecycle methodology | How new workspaces get requested and retired | Stewards + users |
| 8 | Automation and monitoring | What runs unattended, what gets flagged | IT |
| 9 | Roadmap, effort and resources | Phases, duration, who is needed | Sponsor |
| 10 | Governance and RACI | Who decides, who executes, who is informed | Everyone |
| 11 | Success metrics | How we prove it worked | Sponsor |
| 12 | Annexes | Playbooks, catalogs, scripts, user guide | Practitioners |

**Two rules for building this deck.** Section 2 must contain real numbers from the tenant scan before any of sections 4–8 are presented; a governance model proposed without evidence gets argued with. And section 1 should be written last.

---

## 1. Current state *(requirement 1)*

### 1.1 What the customer has described

| Symptom | What it actually indicates |
|---|---|
| 4,000+ workspaces, mixed purposes | No creation control. Workspace creation is open to all users. |
| No clear administrator | No accountability layer between the tenant admin and 4,000 objects |
| No process to know what is still in use | No telemetry pipeline. Activity data is not being captured or retained. |
| Workspaces owned by departed users | No joiner-mover-leaver process touching Power BI |
| Roles, naming, conventions unclear | No standard existed at creation time, so none can be enforced retroactively without a migration |

These are five symptoms of one root cause: **the platform was adopted without an operating model.** State it that way to the sponsor. It reframes the work from "clean up Power BI" (a cost) to "install an operating model" (an investment), and it is also accurate.

### 1.2 What must be measured before designing

Do not present a target model before these numbers exist. They will change the design.

| Metric | Why it changes the design | Source |
|---|---|---|
| Workspaces by state: Active / Deleted / Removing / **Orphaned** | Orphaned means no admin assigned. This is the direct count of the ownership problem. | Admin portal → Workspaces; `GetGroupsAsAdmin` |
| Personal (My Workspace) vs collaborative split | Personal workspaces of departed users are a distinct remediation track | `GetGroupsAsAdmin` (type) |
| Workspaces with zero items | The cheapest wins. Likely a large share of the 4,000. | Scanner API |
| Workspaces with no activity in 90 days | Defines the archive candidate pool | Activity Events API |
| Workspaces whose only admins have disabled Entra accounts | The real orphan count, broader than the portal's "Orphaned" state | Scanner API + Microsoft Graph |
| Reports with publish-to-web enabled | Anonymous public URLs. Immediate risk, immediate fix. | Scanner API / admin portal |
| Workspaces with external (guest) users | Data exposure risk | Scanner API |
| Workspaces on capacity vs shared capacity | Cost and performance implications | `GetCapacitiesAsAdmin` |
| Current licensing: Pro / PPU / F SKU `[VERIFY]` | Whether free users can consume. Reshapes the app strategy entirely. | Admin portal |
| Current **deleted-workspace retention setting** `[VERIFY]` | **Default is 7 days.** Must be raised before any deletion campaign. See 1.3. | Admin portal → tenant settings |

### 1.3 Three findings to expect, and act on immediately

**a. Retention is probably set to the default.** Deleted workspaces have a configurable retention period, default 7 days, adjustable up to 90. Seven days is not enough of a safety net for a bulk cleanup. **Raise this to 90 days before deleting anything.** This is a one-click change and it is the difference between a recoverable mistake and an incident.

**b. Restoring a workspace does not restore its permissions.** If a workspace is deleted and later restored, the access list does not come back. Therefore the inventory snapshot — including the full user/role list per workspace — must be captured and stored **before** any deletion, not after. Annex B covers this.

**c. Workspaces cap at 1,000 items.** Relevant when consolidating many small workspaces into fewer larger ones. Do not design a consolidation target that approaches this.

---

## 2. Target operating model *(requirements 2, 9)*

### 2.1 The four-layer structure

```
Tenant          → guardrails, tenant settings, capacity        (Fabric Administrator)
  Domain        → business area grouping, delegated settings    (Domain admin / steward)
    Workspace   → the container, one archetype, one owner       (Workspace admin)
      Item      → report, semantic model, lakehouse             (Item permissions, RLS)
```

Each layer delegates downward. The tenant admin does not manage workspaces; the tenant admin manages the rules that constrain them. This is the answer to "who administers 4,000 workspaces": nobody should, and the model must make that unnecessary.

### 2.2 Domains

Fabric **domains** group workspaces by business area and are the delegation point. Recommended starting set — keep it to 6–10, aligned to how the business is actually organized `[DECIDE]`:

| Domain | Code |
|---|---|
| Finance | FIN |
| Commercial / Sales | COM |
| Supply chain | SCM |
| Manufacturing / Operations | OPS |
| Human resources | HRS |
| Technology | TEC |
| Platform (governance itself) | PLT |

Sub-domains can be added later. Do not start with sub-domains — a two-level structure that people understand beats a four-level structure that they route around.

**Why domains matter here specifically:** domain admins can be given delegated control over their workspaces and certain tenant setting overrides, without granting tenant-level Fabric Administrator. That is exactly the missing accountability layer.

### 2.3 Workspace archetypes

Five types. Every workspace must be exactly one. This is the single most important classification in the model, because everything else — naming, roles, lifecycle, monitoring thresholds — is derived from it.

| Code | Archetype | What lives here | Tier | Owner | Lifecycle |
|---|---|---|---|---|---|
| **DAT** | Data layer | Certified semantic models, dataflows, lakehouses, pipelines | Governed | BI platform team | Permanent, change-controlled |
| **RPT** | Report layer | Reports and apps consuming DAT models. **No semantic models.** | Governed | Business owner + BI team | Permanent, change-controlled |
| **WRK** | Workgroup | Managed self-service for one team. Own models allowed. | Self-service | Team lead | Annual recertification |
| **SBX** | Sandbox | Individual experimentation. No sharing permitted. | Self-service | Individual | **Expires at 90 days** |
| **PLT** | Platform | Governance monitoring, admin tooling, the inventory itself | Governed | BI platform team | Permanent |

**The DAT/RPT split is the load-bearing decision.** Separating semantic models from reports is what makes certification, reuse, and impact analysis possible. It is also the change users resist most, because it means a report developer can no longer keep the model in the same place as the report. Budget change-management effort for this specifically.

**The SBX expiry is the anti-sprawl mechanism.** Every user gets a sandbox on request, in 5 minutes, no approval. It dies at 90 days unless renewed. This is what makes it politically viable to close open workspace creation — you are not taking away self-service, you are giving it a legitimate, disposable home.

### 2.4 Environments

Apply DEV/TST/PRD **only to DAT and RPT workspaces holding business-critical content.** Applying it universally triples the workspace count and is the most common reason governance programs stall.

| Suffix | Meaning |
|---|---|
| `DEV` | Development, no business users |
| `TST` | User acceptance testing |
| `PRD` | Production, wired to a deployment pipeline |

Content that is not business-critical lives in a single PRD-only workspace. `[DECIDE]` — the definition of "business-critical" is the customer's to make; propose: feeds a regulatory or financial report, or has more than N distinct viewers.

### 2.5 Nomenclature

The naming standard must be **machine-parseable**, because compliance checking has to be automated. A convention that a human has to interpret cannot be enforced across 4,000 objects.

```
<DOMAIN>-<ARCHETYPE>-<Subject>[-<ENV>]

Regex:  ^(FIN|COM|SCM|OPS|HRS|TEC|PLT)-(DAT|RPT|WRK|SBX|PLT)-[A-Za-z0-9]{2,30}(-(DEV|TST|PRD))?$
```

| Example | Reads as |
|---|---|
| `FIN-DAT-Revenue-PRD` | Finance, data layer, revenue subject, production |
| `FIN-RPT-Revenue-PRD` | Finance, report layer, consuming the above |
| `SCM-WRK-Logistics` | Supply chain, workgroup, logistics team |
| `COM-SBX-jperez` | Commercial, sandbox, individual |
| `PLT-PLT-Governance` | Platform domain, platform archetype, the governance workspace itself |

Rules:
- Fixed segment order, hyphen delimiter, no spaces, no accented characters.
- Subject is PascalCase, no hyphens inside it (the parser splits on hyphens).
- The workspace **description** field is mandatory and holds the human-readable name, the business purpose, and the owner's name. Do not try to encode meaning in the workspace name beyond the four segments.
- Renaming an existing workspace does not break report URLs `[VERIFY against current behaviour before mass renaming]`, but it does break bookmarks and documentation. Communicate before renaming.

**The corresponding Entra group naming:**

```
PBI-WS-<WorkspaceName>-<ROLE>      role in {ADM, MBR, CTB, VWR}
PBI-APP-<Domain>-<AppName>-<Audience>

PBI-WS-FIN-DAT-Revenue-PRD-ADM
PBI-WS-FIN-RPT-Revenue-PRD-CTB
PBI-APP-FIN-Revenue-Executives
```

### 2.6 Role assignment by archetype *(requirement 9)*

Power BI has four fixed workspace roles. The design work is mapping them, not inventing them.

| Role | Key capability | Key limitation |
|---|---|---|
| Admin | Everything, including access management and deletion | — |
| Member | Publish, share, add others up to Member | Cannot delete the workspace |
| Contributor | Create and edit content | Cannot share or manage access |
| Viewer | Read published content; **RLS is enforced** | — |

**The critical nuance:** row-level security is enforced for Viewers only. Members and Contributors see all data. The number of Contributors in a workspace containing sensitive data is therefore a security decision, not a convenience decision.

Assignment matrix — every cell is a **group**, never a person:

| | DAT | RPT | WRK | SBX | PLT |
|---|---|---|---|---|---|
| **Admin** | BI platform group + domain steward | BI platform group + business owner | BI platform group + team lead | BI platform group + individual | BI platform group |
| **Member** | Domain steward | Business owner | Team lead | — | Platform engineers |
| **Contributor** | Data engineers | Report developers | Team members | — | — |
| **Viewer** | — (consumed via Build permission) | Exception only | Exception only | — | Governance council |
| **Consumption** | Build permission on the model | **Power BI app audience** | App or direct | None — sharing blocked | App |

**Two structural rules that solve the customer's stated problems:**

1. **The BI platform team group is Admin on every workspace, always.** This makes orphaning structurally impossible. A person leaving can never strand a workspace, because a group is always present. This one rule prevents the problem from recurring after remediation.

2. **A workspace never contains an individual — only groups.** Access review becomes group-membership review, which IT and HR already know how to run, and it can be automated. The only permitted exception is the SBX archetype, where the individual is the point.

### 2.7 Consumption is via apps, not Viewer

Do not use the Viewer role as the standard consumption channel. Use **Power BI apps with audiences**:

- Apps present curated content; workspace access exposes work in progress.
- One workspace can serve different report subsets to different audiences.
- Removing an audience group is one change; auditing hundreds of Viewer assignments is not.

Reserve Viewer for genuine exceptions.

---

## 3. Administration model — the Fabric admin relationship *(requirement 5)*

### 3.1 The delegation chain

| Level | Role | Granted via | Scope | Headcount |
|---|---|---|---|---|
| Tenant | **Fabric Administrator** | Microsoft Entra role | Tenant settings, capacities, domains, workspace lifecycle actions (restore/permanently delete), admin API enablement, access to any workspace | 2–3, PIM-elevated `[VERIFY Entra P2]` |
| Capacity | Capacity administrator | Assigned per capacity | Capacity settings; can override certain tenant settings for that capacity | 1–2 per capacity |
| Domain | Domain admin / contributor | Assigned per domain by Fabric admin | Assign workspaces to the domain, domain-level tenant setting overrides, domain branding | 1–2 per domain (the stewards) |
| Workspace | Workspace admin | Workspace role | One workspace | Group-based, per matrix in 2.6 |
| Automation | **Service principal** | Entra app + security group + tenant setting | Read-only admin APIs across the whole tenant; optionally update APIs | 1 dedicated SP |

### 3.2 What each level actually does day to day

**Fabric Administrator does not administer workspaces.** With 4,000 objects that is not a job, it is a fiction. The Fabric Administrator:
- sets and change-controls tenant settings
- creates domains and appoints domain admins
- manages capacity assignment
- executes the irreversible actions (permanent delete, restore) that nobody else can
- owns the enablement of the service principal

**Domain admins (the stewards) do the workspace-level accountability work:** approving new workspace requests in their domain, certifying content, running quarterly access reviews, and responding to dormancy flags for their workspaces.

**The service principal does the observation.** Every recurring read — inventory, activity, compliance checks — runs unattended under the SP. This is what makes tenant-wide oversight tractable at this scale, and it is covered in Annex B.

### 3.3 Why this specific split matters for this customer

The customer's stated problem is "there is no clear administrator." The instinct is to appoint one. That instinct is wrong at 4,000 workspaces — a single administrator becomes a bottleneck and then a rubber stamp.

The correct answer is: **one accountable tenant owner, delegated domain stewards, automated observation, and group-based workspace ownership that cannot be orphaned.** Present it in exactly that order; it pre-empts the "so who is the admin?" question.

### 3.4 Tenant settings baseline

The guardrails. Minimum defensible set:

| Setting | Target | Why |
|---|---|---|
| Create workspaces | Restricted to `PBI-Workspace-Creators` | The single highest-impact anti-sprawl control. **This is what stops 4,000 becoming 8,000.** |
| Deleted workspace retention | Raise from 7 days to 90 `[DECIDE 30–90]` | Safety net for the remediation campaign |
| Publish to web | Disabled tenant-wide | Anonymous public URLs; no legitimate internal use case |
| Export data / Export to Excel | Restricted to specific groups | Bulk exfiltration path |
| External sharing / guest access | Restricted | |
| Publish apps to entire organization | Restricted group | |
| Certification | Enabled, certifier group = stewards | Required for the endorsement model |
| Usage metrics for content creators | Enabled | Needed for adoption KPIs |
| **Service principals can access read-only admin APIs** | Enabled for the SP's security group | **Required for all automation** |
| **Enhance admin API responses with detailed metadata** | Enabled for the same group | Without this the scan returns shallow data |
| Enhance admin API responses with DAX and mashup expressions | Enabled `[DECIDE]` | Enables lineage and duplicate-model detection; exposes query text |
| Service principals can access admin APIs used for updates | Enabled only if write automation is approved | Separate setting from read-only |
| Fabric item creation | Controlled separately from Power BI `[DECIDE scope]` | Lets you govern Power BI without blocking or opening Fabric |

**Change control:** every tenant setting change gets a record — what changed, who approved, business justification, date. This log is what an auditor asks for, and it is also how you avoid silent regressions.

---

## 4. Intake methodology *(requirement 4)*

How a requirement becomes the right kind of workspace. The goal is that classification is **determined by the answers, not negotiated**.

### 4.1 The decision questions

Six questions, asked in a request form. The archetype falls out of the answers.

| # | Question | Options |
|---|---|---|
| 1 | Who will use the content? | Just me → **SBX** / My team → continue / Beyond my team → continue |
| 2 | Does it feed a regulatory, financial or executive report? | Yes → governed tier / No → continue |
| 3 | Will other teams build on this data? | Yes → needs a **DAT** workspace / No → continue |
| 4 | Does the data contain restricted or personal information? | Yes → governed tier, RLS required / No → **WRK** |
| 5 | Which business domain owns the data? | Pick from the domain list → sets the domain prefix |
| 6 | Who is the accountable owner? | A named individual, mandatory. No owner → no workspace. |

### 4.2 Resulting routing

| Answers | Archetype | Approval | SLA |
|---|---|---|---|
| Q1 = just me | `SBX` | None — automatic | 5 minutes, automated |
| Q1 = my team, Q2/Q3/Q4 = no | `WRK` | Domain steward | 48 hours |
| Q2 or Q3 or Q4 = yes | `DAT` + `RPT` pair | Domain steward + BI platform | 5 business days |
| Platform/governance tooling | `PLT` | BI platform lead | Case by case |

**The SLA commitments are not decoration.** Restricting workspace creation is only acceptable to users if the alternative is fast. If the 48-hour SLA is not met consistently, users will find workarounds and the sprawl returns. Track SLA compliance as a governance KPI, not just an IT metric.

### 4.3 What the request produces

A single request generates: the workspace with a compliant name, the four Entra groups, domain assignment, capacity assignment, description populated, BI platform group added as admin, owner recorded, and expiry date set for SBX. All of this is automatable — see Annex B.

---

## 5. Operations, monitoring and control standards *(requirements 3, 10, 11)*

### 5.1 Workspace lifecycle states

Every workspace moves through defined states with automated transitions and owner notification at each step.

| State | Trigger | Action | Notification |
|---|---|---|---|
| Requested | Form submitted | Approval routing | Requester |
| Provisioned | Approved | Created with groups, domain, description | Owner + steward |
| Active | In use | — | — |
| **Dormant** | No activity 60 days | Flagged in governance report | Owner |
| **Warning** | No activity 90 days | Owner must confirm retention | Owner + steward |
| **Archived** | No response by 120 days | Access removed, content retained, workspace renamed with `ZZ-ARCHIVED-` prefix | Owner + steward |
| **Deleted** | 150 days, no claim | Soft-deleted; recoverable within the retention window | Steward |

SBX workspaces run a compressed version: warning at 75 days, archived at 90.

**Never delete without an archive step.** The archive state — access removed, content intact, obviously renamed — surfaces anyone who actually needed it, at zero risk. Almost every genuine objection arrives during the archive window, not after deletion.

### 5.2 Control standards — what gets checked, how often

| Control | Frequency | Threshold | Action on breach |
|---|---|---|---|
| Naming compliance (regex) | Daily | 100% | Flag; auto-rename or open ticket |
| Workspaces with no active admin | Daily | **0** | Auto-add BI platform group; escalate to steward |
| Workspaces not assigned to a domain | Daily | 0 | Flag to steward |
| Individual (non-group) access assignments | Weekly | 0 outside SBX | Flag; convert to group |
| Workspaces dormant >60 / >90 days | Weekly | Reported | Lifecycle transition |
| SBX past expiry | Daily | 0 | Auto-archive |
| Reports with publish-to-web | Daily | **0** | Immediate disable, notify security |
| External/guest users in workspaces | Weekly | Approved list only | Review |
| Semantic model refresh failure rate | Daily | <5% per workspace | Notify owner |
| Certified content with a stale refresh | Daily | 0 | Notify steward |
| Duplicate semantic models (same source, same measures) | Monthly | Trending down | Consolidation candidate list |
| Capacity utilization / throttling | Daily | <80% sustained | Capacity review |
| Access recertification completion | Quarterly | 100% of governed workspaces | Escalate to sponsor |

### 5.3 Reporting cadence

| Output | Frequency | Audience |
|---|---|---|
| Automated compliance alerts | Real-time / daily | Owner, BI platform |
| Domain digest — dormant, non-compliant, access changes | Weekly | Domain stewards |
| Tenant governance report — KPIs, trend, exceptions | Monthly | Governance council |
| Access recertification campaign | Quarterly | Stewards + owners |
| Model review — is the framework still fit | Annual | Sponsor + council |

### 5.4 Optimization standards

Beyond compliance, the same telemetry supports optimization:

- **Capacity right-sizing** — identify workspaces driving throttling; move or optimize them.
- **Semantic model consolidation** — same source tables, overlapping measures, different owners. The single biggest cost and trust win, and only visible with DAX/mashup metadata enabled.
- **Refresh scheduling** — spread refreshes to flatten capacity peaks.
- **Dead content removal** — reports with zero views over 180 days inside otherwise-active workspaces.
- **Gateway consolidation** — orphaned or duplicated gateway data sources.

---

## 6. Legacy remediation *(requirement 6)*

Summarized here; full method in **Annex A**.

The approach is a **triage ladder**: each workspace is tested against ordered criteria and exits at the first match. Because every test runs against inventory data, the classification of all 4,000 is computed, not debated.

| Wave | Target population | Disposition | Duration |
|---|---|---|---|
| **0** | Security exposures — publish to web, guest users | Remediate immediately, regardless of anything else | Days |
| **1** | Empty workspaces (0 items) | Notify, then delete | 1–2 weeks |
| **2** | Personal workspaces of departed users | Harvest anything valuable, then delete | 2–3 weeks |
| **3** | Has content, no activity in 90 days | Archive with a 30-day claim window | 3–4 weeks |
| **4** | Active but no accountable owner | Claim campaign; reassign or archive | 3–4 weeks |
| **5** | Active and owned | Migrate into the target model, domain by domain | 8–12 weeks |

Waves 0–3 are largely automatable and should remove a substantial share of the 4,000 before any human negotiation begins. **Do not start with wave 5.** Migrating workspaces that should have been deleted is the most common way this kind of program burns its budget.

---

## 7. User guide *(requirement 7)*

A one-page guide for end users, to be published in the customer's intranet and linked from every notification. Draft content:

### What changed
Workspaces are now created through a request form instead of directly in Power BI. Every workspace has a type, a domain, a standard name, and a named owner.

### What you need to do

| If you want to... | Do this | How long |
|---|---|---|
| Experiment on your own | Request a **sandbox**. Approved automatically. | 5 minutes |
| Build something for your team | Request a **workgroup** workspace | 48 hours |
| Publish something the business depends on | Request a **certified** workspace pair | 5 business days |
| Get access to existing content | Request access to the **app**, not the workspace | 48 hours |
| Share a report with someone | Add them to the app audience — do not share the workspace | Immediate |

### Rules to know
1. **Your sandbox expires after 90 days.** You will be warned first. Renew it if you still need it.
2. **You cannot share from a sandbox.** If others need it, promote it to a workgroup.
3. **Certified content lives in two workspaces** — the data model in one, the reports in another. Your reports connect to the shared model instead of containing their own.
4. **If your workspace is unused for 60 days you will get a notice.** Reply to keep it. No reply by 120 days and it gets archived; nothing is deleted without warning.
5. **Access is granted through groups**, not to you individually. Request through the access portal, not by asking a colleague to add you.

### Where to go
- Request a workspace or access: `[link]`
- Your domain steward: `[table]`
- Naming standard and archetype guide: `[link]`
- Support: `[channel]`

---

## 8. Roadmap, effort and resources

| Phase | Weeks | Output | Difficulty |
|---|---|---|---|
| **0. Instrument** — service principal, inventory pipeline, first full scan | 3–4 | Complete tenant inventory; the numbers for the proposal | Medium — blocked on tenant settings |
| **1. Assess** — analyze, classify, quantify, risk register | 2 | Findings pack, disposition list for all 4,000 | Low |
| **2. Design** — target model, naming, roles, tenant baseline, intake | 3 | This framework, customer-specific and approved | Low technically, high politically |
| **3. Stabilize** — waves 0–2, restrict workspace creation, raise retention | 3–4 | Risk removed, dead weight gone, bleeding stopped | Medium |
| **4. Pilot** — one domain fully migrated to the new model | 4 | Proven migration runbook | Medium |
| **5. Remediate** — waves 3–4 across the tenant | 6–8 | Population reduced to what is real | Medium |
| **6. Migrate** — wave 5, domain by domain | 8–12 | Target model in place | **High — where programs die** |
| **7. Operate** — monitoring live, stewards trained, handover | 4 | Runbook, dashboards, recertification cycle | Low |

**Total: 8–11 months** for a tenant of this size. Phases 0–2 (**8–9 weeks**) are the current request and can be quoted independently — they produce the evidence and the design, and they de-risk everything after.

| Resource | Commitment |
|---|---|
| Governance lead (this role) | Full time, phases 0–2; 60% thereafter |
| Fabric Administrator | 20%, but must be responsive — blocks phase 0 |
| Entra ID administrator | 15% — groups, service principal, access packages |
| Data engineer | 50% in phase 0 for the inventory pipeline |
| Domain stewards (6–8) | 1 day each in design; 2–3 days each during their migration wave |
| Communications / change | 20% from phase 3 onward — underestimated more often than any other line |
| Executive sponsor | Decision authority at three gates: after Assess, after Design, before Migrate |

---

## 9. Risks

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Service principal not enabled in time | High | **Blocks everything** | Raise in week 1; it is a tenant setting plus an Entra group |
| Deleting something that mattered | Medium | High | Archive-before-delete, 90-day retention, permission snapshots before any action |
| Claim campaign gets no response | **High** | Medium | Silence is a valid answer — default to archive, and let the archive state surface real objections |
| Restricting workspace creation seen as a blocker | High | Medium | Ship the 5-minute sandbox and the 48-hour SLA *at the same time*, not after |
| DAT/RPT split rejected by report developers | Medium | High | Pilot it with one willing domain first; lead with the reuse benefit, not the rule |
| Scope creeps into full Fabric governance | Medium | Medium | Bound the scope explicitly in the SOW |
| Migration stalls at 60% | **High** | High | Hard cutover dates per domain; unmigrated content moves to archive, not to indefinite exception |
| Stewards appointed but not given time | High | High | Get the time commitment in writing from their managers, not from them |

**Feasibility: high technically, hard organizationally.** Nothing in this design requires a feature that does not exist. Every failure mode above is a people problem. Say this plainly in the proposal — it sets up the sponsorship conversation that determines whether this succeeds.

---

## 10. Success metrics

| Metric | Baseline | Target | When |
|---|---|---|---|
| Total workspaces | ~4,000 | `[set after wave 1 data]` | End of phase 5 |
| Workspaces with no active admin | `[measure]` | **0** | End of phase 3 |
| Naming compliance | ~0% | >95% | End of phase 6 |
| Workspaces assigned to a domain | 0% | 100% | End of phase 6 |
| Access granted via group vs individual | `[measure]` | >95% | End of phase 6 |
| Reports with publish-to-web | `[measure]` | **0** | End of phase 3 |
| Workspaces created outside the process | n/a | 0 per quarter | From phase 3 |
| Workspace request SLA met | n/a | >90% | Ongoing |
| Certified semantic models | `[measure]` | 100% of critical content | End of phase 6 |
| Duplicate semantic models | `[measure]` | −50% | End of phase 6 |
| Quarterly recertification completed | 0% | 100% | From phase 7 |

---

## 11. Open questions for the customer

1. Current licensing: Pro, PPU, or Fabric capacity SKU?
2. Who holds Fabric Administrator today, and will this engagement get that access or a delegated equivalent?
3. Can a service principal be created and enabled for read-only admin APIs? **This is the critical path item.**
4. Is Fabric (lakehouses, notebooks, pipelines) in scope, or Power BI only?
5. Are sensitivity labels / Purview in scope?
6. What is the current deleted-workspace retention setting?
7. Does a joiner-mover-leaver process exist that could be extended to Power BI?
8. Is there an existing data governance body, or does one need to be created?
9. Is the mandate design-only, or design plus execution?
10. Who is the executive sponsor, and do the proposed stewards report into a structure that will give them the time?

Questions 3 and 9 determine the shape of everything else. Settle those first.
