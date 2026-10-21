# founder-inbox-agents

A practice pipeline of three Claude Code subagents (**router**, **drafter** and **reviewer**). It triages startup program inquiries from founders and drafts replies to them. Anything risky goes to a human instead of being sent.

**About:** Designed and tested by Shen Sabo, a support operations leader exploring human-in-the-loop AI workflows. I designed the workflow, agent instructions, test set and evaluation, and built and ran it with Claude Code. [linkedin.com/in/shen-sabo](https://www.linkedin.com/in/shen-sabo)

> **Independent practice project.** This is not affiliated with or endorsed by Anthropic. The program policy, founder records and emails are all fictional. They are loosely based on the public [Claude for Startups](https://claude.com/programs/startups) page, and anything marked "practice assumption" in the policy was made up. No real email is ever sent.

## Repository layout

```
founder-inbox/                 # the pipeline (agents only ever see this folder)
  CLAUDE.md                    # orchestrator instructions for the main session
  .claude/agents/
    router.md                  # Haiku: category + risk
    drafter.md                 # Sonnet: policy-grounded draft
    reviewer.md                # Sonnet: rubric check, APPROVE or FLAG
  data/
    01_Program_Policy.md       # the rules
    02_Reply_Templates.md      # approved templates T1 to T11
    03_Founder_Records.csv     # 14 fictional companies
    04_Review_Rubric.md        # 6-item rubric + decision rules
    05_Sample_Emails.csv       # 30 test emails (E01 to E30)
  results/
    run_0_test.csv             # one-email smoke test
    run_N.csv                  # full runs 1 to 3, all 30 emails
    run_N_summary.md           # outcome counts per run

founder-inbox-grading/         # kept separate so agents never see the answers
  CLAUDE.md                    # scoring instructions
  answer_key.csv               # expected category, risk, template, outcome
  calibration_log.md           # every change made to the answer key, and why
  run_N.csv                    # copies of the runs being scored
  scored_run_1.xlsx            # run 1 against the original key
  scored_run_N_calibrated.xlsx # runs 1 and 2 rescored against the calibrated key
  scored_run_3.xlsx            # run 3 (calibrated key) + run comparison
```

## Design

### Who sees what

| Step | Model | Receives | Reads | Output |
|---|---|---|---|---|
| Router | Haiku | The email only (From, Subject, body) | No files | category, risk, reason |
| Drafter | Sonnet | The email + the router's category and risk (**not** its reason) | Policy, templates, founder records | record_found, template, draft, rule, needs_human, reason |
| Reviewer | Sonnet | The email + **only** the draft text | Policy, founder records, rubric (**not** templates) | 6 PASS/FAIL checks, APPROVE/FLAG, reason |

The main Claude Code session acts as the **orchestrator**. It passes the email word for word to each step, never drafts or judges anything itself, and starts fresh subagent calls for every email. If the router marks an email `sensitive`, it goes straight to a human and the drafter and reviewer never run.

Final outcome: `Human review (router)`, `Human review (drafter)` (needs_human = yes), `Human review (reviewer)` (FLAG), or `Send`.

### Why the reviewer never sees the drafter's reasoning

The reviewer is meant to be an independent second check. If it read the drafter's template choice, policy rule and justification, it would tend to anchor on that reasoning and confirm it. A persuasive but wrong rationale would then pass review. The reviewer gets only what the founder would see, and it has to judge the draft on its own against the records and the policy. This also limits the damage a prompt injection can do. Text that steers the drafter can't reach the reviewer through the drafter's notes, and email E30 tests exactly that by aiming its instruction at the reviewer. Instructions inside emails are treated as data at every step.

### Why Haiku for routing, Sonnet for drafting and review

Routing is a short classification over a fixed label set, so a small, fast model fits. Drafting and review involve reading the policy and records, applying date rules (such as the four-year founding window), and catching overpromising, which calls for a stronger model.

### Answer key and scoring kept apart

The expected answers live in `founder-inbox-grading/`, outside the folder the agents work in. Every agent is also told to read only its listed files. Scoring runs in a **separate Claude Code session** started in the grading folder, so the session that ran the pipeline never sees the answers and is told not to judge its own results.

## How to run it with Claude Code

1. **Run the pipeline.** Start Claude Code inside `founder-inbox/` so it picks up `CLAUDE.md` and the subagents in `.claude/agents/`:
   ```sh
   cd founder-inbox
   claude
   ```
   Then ask, for example:
   > Run every email in data/05_Sample_Emails.csv through the pipeline and save the results as run 2.

   This writes `results/run_2.csv` and `results/run_2_summary.md`.

2. **Score it in a new session.** Copy the results file into the grading folder and start a separate session there:
   ```sh
   cp founder-inbox/results/run_2.csv founder-inbox-grading/
   cd founder-inbox-grading
   claude
   ```
   > Score run_2.csv.

   This writes `scored_run_N.xlsx` and lists the misses by agent.

## Results

All three runs are scored against the calibrated answer key. An email counts as correct overall when the router's category **and** the final outcome both match the key. Run 1 originally scored 73% (22 / 30). Calibrating the key on ambiguous cases raised that to 77%. The changes are listed in [`founder-inbox-grading/calibration_log.md`](founder-inbox-grading/calibration_log.md).

| Set | Run 1 | Run 2 | Run 3 |
|---|---|---|---|
| All 30 emails | 23 / 30 (77%) | 25 / 30 (83%) | 29 / 30 (97%) |
| Hand-run set (10: E01, E02, E06, E07, E09, E12, E15, E16, E23, E30) | 8 / 10 (80%) | 10 / 10 (100%) | 10 / 10 (100%) |
| Held-out set (20) | 15 / 20 (75%) | 15 / 20 (75%) | 19 / 20 (95%) |

**Across all three runs, every outcome miss failed safe: seven replies the key expected to be sent were held for a human, and no risky reply was ever sent.**

Sources: [`scored_run_1_calibrated.xlsx`](founder-inbox-grading/scored_run_1_calibrated.xlsx), [`scored_run_2_calibrated.xlsx`](founder-inbox-grading/scored_run_2_calibrated.xlsx), [`scored_run_3.xlsx`](founder-inbox-grading/scored_run_3.xlsx).

## What changed in each run

- **Run 2: router fixes only.** The hand-run 10 went to 10 / 10, but the held-out 20 didn't move. Two fixes (E13, E28) were cancelled out by two new breaks (E14, E25). Only the held-out set caught it.
- **Run 3: drafter, reviewer, template, policy, and router follow-up fixes.** The held-out set went from 15 to 19, and no email went from pass to fail. The only remaining overall miss is E29.

## Main failures found in run 1

Most misses came from the **router** (13 of 17 individual misses, including 5 of the 8 overall misses):

- **Category confusion between neighbouring labels.** An application status question (E06) went to `general` instead of `eligibility`. A thank-you note (E13) went to `credit_status` instead of `general`. A location question (E19) went to `eligibility` instead of `general`. A credit-sharing request (E28) went to `partner_vc` instead of `credit_status`.
- **Over-triggering on "sensitive."** E23 ("we just raised a seed round, can you approve us now?") was read as an instruction to the AI and stopped. It should have reached the drafter so the funding claim could be checked against the records.
- **Risk calibration.** The router can't see the records, so it rated likely denials low (E03, E08 and E26). It rated out-of-scope but harmless requests high (E21 and E25) and a routine referral check high (E22).

Drafter and reviewer:

- **Drafter, E29:** for a sender with no record, it gave a T10 hand-off. The key expects the T7 "how to apply" reply, which explains the criteria without promising anything.
- **Reviewer, E11:** it flagged a correct T7 reply because of the founder's reincorporation record, which the question didn't touch.
- **Reviewer, E22:** it flagged the standard T8 line "they'll reach out directly if anything applies" as overpromising. The answer key expected a send, but the reviewer was right that the wording reads like a promise. The fix is to reword T8, not to show the reviewer the templates.
- Many T10 hand-offs were flagged for not answering the question. The outcome was still correct, but they suggest T10 replies should name the topic and the team.

## Lessons learned

- **Fixes tuned to cases you've studied can break others, so keep a held-out set.** Run 2 looked perfect on the hand-run 10 while the held-out 20 stood still.
- **A rule added to one agent must be mirrored in every agent that checks the same thing.** After the drafter learned to send T7 to founders with no record, E29 moved from a drafter miss to a reviewer miss, because the reviewer had no matching rule.
- **A new general rule can override an older, more specific one.** In E04, a newer rule took precedence over the specific rule for prior-credit denials, so the drafter picked the wrong template.
- **Some misses were the answer key's fault, so calibrate the key before measuring improvement.** Otherwise a "fix" may only be teaching the agents to match a wrong answer.
- **Audit the transcripts before trusting the scores.** When an agent referenced a file it shouldn't have seen, I checked the transcripts. No agent read outside its allowed files.
- **Every outcome miss in every run failed safe.** All seven outcome misses across the three runs held a reply for a human that the key expected to be sent. No risky reply was ever sent.

## Caveat

By run 3, the misses in all 30 emails had been reviewed, so the held-out set is no longer truly unseen. The 95% held-out score is best read as an upper bound. The next step is to validate on fresh emails.

## Next steps

1. Add a reviewer rule for founders with no record (E29).
2. Resolve the rule collision so prior-credit denials use T4 (E04).
3. Tighten the router's risk rules (E07, E08).
4. Remove the Read tool from the router, which never needs files.
5. Validate on a fresh set of emails the prompts have never seen.
6. Compare Haiku and Sonnet for routing on accuracy, cost, and speed.
