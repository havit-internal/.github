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
└── workflows/                ← Planned: reusable workflows (label sync, QA routing, CI) — not yet built
```

## Related org repos

| Repo | Role |
|------|------|
| `havit-internal/.github`                        | Fallback files: issue templates, PR template, `labels.yml`. Reusable workflows (QA routing, label sync, CI) will also live here, under `.github/workflows/`, once built. |
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
this PR fixes. **Planned:** a QA-routing workflow, living in this repo's own
`.github/workflows/`, will pick these up on merge, relabel the issues
`status:ready-for-qa`, and assign the QA owner from `.github/QAOWNERS` in the
consuming repo. Until that workflow exists, relabeling and QA assignment are
manual — the convention below still matters so it's a clean cutover once the
automation lands.

- `Verify: #N` — This PR fixes issue N. Route to QA on merge.
- `Refs #N`   — Related issue, no action. Cross-link only.
- **Never use** `Closes #N`, `Fixes #N`, or `Resolves #N` — GitHub auto-closes
  the referenced issue on merge, bypassing the QA verification gate.

## Changing something here

Everything in this repo affects every repo in the org. Open a PR, get review
from at least one other maintainer, and merge. Changes take effect immediately
for any repo that doesn't override.
