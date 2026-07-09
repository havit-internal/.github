---
name: create-github-issue
description: Use when the user asks to file, open, or create a GitHub issue — a bug, task, user story, or feature — in any havit-internal repo. Triggers on phrases like "create an issue", "file a bug", "open a GH issue", "log a task/story/feature". Builds the issue to match the target repo's actual issue-form template (Bug report / Task / User story / Feature) instead of a freeform title+body.
version: 1.0.0
---

# Create GitHub issue from template

Creates a GitHub issue whose title, body sections, labels, and native Issue
Type match one of the org's issue-form templates — the same structure a
human would get filling out the form in the GitHub web UI.

Do not just call `gh issue create --title ... --body ...` with freeform text.
The whole point of this skill is to produce output that matches the template
fields exactly, in order, with matching headings.

## 1. Resolve the target repo

- If the user names a repo, use it (`owner/repo`).
- Otherwise, derive it from the current directory: `git remote get-url origin`.
- If neither is available, ask.

## 2. Resolve which template applies

Ask (or infer from the user's wording) which of the four kinds this is:
`bug_report`, `task`, `user_story`, `feature`. If ambiguous, ask — don't guess;
the fields differ enough that guessing wrong wastes the user's answers.

## 3. Fetch the actual template that repo will render

Template inheritance is **all-or-nothing per repo** (see this repo's
README): if the target repo defines *any* of its own issue templates, none
of the org defaults from `havit-internal/.github` apply to it — so you must
check the target repo first, not assume the org default.

```bash
# Does the target repo have its own ISSUE_TEMPLATE dir?
gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE 2>/dev/null
```

- If that returns a non-empty listing, fetch the specific file from **that
  repo**:
  ```bash
  gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE/<template>.yml --jq '.content' | base64 -d
  ```
- If it 404s or is empty, the repo inherits the org defaults — fetch from
  `havit-internal/.github` instead:
  ```bash
  gh api repos/havit-internal/.github/contents/.github/ISSUE_TEMPLATE/<template>.yml --jq '.content' | base64 -d
  ```

If you're already working inside a checked-out copy of the resolved source
repo, reading the local file is equivalent and faster — use that instead of
the API call.

## 4. Parse the template

From the fetched YAML, extract:

- `type:` — the native GitHub Issue Type to set (e.g. `bug`, `task`, `story`, `feature`)
- `labels:` — list of labels to apply (e.g. `needs-triage`)
- `body:` — the ordered list of fields. For each entry with `type: input` or
  `type: textarea` (skip `type: markdown`, it's instructional text only),
  note: `id`, `label`, `description`, `placeholder` or `value` (default
  content), and whether `validations.required` is `true`.

## 5. Gather content for each field

For each non-markdown field, in template order:

- If the user already gave you this information earlier in the conversation,
  use it — don't re-ask.
- If a field has a `value:` (pre-filled default, e.g. the user-story
  "As a … I want … so that …" skeleton), use it as a starting point and ask
  the user to fill in the specifics rather than asking an open-ended question.
- Otherwise ask the user directly for the field, using its `label` and
  `description` verbatim so they know what's expected.
- Required fields need real content — don't invent placeholder text to fill
  them. Optional fields can be omitted entirely if the user has nothing to
  add for them.

## 6. Compose title and body

- Title: a concise summary (from the user, or the first line of the main
  field) — no prefix; the native Issue Type set in step 8 already conveys
  bug/task/story/feature.
- Body: for each field with content, in template order:
  ```
  ### <label>

  <content>

  ```
  Skip fields with no content (don't emit an empty heading).
- Footer: always append an authorship note as the last line of the body,
  identifying the actual AI client running this skill — do not hardcode
  "Claude Code" if you are a different client:
  ```

  ---
  🤖 Authored by an AI assistant (<client name>) via the `gh-issue-templates` plugin.
  ```
  If you know the requesting GitHub user, append `Requested by @<login>.` on
  its own line below that. This note is not optional and is not something to
  ask the user about — always include it so anyone reading the issue knows
  it wasn't typed by a human.

## 7. Confirm before creating

This is a visible, shared-state action — show the user the exact result
before creating anything:

- Target repo
- Title
- Full composed body
- Labels to apply
- Issue Type to set

Get an explicit go-ahead in the current message before proceeding. Creating
the issue without this confirmation is not acceptable, even if the user
approved a similar action earlier in the conversation.

## 8. Create the issue

Use `gh api` directly (not `gh issue create`) so title, body, labels, and the
native Issue Type are all set atomically in one call:

```bash
gh api repos/<owner>/<repo>/issues \
  -f title="<title>" \
  -f body="$(cat body.md)" \
  -f type="<type>" \
  -f 'labels[]=<label-1>' \
  -f 'labels[]=<label-2>'
```

Write the body to a temp file first (in the scratchpad directory) rather than
inlining a multi-line string, to avoid quoting issues.

Report back the created issue's URL (`.html_url` in the response). If the
response shows labels or type missing compared to what you sent, say so
explicitly — GitHub silently drops `type`/`labels` for users without push
access rather than erroring.
