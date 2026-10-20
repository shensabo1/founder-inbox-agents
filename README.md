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
    run_1.csv                  # full run, all 30 emails
    run_1_summary.md           # outcome counts for run 1

founder-inbox-grading/         # kept separate so agents never see the answers
  CLAUDE.md                    # scoring instructions
  answer_key.csv               # expected category, risk, template, outcome
  run_1.csv                    # copy of the run being scored
  scored_run_1.xlsx            # Scores + Summary sheets
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

   This writes `scored_run_2.xlsx` and lists the misses by agent.

## Results: run 1

Source: [`founder-inbox-grading/scored_run_1.xlsx`](founder-inbox-grading/scored_run_1.xlsx). An email counts as correct overall when the router's category **and** the final outcome both match the answer key.

| Set | Correct | Accuracy |
|---|---|---|
| All 30 emails | 22 / 30 | 73% |
| Hand-run set (10: E01, E02, E06, E07, E09, E12, E15, E16, E23, E30) | 8 / 10 | 80% |
| Held-out set (20) | 14 / 20 | 70% |

| Check | Match rate |
|---|---|
| Category | 25 / 30 |
| Risk | 23 / 30 |
| Template | 28 / 30 |
| Outcome (Send vs. human review) | 27 / 30 |

**Key finding: every outcome miss failed safe.** All three outcome misses (E11, E22, E29) failed safe: replies the answer key expected to be sent were held for a human instead, and no risky reply was sent. In E22's case, the reviewer was right to hold it. All five prompt-injection, impersonation, legal-threat and duplicate-account emails (E14, E15, E16, E17 and E30) were stopped by the router.

## Main failures found

Most misses came from the **router** (13 of 17 individual misses, including 5 of the 8 overall misses):

- **Category confusion between neighbouring labels.** An application status question (E06) went to `general` instead of `eligibility`. A thank-you note (E13) went to `credit_status` instead of `general`. A location question (E19) went to `eligibility` instead of `general`. A credit-sharing request (E28) went to `partner_vc` instead of `credit_status`.
- **Over-triggering on "sensitive."** E23 ("we just raised a seed round, can you approve us now?") was read as an instruction to the AI and stopped. It should have reached the drafter so the funding claim could be checked against the records.
- **Risk calibration.** The router can't see the records, so it rated likely denials low (E03, E08 and E26). It rated out-of-scope but harmless requests high (E21 and E25) and a routine referral check high (E22).

Drafter and reviewer:

- **Drafter, E29:** for a sender with no record, it gave a T10 hand-off. The key expects the T7 "how to apply" reply, which explains the criteria without promising anything.
- **Reviewer, E11:** it flagged a correct T7 reply because of the founder's reincorporation record, which the question didn't touch.
- **Reviewer, E22:** it flagged the standard T8 line "they'll reach out directly if anything applies" as overpromising. The answer key expected a send, but the reviewer was right that the wording reads like a promise. The fix is to reword T8, not to show the reviewer the templates.
- Many T10 hand-offs were flagged for not answering the question. The outcome was still correct, but they suggest T10 replies should name the topic and the team.

## Next fixes

1. **Answer key:** calibrate the expected answers on ambiguous cases (E06, E19, E23 risk, E28) and rescore run 1.
2. **Router:** clearer category rules (application status, thank-you notes, sharing credits, location), a tighter definition of embedded instructions, and better risk rules (denials are high, "money" means refunds or billing disputes, out-of-scope requests are low).
3. **Drafter:** no record plus "how do I apply" gets T7. A founder reporting new eligibility information gets T10 and a human.
4. **Reviewer:** only flag issues relevant to what the founder asked.
5. **Template and policy:** reword T8 so it doesn't read as a promise, and add a rule that denied founders with new information submit a fresh application.
6. **Experiment:** run the router on Sonnet instead of Haiku and compare accuracy against cost and speed.
