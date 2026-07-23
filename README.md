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
│   ├── bug_report.yml       ← "Something broken" — Environment + Details required
│   ├── user_story.yml       ← "As a … so that …" with acceptance criteria
│   ├── task.yml             ← Implementation slice with definition-of-done
│   └── config.yml           ← Disables blank issues
├── pull_request_template.md ← Issues / Refs — see QA convention below
├── labels.yml               ← Source of truth for sev:*/status:*/meta labels (work type is an Issue Type, not a label)
└── workflows/
    ├── qa-routing.yml        ← Reusable workflow — see "PR convention" below.
    ├── issue-status-sync.yml ← Reusable workflow — issue closed ⟷ Work-status Done, both directions
    ├── pr-linked-status.yml  ← Reusable workflow — PR linked to issue → Work-status In-progress
    └── label-sync.yml        ← Runs centrally — see "Label sync" below. CI still planned.

plugins/
└── gh-issue-templates/      ← Claude Code plugin — see "Claude Code plugin" below
```

## Related org repos

| Repo | Role |
|------|------|
| `havit-internal/.github`                        | Fallback files: issue templates, PR template, `.github/labels.yml`. Reusable workflows also live here, under `.github/workflows/` (`qa-routing.yml`, `issue-status-sync.yml`, `pr-linked-status.yml`, and `label-sync.yml` built; CI still planned). |
| `havit-internal/002.HFW-NewProjectTemplate-Blazor` | GitHub template repo for new Blazor projects. Not org-wide — covers the Blazor stack specifically, not every repo type. |

Templates from this repo are **inherited** by every repo in the org that does
not define its own. Workflow files are **not** inherited — a reusable workflow
defined here still has to be called explicitly from a thin wrapper in each
consuming repo (`uses: havit-internal/.github/.github/workflows/<name>.yml@main`).
No separate workflows repo is needed for this — reusable workflows can be
called from any repo, including this one. If workflow versioning or ownership
ever needs to diverge from the templates/labels here, split them out then;
until that's a real need, keeping everything in one repo is simpler.

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
- **Bug** — unplanned work; can reference a parent Story or Feature via
  "Related work" but doesn't require one.

Each template's "parent" field is manual (plain issue number, not GitHub's
native sub-issue linking) — link them explicitly and use sub-issues where it
helps navigation.

## Work type vs. labels

Work type (feature / story / task / bug) is **not** a label — it's set via
GitHub's native org-level **Issue Types** (org `Settings → Issue types`),
which sync to every repo automatically with no workflow needed. `Task`,
`Bug`, and `Feature` are already configured on the org; `Story` still needs
to be added there to match `user_story.yml`, which already sets `type: story`.

The issue templates set their Issue Type via the top-level `type:` key in
each `.github/ISSUE_TEMPLATE/*.yml` file (not the `labels:` key). `labels.yml`
below is only for things Issue Types don't cover: severity, workflow status,
and triage/meta labels.

## Label sync

`.github/labels.yml` is the canonical list for severity (`sev:*`), status (`status:*`),
and meta labels (`needs-triage`, `blocked`, `skip-qa`, etc.) — everything
that isn't a work type. `.github/workflows/label-sync.yml` applies it to
every repo in the org automatically — on push to `main` when
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

It needs an org-level secret, **`ORG_LABEL_SYNC_TOKEN`** — a fine-grained
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
   `ORG_LABEL_SYNC_TOKEN`, with repository access limited to
   `havit-internal/.github` (the only repo that needs it).
4. Fine-grained PATs expire (max 1 year) — note the expiry and plan to
   rotate it, or move to a GitHub App later if this becomes long-lived.

**Note:** the `status:*` labels here (`in-progress`/`in-review`/`ready-for-qa`/
`qa-rejected`) now overlap with the org-wide **Work-status** issue field
(`Backlog`/`Ready`/`In-progress`/`In-review`/`Ready for QA`/`Done`) that
`qa-routing` writes to (see below) — the two aren't kept in sync with each
other. Worth deciding separately whether `status:*` should be retired in
favor of Work-status.

## PR convention: `Closes`/`Fixes`/`Resolves` vs `Refs`

The `qa-routing` workflow (see below) piggybacks on GitHub's own closing
keywords instead of inventing a separate one. Use `Closes #N`, `Fixes #N`,
or `Resolves #N` — anywhere in the PR description, one or several
comma-separated (`Fixes #10, #11`) — for every issue this PR fixes. GitHub
auto-closes those issues on merge, and the workflow, in the same run, sets
the org-wide **Work-status** issue field to **Ready for QA** and assigns
everyone in `.github/QAOWNERS`, so the issue doesn't just quietly close —
it still gets a QA pass.

You don't have to type a keyword at all — linking an issue via the PR's
**Development** panel (the "Link a pull request"/"Link an issue" UI) works
identically. The workflow reads GitHub's own resolved `closingIssuesReferences`
for the PR, the same data that panel displays, so a manually linked issue is
picked up exactly like a `Fixes #N` in the text — no body parsing involved.

- `Closes #N` / `Fixes #N` / `Resolves #N` in the body, **or** a manual link
  in the Development panel — This PR fixes issue N. Closed by GitHub and
  routed to QA on merge.
- `Refs #N` — Related issue, no action. Cross-link only, not auto-closed, not
  routed to QA (plain text mentions never appear in `closingIssuesReferences`
  unless linked one of the two ways above).

## QA routing workflow

`.github/workflows/qa-routing.yml` is a reusable workflow. It does not run on its own —
each consuming repo needs a thin wrapper that triggers it on merge:

```yaml
# .github/workflows/qa-routing.yml (in the consuming repo)
name: QA routing

on:
  pull_request:
    types: [closed]

jobs:
  route-to-qa:
    if: github.event.pull_request.merged == true
    permissions:
      issues: write
      pull-requests: read
      contents: read
    uses: havit-internal/.github/.github/workflows/qa-routing.yml@main
    with:
      runner: '["ubuntu-latest"]'   # optional — defaults to this. JSON array
                                    # of runner labels, e.g. '["self-hosted","on-prem"]'
```

What it does on merge:
1. Reads the PR's `closingIssuesReferences` (GraphQL) — the same resolved
   list GitHub shows in the PR's Development panel, covering both
   `Closes`/`Fixes`/`Resolves #N` text and manually linked issues. No
   matches → no-op.
2. For each linked issue, sets the org-wide **Work-status** issue field
   (single select) to **Ready for QA** via the `setIssueFieldValue` GraphQL
   mutation — a single-select field only ever holds one value, so this
   always replaces whatever Work-status was there before (no stacking,
   unlike labels). **Exception:** an issue labeled `skip-qa` goes straight
   to **Done** and gets closed instead — see below.
3. Assigns **everyone** listed in that repo's `.github/QAOWNERS` — see below
   (skipped for `skip-qa` issues, since there's no QA pass to assign).
4. Leaves a comment on the issue linking back to the merged PR.

**`skip-qa` label:** for issues with nothing for a tester to verify (purely
technical work — refactors, dependency bumps, internal tooling). Label the
issue `skip-qa` *before* the PR merges, and `qa-routing` sets Work-status
straight to **Done** and closes the issue itself, instead of Ready for QA
plus a QAOWNERS assignment. `skip-qa` is defined in `labels.yml` alongside
the other meta labels.

**`.github/QAOWNERS` format** (lives in each consuming repo, not here):
one GitHub username per line, `@` optional, `#` for comments, blank lines
ignored:

```
# QA owners for this repo — all are assigned on every merge
@alice
@bob
```

If a repo has no `QAOWNERS` file, the workflow still sets Work-status to
Ready for QA but leaves the issue unassigned (logged as a warning in the
workflow run) — so repos can adopt this incrementally rather than needing
`QAOWNERS` set up before merges work at all.

**Work-status field IDs:** the workflow references the `Work-status` field
and its `Ready for QA` and `Done` options (the latter used by the `skip-qa`
path above) by GraphQL node ID (single-select fields are set by option ID,
not name). Those IDs are hardcoded as constants in `qa-routing.yml` — if the
field or an option is ever deleted and recreated (even with the same name),
it gets new IDs and the workflow needs updating. Look them up via
`repository.issueFields` for any repo in the org (the field is org-wide, so
it shows up the same everywhere).

## Issue status sync workflow

`.github/workflows/issue-status-sync.yml` is a reusable workflow that keeps
an issue's open/closed state and its **Work-status** field in sync, in both
directions. Wrapper:

```yaml
# .github/workflows/issue-status-sync.yml (in the consuming repo)
name: Issue status sync

on:
  issues:
    types: [closed, field_added]

jobs:
  sync:
    permissions:
      issues: write
      contents: read
    uses: havit-internal/.github/.github/workflows/issue-status-sync.yml@main
    with:
      runner: '["ubuntu-latest"]'   # optional — defaults to this
```

What it does:
- **Issue closed as completed** → sets Work-status to **Done**. A close with
  any other reason (won't-fix, duplicate) is left alone — "Done" implies
  actual completion, not "not planned".
- **Work-status set to Done** (the `field_added` activity type, which GitHub
  fires whenever any issue field value is set or changed) → closes the
  issue as completed.

Each direction checks the other side's current state before acting (already
Done? already closed?), so triggering one doesn't bounce back and forth with
the other — it settles after at most one harmless extra run.

## PR-linked issue status workflow

`.github/workflows/pr-linked-status.yml` is a reusable workflow that moves
an issue's **Work-status** to **In-progress** as soon as a PR is linked to
it — same `closingIssuesReferences` detection as `qa-routing.yml` (body
keyword or Development panel link, either way). Wrapper:

```yaml
# .github/workflows/pr-linked-status.yml (in the consuming repo)
name: PR-linked issue status

on:
  pull_request:
    types: [opened, reopened, ready_for_review, edited]

jobs:
  sync:
    permissions:
      issues: write
      pull-requests: read
      contents: read
    uses: havit-internal/.github/.github/workflows/pr-linked-status.yml@main
    with:
      runner: '["ubuntu-latest"]'   # optional — defaults to this
```

It skips issues whose Work-status is already **Ready for QA** or **Done**,
so it never walks status backward (e.g. a small follow-up PR after QA
rejected it shouldn't undo that progress). Caveat: a PR linked *purely*
through the Development panel, with no further edit to the PR itself,
won't trigger this workflow until the PR's next `opened`/`edited`-type
event — there's no dedicated webhook event for "issue linked via panel"
alone.

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
