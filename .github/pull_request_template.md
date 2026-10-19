## Summary
<!-- What does this PR do? 1–3 sentences. -->


## Issues
<!--
  Which issue(s) does this PR fix? Use GitHub's own closing keywords —
  Closes, Fixes, or Resolves — followed by the issue number(s), e.g.
  "Fixes #123" or "Closes #10, #11" (or link the issue via the Development
  panel on the right instead — either way works). The merge does not close
  the issue(s): the `qa-routing` workflow sets their Work status field to
  "Ready for QA" and assigns everyone listed in this repo's
  `.github/QAOWNERS`, and they stay open until QA sets Work status to
  "Done". Repos without that file still get Work status set — the issues
  are just left unassigned and the run logs a warning. An issue labeled
  `skip-qa` goes straight to "Done", is closed on merge and is never
  assigned to QA.
-->
Fixes #


## Related
<!-- Issues this PR relates to but does not fix. Cross-link only — not routed to QA. -->
Refs #


## Checklist
- [ ] Tests added or updated
- [ ] Docs updated where relevant (README, XML doc comments, wiki)
- [ ] Linked issue carries `skip-qa` if it should bypass QA and close on merge
