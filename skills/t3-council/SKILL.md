---
name: t3-council
description: Runs a council of AI agents from different vendors inside T3 Code (Claude, GPT, Gemini, Grok, Kimi, GLM, DeepSeek, …). It picks the strongest available model from each vendor across every enabled T3 provider and has them work independently, then check each other blind. It has two modes. Decide mode handles planning, such as an approach, design, architecture, a plan, root-cause hypotheses or a trade-off, and it ends in one recommendation with the dissent kept. Review mode handles a diff, branch, PR/MR or plan document, and it ends in a cross-verified list of findings to fix, consider or dismiss. Use it whenever the user, working in T3 Code, wants several models or providers to weigh in, debate, agree or review together. Typical asks are "council", "consortium", "ask several models", "second opinion from Gemini/GPT", "cross-provider review", "let the models argue it out", and the Czech "napříč providery" or "ať se domluví". Only inside T3 Code. Not for quick factual questions.
---

# T3 council

Several models from different vendors work the same task independently, check each other blind, and you, the chair, turn that into one answer. The value comes from model diversity. Models from one vendor share training data and blind spots, so a seat is one vendor, not one T3 provider.

## Modes

| | Decide (default) | Review |
|---|---|---|
| For | choosing an approach, design, architecture, a plan, a root-cause hypothesis or a trade-off before or during the work | a finished diff, branch, PR/MR or a written plan |
| Round 1 | each seat proposes an approach | each seat lists findings |
| Round 2 | each seat ranks the anonymized proposals and revises its own | each seat verifies the merged, anonymized findings |
| Result | one recommendation with the dissent kept | findings to fix, consider or dismiss, with a verdict |

Use review mode when the user points at changes or asks for a review, and decide mode otherwise. When it's unclear, decide mode is the safer default, because a review of an approach that is still open is just a decision.

## 0. Check that you are in T3 Code

The council needs the T3 tools `orchestrator_capabilities` and `delegate_task`, often deferred (load them with your tool search). If they don't exist, say this skill needs T3 Code and stop. A single-vendor council of your own subagents is a different thing, so offer it, but don't run it as if it were this.

If the task is trivial (a lookup, a one-line diff, a fix with an obvious answer), say a council is overkill and answer directly. Each seat is a full agent run, twice.

## 1. Write the brief

Every seat gets one brief, word for word. Seats see none of this conversation, so the brief is all they know.

**Both modes:**

- **Context.** Repo path, the files and docs worth reading (paths, not pasted dumps, unless the seat could not reach them), the project rules file (`AGENTS.md`, `CLAUDE.md`), what is already ruled out and why.
- **Rules.** Read-only: edit no file, commit nothing, install nothing, start no subagents, ask no questions. Work it out and answer. Answer in English, whatever language the user or your own settings use. Seats inherit their harness's language settings, and a seat that answers in another language is easy to spot in round 2.

**Decide mode:**

- **Question.** The decision to make, in one or two sentences.
- **Success criteria.** 3–6 concrete criteria "best" is judged by: correctness, migration cost, fits the existing architecture, and so on. You use the same criteria in step 5.
- **Answer shape.** Recommendation first, then the approach in steps, the risks, the alternatives considered and why each was rejected, and the open questions. Cite evidence as `path:line`. At most ~600 words.

**Review mode:**

- **Intent.** One paragraph on what the change is meant to do, from the user's words, commit messages, the PR/MR description and the code. If you can't tell, ask the user before spawning, because a reviewer who guesses the intent reports the wrong things.
- **Target.** How a seat gets the change itself: the exact command (`git diff <base>...HEAD`, plus `git diff` when uncommitted work counts), the PR/MR number and the CLI to read it, or the plan file's path. Seats run in the shared checkout, so they see the working tree as it is.
- **Rubric.** Correctness, security, breaking changes, data loss, concurrency, error handling, and violations of the project rules file. No style nits unless the rules file asks for them.
- **Answer shape.** At most ~10 findings, most severe first. Each one: `path:line`, severity (`blocker`, `major`, `minor`), the defect in one sentence, a concrete failure scenario (input or state, then the wrong outcome) and the evidence. End with "no findings" if there are none. A finding without a failure scenario is an opinion. Leave it out.

If the task is ambiguous enough that seats would answer different questions, ask the user before spawning.

## 2. Pick the seats

Call `orchestrator_capabilities` once (no `search`). From the result:

1. **Usable providers.** Keep those with `canRunCrossProviderChildTask: true` (your own provider needs only `canRunChildTask`). Note every disabled provider whose vendor will then be missing, such as "Codex is disabled, so no GPT seat. Enable it in T3 Code settings."
2. **Group models by vendor**, read from the model id and label: Anthropic (Opus, Sonnet, Fable, Haiku), OpenAI (GPT, Sol, o-series), Google (Gemini), xAI (Grok), Moonshot (Kimi), Zhipu (GLM), DeepSeek, Meta (Muse) and so on. A multi-vendor host such as Cursor, OpenCode or Pi contributes models to several vendors. A model whose vendor you can't tell counts as its own vendor.
3. **Strongest model per vendor.** Prefer the flagship tier over fast or small variants (`fast`, `flash`, `mini`, `lite`, `nano`, `haiku`) and the newest version number. If a provider reports `omittedModels` and its vendor's flagship may be among them, call `orchestrator_capabilities` with `search` for that vendor. When you can't tell which of two is stronger, pick one and say so in the roster.
4. **Provider per vendor.** When a vendor is reachable through several providers, prefer its own harness (Claude via the Claude provider, GPT via Codex, Grok via Grok, Gemini via Antigravity or Gemini CLI). Otherwise use a host such as Cursor.
5. **Size.** Default 4 seats, at most 6. Order: the frontier labs first (Anthropic, OpenAI, Google, xAI), then the strongest open-weight vendor (Kimi, GLM, DeepSeek, …) for diversity. A size, list of models or exclusion the user names wins.
6. **Effort.** Each provider names its reasoning option differently (`effort`, `reasoning_effort`, `reasoning`, `reasoningEffort`), so take the id and its values from the capabilities result, never from memory. Set `high`, or `xhigh` for a hard question when offered. Use `max` or `ultrathink` only when the user asks. They cost much more for little gain here.

Your own vendor gets a seat too, run as a fresh child, so the chair's framing doesn't leak into a seat. You stay the chair.

Show the roster as a table (seat, vendor, provider, model, effort, why this one) together with the missing vendors, then start right away. The user can stop or change it while round 1 runs. Ask first only when the user asked to approve the roster.

## 3. Round 1: independent work

Start every seat in one message, one `delegate_task` each:

- `task`: the brief from step 1, unchanged. Don't name the other seats, so nobody anchors on a vendor.
- `target`: `providerInstanceId`, `model`, and `options` with the effort from step 2.
- `role`: `"design"` in decide mode (`"research"` for a root-cause question), `"review"` in review mode. Also `mode: "async"` and a `title` such as `council r1 · Gemini`.
- `isolation`: leave the default `shared` so seats read the current checkout, uncommitted work included. Read-only is enforced only by the brief. Narrow runtime modes pause for approval rather than block writes, so don't use them. Use `"worktree"` only when seats must build a prototype. A worktree starts from the last commit, so uncommitted changes are missing there, and the brief has to say so.

Don't poll. Follow each result's `next` and wait for the completion notices, then read each `summary`. A seat that errors or returns nothing drops out. Report it and carry on with the rest, and don't swap in another vendor unasked. With fewer than 2 answers left, stop and report, because there is nothing to compare.

## 4. Round 2: blind cross-check

Strip anything that identifies a seat: "As Claude…", vendor names, signatures, and the answer's language when seats answered in different ones (translate all into one). Then start a fresh `delegate_task` per surviving seat, on the same model and effort, never a message to its round-1 thread. Each task is the brief from step 1 followed by the block for the mode.

**Decide mode.** Label the proposals A, B, C… in an order that isn't the seat order:

```text
Several independent experts answered this brief. Their proposals follow, anonymized. One of them may be yours. Judge them on the merits, not on authorship.

## Proposal A
...
## Proposal B
...

Read-only, as before. Reply with:
1. Ranking: best to worst, one line each on why.
2. Per proposal: its strongest idea and its most serious flaw. Check its claims against the code where it cites any.
3. Final recommendation: your best approach now, keeping what you learned from the others. Change your mind where they convinced you, and say what convinced you.
4. Remaining disagreement: the one point where you still think most of the others are wrong, with your evidence, or "none".
```

**Review mode.** Merge first: findings that describe the same defect at the same place become one, and you keep for yourself how many seats raised each. Number them F1, F2… sorted by severity, not by seat, and don't show how many seats raised a finding. That would turn the check into a vote. Past about 20 findings, send every blocker and major, plus the minors raised by two or more seats.

```text
Several independent reviewers reviewed this change. Their findings are merged and anonymized below. Some may be yours.

F1 [major] path:line: defect. Failure scenario: ... Evidence: ...
F2 ...

Read-only, as before. For every finding, read the code yourself and reply:
- F<n>: confirmed | refuted | unsure, with one line of evidence (path:line, or the input that does or does not fail).
Then list at most 3 new findings in the same shape as the brief, only ones the others made you see. Write "none" if there are none.
```

## 5. Settle disagreements and synthesize

1. **Verify.** Open the code behind every claim the result rests on, and behind every claim the seats contradict each other on. A council can agree on something false. Prefer file evidence over the majority.
2. **Aggregate.**
   - Decide mode: sum the rankings (Borda count is fine) without each seat's vote on its own proposal, because seats tend to rank their own first even blind. Note which ideas several seats adopted in round 2. An idea several vendors converged on independently is the strongest signal.
   - Review mode: per finding, how many seats raised it in round 1 and how the round-2 checks split. A finding several vendors raised independently and the rest confirmed is near certain. A lone finding the rest refuted needs your own check before you dismiss it, because the one model may be the only one that saw it.
3. **Optional round 3.** Only when a disagreement remains on a point that changes the result and the code can't settle it. Start one fresh `delegate_task` per dissenting seat, plus one seat from the majority, with that single point, both sides' arguments and their evidence, and ask for a verdict with evidence. One round at most. Then decide yourself.
4. **Judge.** You are the chair, not a vote counter. In decide mode, decide against the criteria from step 1. In review mode, sort every finding into **Fix** (a real defect that would block a PR), **Consider** (real, but the cost of fixing it now may not pay off), or **Dismissed** (wrong, out of scope, or no failure scenario survives). Say why whenever you overrule the majority.

## 6. Report

Answer in the user's language.

**Decide mode:**

1. **Recommendation.** The approach in a few steps, and why it wins on the criteria.
2. **Council.** The roster table with a status per seat (done, dropped and why) and the missing vendors.
3. **Consensus.** What most seats agreed on, with the seats that said it.
4. **Disagreements.** Each one, who stood where, and how you settled it.
5. **Strongest dissent.** The best argument against the recommendation, worth knowing even though it lost.
6. **Ranking.** Aggregated round-2 ranking of the round-1 proposals, with labels mapped back to models.
7. **Open questions.** Decisions only the user can make.

**Review mode:**

1. **Verdict.** Ship, ship after fixes, or hold, in one line with the reason.
2. **Council.** The roster table, as in decide mode.
3. **Fix.** A table: `path:line`, severity, the defect, the failure scenario, raised by (models), round-2 checks (confirmed / refuted / unsure).
4. **Consider.** Same table, with why it isn't a Fix.
5. **Dismissed.** One line each with the reason, so the user sees nothing was lost.
6. **Open questions.** Intent or product decisions only the user can make.

Link each seat's thread with the `link` from its `delegate_task` result. Don't paste raw answers. Change nothing in the repo unless the user asks. The council recommends or reports, and the user decides what to apply.
