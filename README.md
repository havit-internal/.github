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
│   ├── bug_report.yml       ← "Something broken" — severity + repro required
│   ├── user_story.yml       ← "As a … so that …" with acceptance criteria
│   ├── task.yml             ← Implementation slice with definition-of-done
│   └── config.yml           ← Disables blank issues
├── pull_request_template.md ← Verify: / Refs — see QA convention below
└── labels.yml               ← Source of truth for org-wide labels
```

## The three org repos and what each holds

| Repo | Role |
|------|------|
| `havit-internal/.github`           | Fallback files: issue templates, PR template, `labels.yml`. |
| `havit-internal/.github-workflows` | Central reusable workflows (QA routing, label sync, CI). |
| `havit-internal/repo-template`     | File baseline copied into new repos on creation. |

Templates from this repo are **inherited** by every repo in the org that does
not define its own. Workflow files, on the other hand, are **not** inherited —
they only run in the repo they live in, which is why the reusable workflows
sit in a separate repo and each consumer carries a thin wrapper.

## Issue template inheritance — the gotcha

Fallback is **all-or-nothing per repo**. As soon as a repo defines *any*
issue template of its own (even one), the org defaults stop applying to that
repo entirely — they do not merge with local templates.

Prefer keeping repos with **zero local templates** so they inherit the full
set defined here.

## Label sync

`labels.yml` is the canonical list. It is applied to every repo in the org by
a scheduled workflow in `havit-internal/.github-workflows` (label-sync). Do
not create ad-hoc labels in individual repos — edit this file and open a PR.

## PR convention: `Verify:` vs `Refs`

The PR template contains a `Verify:` line. Fill it with the issue number(s)
this PR fixes — the QA-routing workflow will pick these up on merge, relabel
the issues `status:ready-for-qa`, and assign the QA owner from `.github/QAOWNERS`
in the consuming repo.

- `Verify: #N` — This PR fixes issue N. Route to QA on merge.
- `Refs #N`   — Related issue, no action. Cross-link only.
- **Never use** `Closes #N`, `Fixes #N`, or `Resolves #N` — GitHub auto-closes
  the referenced issue on merge, bypassing the QA verification gate.

## Changing something here

Everything in this repo affects every repo in the org. Open a PR, get review
from at least one other maintainer, and merge. Changes take effect immediately
for any repo that doesn't override.
