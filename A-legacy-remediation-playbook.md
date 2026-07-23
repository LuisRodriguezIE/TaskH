# Annex A — Legacy workspace remediation playbook

Companion to `powerbi-governance-framework.md`. Covers requirement 6: how to classify, clean, and resolve the ~4,000 existing workspaces.

---

## A1. Principle

**Classification is computed, not negotiated.** With 4,000 workspaces there is no version of this that works by asking people. Every workspace gets a disposition derived from measurable signals; humans only handle the exceptions and the appeals.

Three non-negotiable rules:

1. **Never delete without an archive step.** Archive = access removed, content intact, workspace visibly renamed. Objections surface during the archive window at zero risk.
2. **Snapshot permissions before any destructive action.** Restoring a workspace does **not** restore its access list. If you did not capture it, it is gone.
3. **Raise the deleted-workspace retention setting before wave 1.** Default is 7 days, adjustable to 90. Do this first.

---

## A2. Signal collection

Every workspace gets a row with these fields. All are obtainable from the admin APIs — see Annex B for the pipeline.

### Identity

| Field | Source | Notes |
|---|---|---|
| `workspace_id`, `name`, `description` | `GetGroupsAsAdmin` / Scanner | |
| `state` | `GetGroupsAsAdmin` | Active / Deleted / Removing / **Orphaned** |
| `type` | `GetGroupsAsAdmin` | Workspace vs **PersonalGroup** (My Workspace) |
| `capacity_id`, `is_on_dedicated_capacity` | `GetGroupsAsAdmin` | |
| `users[]` with `role` and `principal_type` | Scanner API | The permission snapshot. **Persist this.** |
| `created_date` | Scanner API | |

### Ownership (derived, requires Microsoft Graph)

| Field | Rule |
|---|---|
| `admin_users[]` | Users with role = Admin |
| `active_admin_count` | Count of admins whose Entra `accountEnabled` = true |
| `has_active_owner` | `active_admin_count > 0` |
| `owner_is_group` | Any admin is a security group rather than a user |

Note that the portal's `Orphaned` state is narrower than the real problem — it means no admin is assigned at all. A workspace whose only admin has a disabled Entra account still shows as `Active`. **`has_active_owner` is the field that matters**, and it only exists if you join against Graph.

### Content

| Field | Source |
|---|---|
| `item_count`, and counts by type (report, semanticModel, dataflow, lakehouse, notebook, …) | Scanner API |
| `has_certified_item`, `has_promoted_item` | Scanner API (endorsement) |
| `has_published_app` | Apps API / Scanner |
| `datasource_count`, `gateway_dependencies[]` | Scanner API |
| `sensitivity_labels[]` | Scanner API (read only; label changes require a user token, not a service principal) |

### Usage — requires accumulating Activity Events

| Field | Rule |
|---|---|
| `last_view_date`, `distinct_viewers_30d`, `distinct_viewers_90d` | `ViewReport`, `ViewDashboard` events |
| `last_edit_date` | `CreateReport`, `EditReport`, `PublishReport` events |
| `last_refresh_date`, `refresh_success_rate_30d` | `RefreshDataset` events |
| `is_used_90d` | `distinct_viewers_90d > 0 OR last_refresh_date within 90d` |

**Critical constraint:** the Activity Events API only serves a rolling ~30-day window, one day per request `[VERIFY current limits]`. You cannot retroactively obtain 90 days of history. **Start the daily activity capture on day one of the engagement** and let it accumulate while the rest of the assessment proceeds. If you wait until you "need" it, you add three months to the timeline.

If the engagement cannot wait 90 days, `last_refresh_date` and `last_edit_date` from the Scanner API give a usable, weaker proxy immediately.

### Risk

| Field | Why it matters |
|---|---|
| `has_publish_to_web_report` | Anonymous public URL. Highest-severity finding. |
| `external_user_count` | Guest users with access |
| `has_org_sharing_links` | Org-wide sharing links on items |
| `is_on_shared_capacity` | Cost/performance, not security |

### Derived classification inputs

```
has_content        = item_count > 0
has_active_owner   = active_admin_count > 0
is_used_90d        = distinct_viewers_90d > 0 OR last_refresh_date >= today-90
is_risky           = has_publish_to_web_report OR external_user_count > 0
is_business_critical = has_certified_item OR has_published_app
                       OR distinct_viewers_90d >= 25        [DECIDE threshold]
is_personal        = type == 'PersonalGroup'
owner_departed     = is_personal AND owner account disabled
```

---

## A3. Triage ladder

Tests run in order. Each workspace exits at the first match.

| # | Test | Disposition | Automatable |
|---|---|---|---|
| 0 | `is_risky` | **REMEDIATE** — disable publish-to-web, remove guest access, then re-enter the ladder | Detection yes, action semi |
| 1 | `has_content == false` | **DELETE** — 14-day notice to owner if one exists, then delete | Fully |
| 2 | `owner_departed` | **HARVEST** — review content, migrate what matters, then delete | Detection yes, review manual |
| 3 | `is_used_90d == false` | **ARCHIVE** — notify, 30-day claim window, archive, delete at +30 more | Fully |
| 4 | `has_active_owner == false` | **REASSIGN** — claim campaign; success → continue, failure → ARCHIVE | Detection yes, campaign manual |
| 5 | `is_business_critical` | **MIGRATE-CERTIFIED** — into the DAT/RPT structure | Manual, per workspace |
| 6 | otherwise | **MIGRATE-WORKGROUP** — into a WRK workspace, or CONSOLIDATE if duplicate | Semi |

### Ordering rationale

- **Risk first, unconditionally.** A publicly exposed report is not a cleanup item; it is an incident. It gets fixed before anyone discusses whether the workspace is still needed.
- **Empty before idle.** Empty workspaces need no judgement and no negotiation. They are the fastest reduction in the population and they build momentum.
- **Departed-owner personal workspaces before general idle.** These are the highest-probability location of orphaned business-critical content, and they are also where compliance exposure hides. Handle them deliberately, not as part of a bulk sweep.
- **Idle before unowned.** An unowned but heavily used workspace is a real problem needing a real owner. An unowned and unused workspace is just noise. Removing the noise first shrinks the campaign to a size people will actually engage with.

### Overrides

Two flags stop the ladder regardless of position:

- `has_certified_item` → never auto-archive; route to steward review.
- Sensitivity label above a defined threshold → never auto-delete; route to security review.

---

## A4. Wave execution

| Wave | Population | Weeks | Owner | Gate to proceed |
|---|---|---|---|---|
| **0 — Secure** | `is_risky` | 1 | BI platform + security | All exposures closed |
| **1 — Empty** | `has_content == false` | 2 | Automated | Population reduction reported |
| **2 — Departed** | `owner_departed` | 3 | BI platform + HR + stewards | Harvest list resolved |
| **3 — Idle** | `is_used_90d == false` | 4 | Automated + steward exception handling | Claim window closed |
| **4 — Unowned** | `has_active_owner == false` | 4 | Stewards + comms | Campaign closed |
| **5 — Migrate** | Everything remaining | 8–12 | Per domain | Domain sign-off |

### Wave 0 — Secure (week 1)

1. Query for `has_publish_to_web_report` and `external_user_count > 0`.
2. Disable the publish-to-web tenant setting **tenant-wide**. This kills existing embeds too — communicate before, not after.
3. Produce a per-workspace list of guest users; route to security for approve-or-revoke.
4. Remove org-wide sharing links where not justified.

Wave 0 is also the fastest way to demonstrate value to a sponsor. It typically produces a finding that justifies the whole program in the first week.

### Wave 1 — Empty (weeks 2–3)

1. Filter `item_count == 0`.
2. Split: has an owner vs orphaned.
3. Owned → automated notice, 14 days, then delete. Orphaned → delete directly.
4. Snapshot the permission list before deletion.
5. Report the population reduction. This number is the sponsor's first proof point.

### Wave 2 — Departed owners (weeks 4–6)

1. Join personal workspaces against Graph `accountEnabled == false`.
2. For each, list items. Anything with a semantic model, a refresh history, or recent views goes on a **harvest list**.
3. Route harvest candidates to the relevant domain steward: keep and migrate, or discard.
4. Everything not on the harvest list: delete after snapshot.

Note: the personal workspace of a departed user surfaces as `Deleted` in the admin portal with content retained for 90 days, and an admin can restore it as a normal collaborative workspace. **That 90-day clock may already be running for people who left months ago.** Prioritize recent leavers; accept that older content may already be unrecoverable, and say so early rather than discovering it during the wave.

### Wave 3 — Idle (weeks 7–10)

1. Filter `is_used_90d == false AND has_content == true`.
2. Exclude `has_certified_item` → steward review queue.
3. Notify owners: 30-day claim window, explicit "reply to keep" instruction.
4. On expiry: remove all access, rename with `ZZ-ARCHIVED-<date>-` prefix, retain content.
5. Thirty days after archiving with no objection: soft-delete.

**Expect a very low response rate.** Design for that: silence is a valid signal, the archive state is reversible, and the rename makes the archive obvious to anyone who goes looking. Do not extend the window because responses are low — low response *is* the finding.

### Wave 4 — Unowned but used (weeks 11–14)

The genuinely hard wave. These workspaces are in use and nobody is accountable.

1. Filter `has_active_owner == false AND is_used_90d == true`.
2. Identify candidate owners from telemetry: most frequent editor, most frequent viewer, or the domain the data comes from.
3. Run a targeted claim campaign — named individual, direct approach, not a broadcast. Broadcasts do not work for this.
4. Escalate unclaimed workspaces to the domain steward, then to the sponsor.
5. Still unclaimed after escalation → archive, and let the archive surface the real owner.

**Add the BI platform group as Admin to every one of these immediately, before the campaign.** It costs nothing, resolves the orphan state, and guarantees the content is recoverable regardless of how the campaign goes.

### Wave 5 — Migrate (weeks 15+)

Domain by domain, not tenant-wide. Per domain:

1. Steward reviews the remaining workspaces and assigns archetype and target name.
2. Create target workspaces per the naming standard with groups pre-provisioned.
3. Split content: semantic models to `DAT`, reports to `RPT`, or consolidate into `WRK`.
4. Rebind reports to the shared models. This is the real work and the real cost.
5. Publish apps; move consumers off direct workspace access.
6. Run parallel for 2 weeks.
7. Archive the source workspace. Do not delete during the migration wave.

**Set a hard cutover date per domain.** Content not migrated by that date goes to archive, not to an indefinite exception list. Exception lists are how migrations stall at 60% forever.

---

## A5. Communications

Every wave needs a communication plan. Under-communicating is the most reliable way to turn a technically sound cleanup into a political problem.

| Timing | Message | Audience | Channel |
|---|---|---|---|
| Program start | What is happening, why, what to expect, where to object | All Power BI users | Email + intranet + a live session |
| Wave start | This wave targets X; if you own one you will hear directly | All users | Email |
| Per workspace | Your workspace `<name>` is affected; here is the deadline and the one action to take | Named owner | Automated email + Teams |
| Reminder | 7 days before deadline | Named owner | Automated |
| Action taken | What happened, how to recover it, who to contact | Owner + steward | Automated |
| Wave close | Results, population reduction, what is next | Sponsor + stewards | Report |

Notification template principles: name the workspace explicitly, state exactly one required action, give a hard date, give a named human contact, and state plainly that content is recoverable. Anything longer will not be read.

---

## A6. Reporting the remediation

A single tracking view, refreshed daily, showing:

- Population by disposition class, trending over time
- Wave progress against plan
- Claim campaign response rate
- Exceptions and appeals, and their age
- Items harvested and their destination
- Deletions executed vs recoverable window remaining

The trend line is what the sponsor watches. Make the reduction visible weekly from wave 1 onward — momentum is what carries this program through wave 5.

---

## A7. What to expect

Two calibration points to set expectations honestly rather than optimistically:

- A tenant with open workspace creation and no lifecycle process accumulates a **large** proportion of empty, single-user, and dormant workspaces. The share is unknown until you scan, but it is reasonable to plan capacity for waves 1–3 removing the majority of the 4,000, and to be pleasantly surprised rather than under-resourced. Do not commit to a number before the scan.
- Wave 5 cost scales with **content complexity, not workspace count**. Fifty workspaces containing interconnected models cost more than 500 containing one report each. The wave 1–4 reduction is what makes wave 5 estimable at all — which is the argument for doing them in this order.
