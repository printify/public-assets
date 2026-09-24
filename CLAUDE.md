## AI code review rubric

This repo's review rubric lives at `.claude/skills/review-pr-additions/SKILL.md`,
consumed by the `code-reviewer` plugin's `review-pr` skill during agentic PR review
(via `fyulorg/agentic-review`). It layers repo-specific checks over the universal baseline.

Self-healing: when you change architecture, domain boundaries, test conventions, or
process rules, refresh the rubric in the same change — re-run `/generate-review-rubric`
or edit the file — and bump its `_Last verified_` date.
