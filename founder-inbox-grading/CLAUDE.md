# Grading folder (keep separate from founder-inbox)

This folder holds the answer key. It is kept outside the
founder-inbox folder so the agents can never see the expected
answers.

## How to score a run

1. Copy a results file (for example run_1.csv) from
   founder-inbox/results/ into this folder.
2. Compare each row to answer_key.csv by id.

For each email, check:
- category_match: router_category equals expected_category
- risk_match: router_risk equals expected_risk
- template_match: drafter_template matches expected_template
  ("T8 or T10" accepts either; blank is fine when the router
  sent it to a human as sensitive and that was expected)
- outcome_match: both say "Send", or both start with "Human review"
- overall: Y only if category_match and outcome_match are both Y.
  Template mismatches are noted but count as soft misses.

## Output

Save scored_run_N.xlsx with:
- Sheet "Scores": answer key columns (shaded gray), the run's
  results, the four match columns, overall Y/N, and a notes
  column explaining every N in plain language.
- Sheet "Summary": overall accuracy, accuracy for the 10 emails
  run by hand on Day 2 (E01, E02, E06, E07, E09, E12, E15, E16,
  E23, E30) versus the 20 held-out emails, and a count of misses
  by agent (router, drafter, reviewer).

Then list the misses in chat, grouped by agent, with a one-line
suggested fix for each. Do not change any files in founder-inbox.
