---
name: t3-council
description: Runs a council of AI agents from different vendors inside T3 Code (Claude, GPT, Gemini, Grok, Kimi, GLM, DeepSeek, …). It picks the strongest available model from each vendor across every enabled T3 provider, lets them propose an approach independently, has each one review the others' anonymized proposals, and synthesizes one recommendation that shows where they agree, where they clash and the strongest dissent. Use it whenever the user, working in T3 Code, wants several models or providers to weigh in, debate or agree on the best approach, design, architecture, plan, root-cause hypothesis or trade-off. Typical asks are "council", "consortium", "ask several models", "second opinion from Gemini/GPT", "cross-provider", "let the models argue it out", and the Czech "napříč providery" or "ať se domluví". Not for reviewing a finished diff (use a code-review skill), not for quick factual questions, and only inside T3 Code.
---

# T3 council

Several models from different vendors work the same question independently, critique each other blind, and you, the chair, turn that into one recommendation. The value comes from model diversity. Models from one vendor share training data and blind spots, so a seat is one vendor, not one T3 provider.

## 0. Check that you are in T3 Code

The council needs the T3 tools `orchestrator_capabilities` and `delegate_task`, often deferred (load them with your tool search). If they don't exist, say this skill needs T3 Code and stop. A single-vendor council of your own subagents is a different thing, so offer it, but don't run it as if it were this.

If the request is trivial (a lookup, a one-file fix with an obvious answer), say a council is overkill and answer directly. Each seat is a full agent run, twice.

## 1. Frame the question

Write one brief that every seat gets word for word. Seats see none of this conversation, so the brief is all they know:

- **Question.** The decision to make, in one or two sentences.
- **Context.** Repo path, the files and docs worth reading (paths, not pasted dumps, unless the seat could not reach them), the project rules file (`AGENTS.md`, `CLAUDE.md`), what is already ruled out and why.
- **Constraints and success criteria.** 3–6 concrete criteria "best" is judged by: correctness, migration cost, fits the existing architecture, and so on. You use the same criteria in step 5.
- **Answer shape.** Recommendation first, then the approach in steps, the risks, the alternatives considered and why each was rejected, and the open questions. Cite evidence as `path:line`. At most ~600 words.
- **Rules.** Read-only: edit no file, commit nothing, install nothing, start no subagents, ask no questions. Work it out and answer.

If the question is ambiguous enough that seats would answer different questions, ask the user before spawning.

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

## 3. Round 1: independent proposals

Start every seat in one message, one `delegate_task` each:

- `task`: the brief from step 1, unchanged. Don't name the other seats, so nobody anchors on a vendor.
- `target`: `providerInstanceId`, `model`, and `options` with the effort from step 2.
- `role: "design"` (or `"research"` for a root-cause question), `mode: "async"`, a `title` such as `council r1 · Gemini`.
- `isolation`: leave the default `shared` so seats read the current checkout, uncommitted work included. Read-only is enforced only by the brief. Narrow runtime modes pause for approval rather than block writes, so don't use them. Use `"worktree"` only when seats must build a prototype. A worktree starts from the last commit, so uncommitted changes are missing there, and the brief has to say so.

Don't poll. Follow each result's `next` and wait for the completion notices, then read each `summary`. A seat that errors or returns nothing drops out. Report it and carry on with the rest, and don't swap in another vendor unasked. With fewer than 2 proposals left, stop and report, because there is nothing to compare.

## 4. Round 2: blind peer review

Strip anything that identifies a seat ("As Claude…", vendor names, signatures) and label the proposals A, B, C… in an order that isn't the seat order. Then start a fresh `delegate_task` per surviving seat, on the same model and effort, never a message to its round-1 thread:

```text
<the brief from step 1>

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

## 5. Settle disagreements and synthesize

1. **Verify.** Open the code behind every claim the recommendation rests on, and every claim the seats contradict each other on. A council can agree on something false. Prefer file evidence over the majority.
2. **Aggregate.** Sum the rankings (Borda count is fine), and note which ideas several seats adopted in round 2. An idea several vendors converged on independently is the strongest signal.
3. **Optional round 3.** Only when a disagreement remains on a point that changes the recommendation and the code can't settle it. Start one fresh `delegate_task` per dissenting seat, plus one seat from the majority, with that single point, both sides' arguments and their evidence, and ask for a verdict with evidence. One round at most. Then decide yourself.
4. **Judge.** You are the chair, not a vote counter. Decide against the criteria from step 1 and say why when you overrule the majority.

## 6. Report

Answer in the user's language, in this shape:

1. **Recommendation.** The approach in a few steps, and why it wins on the criteria.
2. **Council.** The roster table with a status per seat (done, dropped and why) and the missing vendors.
3. **Consensus.** What most seats agreed on, with the seats that said it.
4. **Disagreements.** Each one, who stood where, and how you settled it.
5. **Strongest dissent.** The best argument against the recommendation, worth knowing even though it lost.
6. **Ranking.** Aggregated round-2 ranking of the round-1 proposals, with labels mapped back to models.
7. **Open questions.** Decisions only the user can make.

Link each seat's thread with the `link` from its `delegate_task` result. Don't paste raw proposals. Change nothing in the repo unless the user asks. The council recommends, the user decides.
