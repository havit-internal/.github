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
│   ├── bug_report.yml       ← "Something broken" — severity + repro required
│   ├── user_story.yml       ← "As a … so that …" with acceptance criteria
│   ├── task.yml             ← Implementation slice with definition-of-done
│   └── config.yml           ← Disables blank issues
├── pull_request_template.md ← Verify: / Refs — see QA convention below
├── labels.yml               ← Source of truth for sev:*/status:*/meta labels (work type is an Issue Type, not a label)
└── workflows/
    └── qa-routing.yml        ← Reusable workflow — see "PR convention" below. Label sync / CI still planned.
```

## Related org repos

| Repo | Role |
|------|------|
| `havit-internal/.github`                        | Fallback files: issue templates, PR template, `labels.yml`. Reusable workflows also live here, under `.github/workflows/` (`qa-routing.yml` built; label sync and CI still planned). |
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

`labels.yml` is the canonical list for severity (`sev:*`), status (`status:*`),
and meta labels (`needs-triage`, `blocked`, etc.) — everything that isn't a
work type. **Planned:** a scheduled label-sync workflow, living in this
repo's own `.github/workflows/`, will apply it to every repo in the org. That
workflow does not exist yet, so today this file is documentation only —
labels must be applied to each repo manually until it's built. Do not create
ad-hoc labels in individual repos — edit this file and open a PR.

## PR convention: `Verify:` vs `Refs`

The PR template contains a `Verify:` line. Fill it with the issue number(s)
this PR fixes — one `Verify: #N` per line, or several numbers on one line.
On merge, the `qa-routing` workflow (see below) picks these up, relabels the
issues `status:ready-for-qa`, and assigns everyone in `.github/QAOWNERS`.

- `Verify: #N` — This PR fixes issue N. Routed to QA on merge.
- `Refs #N`   — Related issue, no action. Cross-link only.
- **Never use** `Closes #N`, `Fixes #N`, or `Resolves #N` — GitHub auto-closes
  the referenced issue on merge, bypassing the QA verification gate.

## QA routing workflow

`workflows/qa-routing.yml` is a reusable workflow. It does not run on its own —
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
      contents: read
    uses: havit-internal/.github/.github/workflows/qa-routing.yml@main
```

What it does on merge:
1. Scans the PR body for `Verify: #N` lines. No matches → no-op.
2. For each linked issue, strips any existing `status:*` label and adds
   `status:ready-for-qa` (handles the qa-rejected → refix → re-verify loop
   cleanly, since the old status is always replaced, not stacked).
3. Assigns **everyone** listed in that repo's `.github/QAOWNERS` — see below.
4. Leaves a comment on the issue linking back to the merged PR.

**`.github/QAOWNERS` format** (lives in each consuming repo, not here):
one GitHub username per line, `@` optional, `#` for comments, blank lines
ignored:

```
# QA owners for this repo — all are assigned on every merge
@alice
@bob
```

If a repo has no `QAOWNERS` file, the workflow still relabels the issue
`status:ready-for-qa` but leaves it unassigned (logged as a warning in the
workflow run) — so repos can adopt this incrementally rather than needing
`QAOWNERS` set up before merges work at all.

## Changing something here

Everything in this repo affects every repo in the org. Open a PR, get review
from at least one other maintainer, and merge. Changes take effect immediately
for any repo that doesn't override.
