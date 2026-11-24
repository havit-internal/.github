# havit-internal/.github

Organization-wide defaults for repositories in the `havit-internal` organization.

GitHub treats a repo named exactly `.github` in an org specially: any repo that
does not define its own version of these files falls back to what lives here.
That gives the whole org consistent issue templates, PR templates, community
health files, and a canonical label set with zero per-repo work.

## What lives here

```
.github/
├── ISSUE_TEMPLATE/
│   ├── feature.yml          ← Outcome-level container; rolls up Stories
│   ├── bug_report.yml       ← "Something broken" in dev, staging, or production
│   ├── user_story.yml       ← A user-facing slice of work
│   ├── task.yml             ← Implementation slice — a piece of a Story
│   └── config.yml           ← Disables blank issues
├── pull_request_template.md ← Issues / Refs — see QA convention below
├── labels.yml               ← Source of truth for sev:*/meta labels (work type is an Issue Type, not a label; workflow status is GitHub's native issue status, not a label)
└── workflows/
    ├── qa-routing.yml          ← Reusable workflow — see "PR convention" below.
    ├── subissue-rollup.yml     ← Reusable workflow — child Task/Bug status rolls up to parent Story/Feature
    ├── label-sync.yml          ← Runs centrally — see "Label sync" below. CI still planned.
    ├── migrate-work-status.yml ← Runs centrally, by hand, once — see "Migrating from Work status" below
    ├── issue-status-sync.yml   ← Retired stub — native statuses replaced it; remove its wrappers
    └── pr-linked-status.yml    ← Retired stub — native statuses replaced it; remove its wrappers

workflow-templates/          ← Ready-made wrappers for the reusable workflows — see "Adding the wrappers to a repo"
├── qa-routing.yml            (+ .properties.json)
└── subissue-rollup.yml       (+ .properties.json)

plugins/
└── gh-issue-templates/      ← Claude Code plugin — see "Claude Code plugin" below
```

## Related org repos

| Repo | Role |
|------|------|
| `havit-internal/.github`                        | Fallback files: issue templates, PR template, `.github/labels.yml`. Reusable workflows also live here, under `.github/workflows/` (`qa-routing.yml`, `subissue-rollup.yml`, and `label-sync.yml` built; CI still planned). |
| `havit-internal/002.HFW-NewProjectTemplate-Blazor` | GitHub template repo for new Blazor projects. Not org-wide — covers the Blazor stack specifically, not every repo type. |

Templates from this repo are **inherited** by every repo in the org that does
not define its own. Workflow files are **not** inherited — a reusable workflow
defined here still has to be called explicitly from a thin wrapper in each
consuming repo (`uses: havit-internal/.github/.github/workflows/<name>.yml@main`).
Those wrappers can be added through the GitHub UI (Actions → New workflow) —
see "Adding the wrappers to a repo" below. No separate workflows repo is needed for this — reusable workflows can be
called from any repo, including this one. If workflow versioning or ownership
ever needs to diverge from the templates/labels here, split them out then;
until that's a real need, keeping everything in one repo is simpler.

## Adding the wrappers to a repo

Each wrapper is published as an org **workflow template**
(`workflow-templates/` at the root of this repo). In the consuming repo, go
to **Actions → New workflow**, find the **By havit-internal** section, and
click **Configure** on *QA routing* or *Sub-issue status rollup*, then
commit. Nothing to copy by hand, and the `uses:` path to the
reusable workflow can't be mistyped.

`workflow-templates/*.yml` is the single source for the wrapper YAML — the
sections below describe what each workflow does and link there instead of
repeating it. Without the UI, copy the file into the consuming repo's
`.github/workflows/` under the same name. The wrappers take no inputs. Where
the jobs run is decided org-wide by one variable — see "Which runner" below.

## Issue template inheritance — the gotcha

Fallback is **all-or-nothing per repo**. As soon as a repo defines *any*
issue template of its own (even one), the org defaults stop applying to that
repo entirely — they do not merge with local templates.

Prefer keeping repos with **zero local templates** so they inherit the full
set defined here.

## Feature → Story → Task hierarchy

- **Feature** — outcome-level container. Not implemented directly; holds Stories.
- **Story** — a vertical, user-facing slice of a Feature. The unit that gets
  planned and delivered.
- **Task** — an implementation-level piece of a Story.
- **Bug** — unplanned work; can reference a parent Story or Feature but
  doesn't require one.

Every template is a single required **Description** box — free-form plain
text, no sections, no checklists, no prefilled content. Write whatever the
issue needs; nothing is asked for that the author doesn't want to give.

Parent/child linking is manual — mention the parent issue number in the
description, and use GitHub's sub-issue UI where it helps navigation.

## Work type vs. labels

Work type (feature / story / task / bug) is **not** a label — it's set via
GitHub's native org-level **Issue Types** (org `Settings → Issue types`),
which sync to every repo automatically with no workflow needed. `Task`,
`Bug`, `Feature`, and `Story` are all configured and enabled on the org,
matching the `type:` key each issue template sets.

The issue templates set their Issue Type via the top-level `type:` key in
each `.github/ISSUE_TEMPLATE/*.yml` file (not the `labels:` key). `labels.yml`
below is only for things Issue Types and issue statuses don't cover:
severity and triage/meta labels. Workflow status itself is **not** a label —
it's GitHub's native **issue status** (see "Native issue statuses" below).

No template presets `needs-triage` (or any other label), and that's
deliberate — **don't add `labels:` back**. A label listed in a template is
applied by GitHub a moment *after* the issue is created, as a separate step
that ignores the label picker, so deselecting it in the new-issue dialog
doesn't stick: it reappears on the created issue. Untriaged is better
expressed as a filter — `is:issue no:assignee` — and `needs-triage` stays in
`labels.yml` for whoever wants to set it by hand.

## Label sync

`.github/labels.yml` is the canonical list for severity (`sev:*`) and meta
labels (`needs-triage`, `blocked`, `skip-qa`, etc.) — everything that isn't a
work type or a workflow status. `.github/workflows/label-sync.yml` applies it
to every repo in the org automatically — on push to `main` when
`.github/labels.yml` changes, on a weekly schedule (catches repos created
since the last sync), and on demand (`workflow_dispatch`). Do not create
ad-hoc labels in individual repos — edit this file and open a PR here
instead.

Unlike the other workflows in this repo, `label-sync.yml` is **not**
reusable / per-repo-opt-in — it runs centrally, only here, and pushes out to
every repo in the org. No wrapper needed anywhere else.

It's **non-destructive**: creates labels that don't exist yet in a repo, and
updates the color/description of ones that already match by name, but never
deletes or otherwise touches a label that isn't in `.github/labels.yml` — a
repo's own ad-hoc labels are left alone.

It needs an org-level secret, **`ORG_LABEL_SYNC_SECRET`** — a fine-grained
PAT (or GitHub App token) with **Issues: Read and write** across *all
repositories* in the org. The default `GITHUB_TOKEN` won't work here: it's
scoped only to the repo the workflow runs in, and this workflow writes
labels to every other repo in the org. To set it up:

1. Create a [fine-grained PAT](https://github.com/settings/personal-access-tokens/new).
   Resource owner: `havit-internal`. Repository access: **All repositories**
   (so newly created repos are covered without reconfiguring the token).
   Permissions → Repository permissions → **Issues: Read and write**.
2. If the org requires approval for fine-grained PATs, an org owner needs to
   approve it before it's usable.
3. Add it as an **organization-level** Actions secret (Org Settings →
   Secrets and variables → Actions → New organization secret), named
   `ORG_LABEL_SYNC_SECRET`, with repository access limited to
   `havit-internal/.github` (the only repo that needs it).
4. Fine-grained PATs expire (max 1 year) — note the expiry and plan to
   rotate it, or move to a GitHub App later if this becomes long-lived.

## PR convention: `Closes`/`Fixes`/`Resolves` vs `Refs`

The `qa-routing` workflow (see below) piggybacks on GitHub's own closing
keywords instead of inventing a separate one. Use `Closes #N`, `Fixes #N`,
or `Resolves #N` — anywhere in the PR description, one or several
comma-separated (`Fixes #10, #11`) — for every issue this PR fixes. Here
the keyword only *links* the issue: our repos have GitHub's auto-close of
linked issues on merge turned off, so the merge does **not** close it (see
"Native issue statuses" below for the setting).

Linking the PR already moves the issue's status — a draft PR to
**In progress**, a PR ready for review to **In review**. On merge the issue
stays **In review**, which is our "waiting for QA", and `qa-routing` assigns
everyone in `.github/QAOWNERS`. It gets closed only when it's actually done —
when QA sets the status to **Done**, which closes it. If QA rejects the fix,
QA sets the status back to **In progress**. The one exception is a `skip-qa`
issue, which `qa-routing` sets to Done (and so closes) itself — see below.

You don't have to type a keyword at all — linking an issue via the PR's
**Development** panel (the "Link a pull request"/"Link an issue" UI) works
identically. The workflow reads GitHub's own resolved `closingIssuesReferences`
for the PR, the same data that panel displays, so a manually linked issue is
picked up exactly like a `Fixes #N` in the text — no body parsing involved.

- `Closes #N` / `Fixes #N` / `Resolves #N` in the body, **or** a manual link
  in the Development panel — This PR fixes issue N. On merge, stays
  In review and is assigned to QA by `qa-routing`; stays open until QA moves
  it to Done.
- `Refs #N` — Related issue, no action. Cross-link only, not routed to QA
  (plain text mentions never appear in `closingIssuesReferences` unless
  linked one of the two ways above).

## QA routing workflow

`.github/workflows/qa-routing.yml` is a reusable workflow. It does not run on its own —
each consuming repo needs a thin wrapper that triggers it on merge:
[`workflow-templates/qa-routing.yml`](workflow-templates/qa-routing.yml). Add it through the UI:
Actions → New workflow → By havit-internal → Configure.

### Which runner

The reusable workflows (qa-routing, subissue-rollup) run on our self-hosted
Windows runners by default — labels
`self-hosted`, `windows`, `qa-routing`. To move them to GitHub-hosted
`ubuntu-slim` (e.g. while the self-hosted runners are down), set the
organization variable `ISSUE_WORKFLOWS_RUN_ON_HOSTED_RUNNER` to `1`
(Organization settings → Secrets and variables → Actions → Variables, scoped to
all repositories). Any other value, or no variable at all, means self-hosted.
The switch applies to new runs only: jobs already queued for a self-hosted
runner stay queued, so cancel and re-run them. Nothing changes in the
consuming repos — the wrappers pass no inputs. `label-sync` is not affected;
it always runs on GitHub-hosted `ubuntu-slim` (it runs only in this repo, takes
about a minute, and needs `yq` and a checkout).

**Self-hosted runners need Actions Runner v2.327.1 or newer.** These workflows
run `actions/github-script` v9 (and `label-sync` runs `actions/checkout` v7),
which refuse to start on an older runner — the job fails before any script of
ours executes. Hosted runners like `ubuntu-slim` are always new enough, so
this only matters for the self-hosted runners.

What it does on merge:
1. Reads the PR's `closingIssuesReferences` (GraphQL) — the same resolved
   list GitHub shows in the PR's Development panel, covering both
   `Closes`/`Fixes`/`Resolves #N` text and manually linked issues. No
   matches → no-op.
2. For each linked issue, assigns **everyone** listed in that repo's
   `.github/QAOWNERS` — see below.
3. Makes sure the issue's native status is **In review** (`setIssueStatus`
   GraphQL mutation). GitHub usually set it already when the PR was opened,
   so this mostly confirms it; it's done *after* the assignment because
   GitHub moves an issue from Backlog to Todo when someone is assigned. An
   issue the merge closed anyway (the repo still has auto-close on) gets
   reopened by this — a closed issue can't be In review — and the run logs a
   warning naming the setting to turn off. An issue closed as *not planned*
   or *duplicate* is left closed, and so is one that was already closed as
   completed by anything other than this merge (QA's Done, a person, an
   earlier PR) — re-running the workflow or merging a follow-up PR can't
   reopen finished work. **Exception:** an issue labeled `skip-qa`
   goes to **Done** instead, which closes it — see below.
4. Leaves a comment on the issue linking back to the merged PR.

**`skip-qa` label:** for issues with nothing for a tester to verify (purely
technical work — refactors, dependency bumps, internal tooling). Label the
issue `skip-qa` *before* the PR merges, and `qa-routing` sets it straight to
**Done** (which closes it), instead of In review plus a QAOWNERS assignment.
`skip-qa` is defined in `labels.yml` alongside the other meta labels.

**`.github/QAOWNERS` format** (lives in each consuming repo, not here):
one GitHub username per line, `@` optional, `#` for comments, blank lines
ignored:

```
# QA owners for this repo — all are assigned on every merge
@alice
@bob
```

If a repo has no `QAOWNERS` file, the workflow still routes the issue to
In review but leaves it unassigned (logged as a warning in the workflow run)
— so repos can adopt this incrementally rather than needing `QAOWNERS` set
up before merges work at all.

The status IDs aren't hardcoded: each run looks them up by name from
`repository.planningStatuses`, and fails loudly if GitHub ever renames
**In review** or **Done**.

## Native issue statuses

Workflow status is GitHub's native **issue status** — a fixed set, the same
in every repo, that can't be extended:

| Status | Issue is | Set by |
|---|---|---|
| **Backlog** | open | GitHub, on every new issue; also on reopen |
| **Todo** | open | GitHub, when someone is assigned to a Backlog issue |
| **In progress** | open | GitHub, when a **draft** PR is linked |
| **In review** | open | GitHub, when a non-draft PR is linked or a draft is marked ready; `qa-routing` on merge. Means "waiting for someone else" — code review, then QA |
| **Done** | closed as completed | QA (or anyone) when it's done; `qa-routing` for `skip-qa`; closing an issue as completed |
| **Not planned** | closed as not planned | closing an issue as not planned |
| **Duplicate** | closed as duplicate | closing an issue as duplicate |

The status and the open/closed state are one thing: setting **Done**,
**Not planned** or **Duplicate** closes the issue, closing it sets the
matching status, and setting an open status on a closed issue reopens it.
Reopening with GitHub's **Reopen** button sets **Backlog** — to send a
closed issue back to work, set **In progress** instead, which reopens it
too. Assigning someone to an issue that's already In review leaves it there.

GitHub's automations above need no workflow at all, which is why
`issue-status-sync` and `pr-linked-status` are retired. Both are left as
no-op stubs so the wrappers still in some repos keep finishing (as skipped)
instead of failing — **delete those wrappers**
(`.github/workflows/issue-status-sync.yml` and `pr-linked-status.yml` in the
consuming repo).

**Auto-close must be off in every repo.** With it on, a merge closes the
linked issue as Done before QA has seen it. The setting is **Settings →
General → Issues → "Auto-close issues with merged linked pull requests"**;
it's on by default for a new repo and isn't exposed in the REST or GraphQL
API, so it can't be checked or set from here — turn it off by hand when a
repo is created. If it's left on, `qa-routing` reopens the issue to
In review and warns in its run log.

Statuses show up as icons in issue lists, can be filtered with `status:`,
and a repository's saved issue view can use a **Board** layout with columns
by status (swimlanes by type, priority, milestone, …) — enough for a team
that doesn't plan in iterations. Iterations exist only in Projects; see
"Project template" below.

## Sub-issue status rollup workflow

`.github/workflows/subissue-rollup.yml` is a reusable workflow that rolls a
parent issue's native **status** up from its children — Task/Bug statuses roll
up to their Story, Story statuses roll up to their Feature. It's the only
workflow here that acts on the Feature → Story → Task hierarchy; `qa-routing`
acts on one issue at a time. Wrapper:
[`workflow-templates/subissue-rollup.yml`](workflow-templates/subissue-rollup.yml). Add it through the UI:
Actions → New workflow → By havit-internal → Configure.

The rollup rule (child → parent only, never parent → child):

| State of the children | Parent status |
|-----------------------|---------------|
| No sub-issues | left alone — behaves as a normal issue |
| All still Backlog / Todo | left alone |
| Any started (In progress / In review / Done), not all Done | **In progress** |
| **All** children Done | **Done**, which closes the parent as completed |

**"Done" means the child's status is Done.** A task whose PR has merged waits
in **In review** until QA moves it to Done (auto-close is off, so the merge
doesn't close it), so it counts as *started*, not done, and merging the last
task can't mark a Story done before QA has signed off. The same holds in a
repo that still has auto-close on: a child closed by a pull request counts as
started (unless it's labeled `skip-qa`), because the merge closed it, not QA —
`qa-routing` reopens it to In review, and the rollup moves on once QA sets
Done. A repo with auto-close on and no `qa-routing` would therefore never see
its parents complete on their own — another reason auto-close must be off.

Key behaviours:

- **Native sub-issues only.** Rollup reads GitHub's `issue.parent` /
  `issue.subIssues`, so children must be linked with the **sub-issue UI/API**,
  not just a `#123` mention in the description. The manual-mention convention
  in "Feature → Story → Task hierarchy" above is fine for navigation but is
  invisible to this workflow.
- **Statuses by category, not by name.** Each status belongs to a category
  (backlog / planned / started / completed / not planned / duplicate), and the
  rule above reads those. An issue created before native statuses existed can
  have no status at all; it counts by its open/closed state and close reason.
- **Self-contained — it climbs the tree itself.** A status written with the
  default `GITHUB_TOKEN` does **not** trigger further workflow runs, so rollup
  can't lean on a re-fired rollup to climb Story → Feature. It walks the
  ancestor chain **within the same run** instead (each climb keeps its own
  `visited` set to stop it looping). So the last child reaching Done closes the
  Story *and* rolls that up into the Feature in one go.
- **Forward-only.** It only ever advances a parent toward Done. A child
  reopening under an already-closed parent does **not** reopen the parent —
  that's left to a human. A parent closed as *not planned* or *duplicate* is
  never touched (setting a status would reopen it). This is the deliberate
  cost of having no Story-level QA step.
- **"Not planned" children are excluded.** A child closed as *not planned* or
  *duplicate* is dropped from the rollup, so it can neither block the parent
  from reaching Done nor count as progress.
- **Same-repo only.** Sub-issues can cross repositories, but — like
  `qa-routing` — a parent or child in another repo is skipped: its events fire
  in that repo, and a run here can only read and write issues in its own.

What drives it: the wrapper triggers on a child's `status_changed` — GitHub
fires it for a person's change, for its own automations (PR linked, someone
assigned) and for a close or reopen, so one activity type covers all of them —
plus the `sub_issue_*` / `parent_issue_*` events, which catch a child being
re-parented without any status change. Status IDs are looked up by name on
each run, nothing is hardcoded.

**Status set by `qa-routing` reaches the rollup via a dispatch.** `qa-routing`
writes the status with `GITHUB_TOKEN`, so its writes fire no `status_changed`
— a merge (In review) or a `skip-qa` Done would otherwise never reach the
parent. After such a write it sends a `work-status-changed`
`repository_dispatch` listing the issues (the name predates native statuses
and is kept so existing wrappers keep listening), which `GITHUB_TOKEN` *can*
raise, and the rollup wrapper listens for it. Sending it needs **`contents:
write`** in the `qa-routing` wrapper (the template grants it). A repo that
keeps `contents: read` still works — the dispatch is skipped with an info line
— it just doesn't get `qa-routing`'s status rolled up. The reusable workflows
deliberately declare no `permissions:` of their own, so they inherit whatever
the wrapper grants and an older wrapper keeps starting.

**Concurrency caveat:** two status changes on *different children of the same
parent* can run concurrently and race on the parent's status — one run computes
Done while another computes In progress, and the forward-only guard only sees
its own snapshot, so the later writer can still move the parent back within that
narrow window. Each run recomputes from the parent's live children, so the next
event on that parent self-corrects; a hard guarantee would need a per-parent
lock that GitHub Actions can't express. Low-volume issue traffic makes this
rare, but it's a known limitation rather than a solved problem.

**Rollout — pilot first.** Like the other reusable workflows, this does
nothing until a consuming repo adds the wrapper. Add it to **one** repo first
(Actions → New workflow → *Sub-issue status rollup* → Configure), make sure
that repo's `qa-routing` wrapper grants `contents: write`, and exercise the
hierarchy there — start a child → the Story goes In progress; finish all
children → the Story goes Done and closes; same again up to the Feature — and
only then add it to the other repos. A repo that already has the older rollup
wrapper must replace it with the current template: the old one listens for
`field_added` (Work status), not `status_changed`. Nothing in this repo
changes to onboard another repo; it's wrapper-only, per repo.

## Project template: kanban on project Status, synced with the issue status

A project can't show an issue's native status directly — there's no such field
type. It keeps its own built-in **Status** field instead, and GitHub syncs that
with the issue's status **by option name** (case-insensitive), both ways:
change the issue's status and the project's Status follows; drag a card to
another column and the issue's status follows (a card dragged to **Done**
closes the issue). So the template's Status options must carry exactly the
native names. An option with no native counterpart stays project-only and is
overwritten by the next status change on the issue.

The sync happens only **on a change**: adding an issue to a project doesn't
copy its current status, even when the issue is already In progress, so a
freshly added item shows an empty Status until its issue's status next
changes — or until triage sets it (see below).

### Template shape

One org-level project, configured once, then Settings → Templates → **Copy as
template** so it shows up under `New project`.

Fields on the project:

| Field | Source | Role |
|---|---|---|
| `Status` | built-in project field, options **Backlog / Todo / In progress / In review / Done / Not planned** | board columns — synced with the issue's native status |
| `Type` | native Issue Type | swimlanes — Feature / Story / Task / Bug |
| `Priority` | org issue field | Urgent / High / Medium / Low — sort within a column |
| `Iteration` | project iteration field, 1-week | sprint scoping — the `@current` / `@next` views filter on it |
| `Parent issue`, `Sub-issues progress` | built-in | Feature → Story → Task roll-up |

**Duplicate** is left out of Status on purpose: GitHub needs to know *which*
issue it duplicates to close one that way, and a card dragged to a column
can't say. Close duplicates from the issue itself.

`Priority` is org-visibility **Org only**, and org-only issue fields are
hidden in projects that are public or internal — so either keep the project
private or flip `Priority` to All in the org's issue field settings.

Views, in tab order — boards for moving work day to day, tables for planning,
triage and bulk edits (inline edits, paste down a column, sort and group
without dragging cards). Every view is sorted by `Priority`, and every view
except **All work** is also filtered to `is:issue`:

| View | Layout | Filter / setup | Who, when |
|---|---|---|---|
| **Current iteration** | board | `iteration:@current`; columns `Status`, swimlanes `Type` | whole team, daily — the default view |
| **QA queue** | table | `status:"In review"`; shows `Assignees`, `Linked pull requests`, `Iteration` | testers — In review also holds issues still in code review; a merged PR in `Linked pull requests` and the QA assignee set by `qa-routing` tell them apart |
| **Next iteration** | table | `iteration:@next`; shows `Type`, `Priority`, `Assignees`, `Sub-issues progress` | sprint planning |
| **Backlog & triage** | table | `no:iteration -status:Todo,"In progress","In review",Done,"Not planned"` — i.e. Status empty or Backlog, no iteration | triage: set type, priority, Status, then an iteration |
| **Bugs** | table | `type:Bug`; shows `Labels` (for `sev:*`), `Iteration`, `Status` | deciding which bugs go into this or the next sprint |
| **Features** | table | `type:Feature`; shows `Sub-issues progress`, `Iteration` | product and leads: are each Feature's Stories moving? |
| **All work** | table | no filter at all — deliberately includes PRs and drafts; grouped by `Status` | overview and search, the catch-all |

The Backlog filter excludes the other Status values because a filter can't
express "empty *or* Backlog" directly. There's no roadmap view: its date axis
can't be set through the API, so add one by hand on a project that wants a
timeline.

Add the org fields from a table view (`+` in the header → Add field → the org
issue fields are listed alongside project fields); set column field and Group
by from the board's view-options menu. Pick filter values from the suggestion
dropdown rather than typing qualifiers — GitHub writes the qualifier itself,
including for multi-word field names.

The views were created through the REST API
(`POST /orgs/{org}/projectsV2/{number}/views`; a classic token or `gh` login
needs the `project` scope, a fine-grained token or GitHub App needs the
organization **Projects** read-and-write permission), which takes name, layout, filter, visible fields, sort, column field
and Group by. Two gaps: it silently drops Group by `Type` (the native Issue
Type isn't exposed as a groupable field), so the swimlanes on **Current
iteration** are set by hand; and there's no call to edit an existing view's
layout settings — GraphQL `updateProjectV2View` only changes name, filter and
visible fields. `deleteProjectV2View` does work, which is how the default
Table / Board / Roadmap views were removed.

### Why the views are filtered to `is:issue`

Only issues have a native status. Pull requests and draft issues can be given a
project Status, but it syncs with nothing, and nothing moves it — they'd pile up
in whatever column they were dropped in. `is:issue` keeps PRs and drafts off the
board; PRs are still visible on the cards through `Linked pull requests`.

A fresh issue is Backlog and has no iteration, and since the sync doesn't run
when an item is added, its project Status starts empty — so it never reaches
**Current iteration**; it shows up in **Backlog & triage**. That view is the
intake lane: triage means setting type, priority and Status, then an
iteration. Setting Status there also sets the issue's own status.

### Which built-in project automations to turn off

Because the project's Status is now synced with the issue's status, every
built-in project workflow that **sets Status** moves the issue too — and they
fight the native behaviour: "Item closed → Done" can turn an issue closed as
*not planned* into one closed as completed, "Item added to project" and "Pull
request linked to issue" would overwrite whatever status the issue has, and
"Auto-close issue" is redundant. **Turn off every built-in workflow that sets
Status** — "Item added to project", "Item closed", "Pull request linked to
issue", "Pull request merged", "Auto-close issue". GitHub's native statuses
already do that job on the issue itself, so it holds for issues in no project
at all.

The built-in workflows that don't touch Status stay worth using — auto-add
(pull items from a repo into the project) above all, plus "Auto-add sub-issues
to project" and auto-archive.

### What a copy carries, and what it doesn't

Copying a project (or creating one from the template) brings the views, the
fields and their values, draft issues, insights, and configured workflows —
**except auto-add workflows**, which are never copied. Every new project
therefore needs its own "auto-add items from repo X" workflow wired up by
hand. The Status options come along as they are, so a copy syncs with the
issues' statuses from day one.

## Migrating from Work status

Before native statuses, workflow status lived in an org issue field,
**Work status** (Backlog / Ready / In progress / Ready for QA / Done), and the
retired `issue-status-sync` / `pr-linked-status` workflows drove it. Issues
created before native statuses existed have no native status, and the ones
that do may disagree with their Work status.

`.github/workflows/migrate-work-status.yml` copies Work status onto the
native status for every **open** issue in the org: Backlog → Backlog,
Ready → Todo, In progress → In progress, Ready for QA → In review. It never
moves an issue backward — if GitHub's automations already put it further along
(say In review because a PR is open), that's kept — and an open issue whose
Work status is Done is only listed, because setting Done would close it. Run it
by hand from this repo's Actions tab: it's a **dry run by default** and puts
what it would change in the job summary; run it again with the box unticked to
write. An optional `repo` input limits it to one repo. It writes through
`ORG_LABEL_SYNC_SECRET` (see "Label sync"), so each change fires
`status_changed` in that repo and syncs to any project whose Status already has
the native names — switch the projects over first.

Order of the switch-over:

1. Merge the PR that introduced native statuses here — `qa-routing` and
   `subissue-rollup` move to native statuses for every repo at once.
2. Per project (and the template): give Status the native options, point the
   board columns, filters and grouping at `Status` instead of `Work status`,
   and turn off the built-in workflows that set Status.
3. Run `migrate-work-status` — dry run, check the summary, then for real.
4. Per repo: delete the `issue-status-sync` / `pr-linked-status` wrappers,
   replace an old `subissue-rollup` wrapper with the current template, and
   check auto-close is off.
5. Only then delete the **Work status** field (org Settings → Planning → Issue
   fields) — deleting it drops every value it holds.

## Claude Code plugin

This repo doubles as a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`).
It currently ships one plugin, `gh-issue-templates`, with a skill that creates
GitHub issues matching the actual template a target repo renders (including
the all-or-nothing inheritance gotcha above) — field order, labels, and
native Issue Type — instead of a freeform title/body.

Install once, works in any repo:

```
/plugin marketplace add havit-internal/.github
/plugin install gh-issue-templates
```

## Changing something here

Everything in this repo affects every repo in the org. Open a PR, get review
from at least one other maintainer, and merge. Changes take effect immediately
for any repo that doesn't override.
