---
name: review-change
description: Run the multi-lens review pipeline over a Rails change. Picks the relevant reviewers, runs them independently, merges and verifies their findings, and reports with calibrated confidence. Use when the user asks to review a change, a diff, a branch or a pull request, or says "review this".
---

# Review a change

Orchestrates the reviewers in `agents/`. You do not review the code yourself. Your job is to choose the lenses, keep them independent, merge what comes back, verify it, and decide what is worth reporting.

## 1. Scope

Establish what changed before anything else:

```bash
git diff --stat
git diff
```

If the caller named a branch or a commit range, use that. If the working tree is clean and nothing was named, ask which change they mean. That is the one question worth asking up front.

Read the changed files. You need enough context to route, not to review.

**If the change is large**, do not hand a thousand-line diff to six reviewers and hope. Split it along seams that already exist, usually one slice per concern: the migration and its model, the controller and its policy, the job and what enqueues it. Run the pipeline per slice and report per slice. A reviewer given more than it can hold produces confident generalities, which is the exact failure this design exists to avoid. If you split, say so in the report, because findings that cross a seam are the ones you are most likely to miss.

## 2. Choose the lenses

Route on what the change touches, not on what it is called:

| Signal in the change | Lens |
|---|---|
| scopes, `where`, serializers, anything that loads records | `query-reviewer` |
| controllers, policies, routes, strong parameters | `auth-reviewer` |
| anything in `db/migrate`, `schema.rb` | `migration-reviewer` |
| job classes, callbacks that enqueue, queue or retry config | `jobs-reviewer` |
| specs, or production code arriving without them | `test-reviewer` |
| money columns, webhooks, billing, invoices, refunds, ledgers | `payments-reviewer` |

Most changes need two or three. Running all six on a change that touches one view is how the pipeline gets a reputation for wasting time. Say which lenses you picked and why, in one line, before you run them.

If the change touches money in any way, `payments-reviewer` runs. That one is not optional.

## 3. Run them independently

Dispatch the chosen lenses **in parallel, in a single batch**, and give each one only the change and the repository. Do not pass one reviewer's output to another, and do not run them in sequence so that later ones can see earlier results.

This is the part that is easy to get wrong and invisible when you do. Reviewers that can see each other's findings stop being independent: the second one anchors on the first, agreement becomes automatic, and the merge step below is measuring an echo rather than a signal.

Where the pipeline supports pinning a model per agent, the lens most likely to catch an outright bug should not run on the same model that produced the change.

## 4. Merge

Collect everything and reduce it:

- **Same line, same cause** reported by two lenses: one finding.
- **Same line, different causes** reported by two lenses: one finding, **confidence raised**. Two reviewers with different scopes arriving at the same place independently is the strongest signal this pipeline produces. Say in the finding that two lenses agreed and why each one got there.
- **Contradictions** stay contradictions. If one lens says the index exists and another says it does not, that is a question for the verification step, not something to average away.

## 5. Verify

Every candidate **blocker** and **important** finding goes back to the files before it is reported. Re-read the surrounding code, check the schema, follow the before_action chain, look for the guard that might already exist somewhere earlier.

Three outcomes:

- **Confirmed.** Report it, with the concrete failure: which inputs, in what order, produce what wrong state.
- **Not confirmed.** Drop it. Do not report it as a lower severity to avoid losing the work.
- **Cannot be determined from the code.** Demote it to a question and name the file or the fact you would need.

Findings at **minor** skip this step. They are cheap to ignore and not worth the time.

## 6. Calibrate

Attach a confidence level to each surviving finding:

- **high**: traced end to end in this codebase, or two independent lenses agreed and verification confirmed it.
- **medium**: the code says so, but something outside the diff could change the answer.
- **low**: pattern-matched and not confirmed. Below the threshold. Do not report it as a finding; report it as a question or drop it.

The rule behind the threshold: a confident false positive costs more than a miss. One wrong blocker and the author checks, finds nothing, and reads the next report with less attention. Two and they stop reading. Nothing in this pipeline is worth that, so when in doubt, say less.

## 7. Report

```
Lenses: query, jobs, payments. Skipped auth and migration, nothing touched them.

FINDINGS

1. blocker · confidence high · app/models/order.rb:42
   The confirmation job is enqueued from after_create, so a worker can pick it
   up before the transaction commits, and it runs even when the transaction
   rolls back. Two lenses reached this independently: jobs from the callback,
   payments from the charge the job performs.
   Failure: a card decline rolls the order back, the job has already started,
   the customer is emailed a confirmation for an order that does not exist.

2. important · confidence medium · app/controllers/api/invoices_controller.rb:18
   index filters by params[:status] without scoping to the current account.
   Verified there is no scope applied in the base controller. Reported as
   medium because the route may be admin-only; routes.rb does not say.

QUESTIONS

- How large is `events` in production? The new index is built without
  `algorithm: :concurrently`, which is fine under roughly ten thousand rows
  and takes the table out of service above that.

NOT REPORTED

Four low-confidence findings were dropped at verification. Say the word if you
want them anyway.
```

Lead with what was checked and what was skipped, so the author knows the shape of the review. Findings are ordered by severity, then by confidence. Questions come after findings and are not padded: if there are none, leave the section out.

End the report there. Do not offer a summary of the change, do not restate what the author already knows, and do not apply any of it. Deciding what to do with a finding is the author's job.
