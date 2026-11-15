---
name: unslop-comments
description: Cleans up code comments in a diff or a set of files. It deletes comments that restate the code, narrate history, or excuse a workaround, removes commented-out code and lint or type suppressions that hide real bugs, and rewrites the few comments that stay into one plain sentence that says why. It flags code that needs a comment to be understood, so the code can be renamed or reshaped instead. Use before handing work over for review or opening a PR/MR, when the user says "unslop comments", "no comments", "clean up comments" or "remove AI comments", or when a diff written by an agent is full of explanatory comments. Works for TypeScript, JavaScript, Vue, PHP, CSS and any other language with comments.
---

# Unslop comments

Most comments an agent writes repeat the code, tell the story of the change, or apologise for a workaround. They go stale and the reader has to check them against the code anyway. Delete them. Keep a comment only when it says something the code cannot, and then write it as one plain sentence.

This skill changes comments. It does not change behaviour. Its only code changes are the small ones listed in step 4.

## 1. Scope

Use the files or diff the user names. Otherwise use the working tree against the merge base with the default branch:

```sh
git fetch origin
git diff $(git merge-base HEAD origin/HEAD)
```

Judge only comments that the scope adds or changes. A comment the change did not touch stays, unless the change made it false. Then fix or delete it.

## 2. Read with fresh eyes

The agent that wrote a comment is the one least able to see that it is slop. If you wrote this code in the current session and can spawn subagents, have one fresh read-only subagent do steps 3 and 5 on the scope and return its list, then apply the list yourself after you check it. Give it this file's path and the scope, nothing else. Without subagents, do it yourself and go comment by comment, not file by file.

## 3. Decide each comment

Every comment in scope is **delete**, **keep** or **rewrite**. When unsure, delete.

**Delete:**

- It says what the next line does: `// fetch the user`, `<!-- Header -->`, `/** Returns the total */` on `getTotal()`.
- It narrates the change or its history: `// changed to use X`, `// previously…`, `// new approach`, `// added for SJ-123`. That belongs in the commit message.
- It is a section banner or divider: `// ===== Helpers =====`, `// --- state ---`.
- It is commented-out code. Git has it.
- It is a TODO without an owner, a ticket or a concrete next step: `// TODO: improve this`.
- It explains a surprise in our own code: `// we have to call this twice because the store is not ready`. The comment dies and the code gets flagged (step 5), because the fix is a better name, a type or a different shape, not prose.
- It justifies itself at length: `IMPORTANT`, `do not remove`, `fine for now`, `too risky to change`. Read the code it guards. If the claim is not true on a live path today, delete it. If it is true and about our own code, delete it and flag the code.
- A doc comment on private or internal code that only repeats the name and the types.

**Keep:**

- License or legal headers.
- A constraint from something we cannot change: a browser bug, a vendor API, a protocol, a framework limitation. It should name the thing, and a link to the issue helps.
- A link to an issue, RFC or spec that explains a rule the code cannot express.
- Tool directives: `// prettier-ignore`, `/* @vite-ignore */`, `/* @__PURE__ */`, `// @ts-check`, PHPDoc generics and types the language cannot express (`@template`, `@var list<Foo>`, `@return array<string, int>`).
- Doc comments that define a public API: an exported function of a shared package, a public library method. Keep the contract, cut the narration.
- A comment the user asked for in this conversation or the project's rules file.

**Suppressions** (`eslint-disable`, `oxlint-disable`, `@ts-expect-error`, `@ts-ignore`, `@phpstan-ignore`, `noqa`, `@SuppressWarnings`): look up the rule. If it only enforces style or is known to misfire on this pattern, keep the suppression and give it a reason. If it catches real bugs (types, unsafe access, missing awaits, hook rules), remove the suppression and fix the code when the fix is small and inside the scope. Otherwise flag it.

**Rewrite** a kept comment when it is longer or vaguer than it needs to be (step 4).

## 4. Rewrite what stays

A kept comment is one sentence, two at most, in the language the codebase uses for comments:

- Say why, not what. `// Safari drops focus on the first click inside a dialog (WebKit 251234)`, not `// handle focus`.
- Name the outside cause and link it if a link exists.
- Use plain words. No "ensure", "leverage", "utilize", "crucial", "robust", "seamless", "comprehensive", "facilitate", "in order to", "note that", "it is important to".
- No hedging ("might potentially"), no reassurance ("this is safe because…" with no evidence), no first person ("we", "I") unless the codebase writes comments that way.
- No em dashes, no decorative emoji, no bold, no title case. Write whole sentences with their articles and verbs, no arrows or symbol shorthand.
- Active voice with a named actor: "the API returns dates in UTC", not "dates are returned in UTC".

**Code changes this skill may make**, only inside the scope and only when the comment it replaces disappears with them:

- Rename a local variable, parameter or private function so the deleted comment is no longer needed.
- Pull a magic value into a named constant.
- Delete a dead branch or unused code that a deleted comment was describing.
- Fix the code behind a removed suppression, as in step 3.

Anything larger (a new type, a different data flow, a split component) is a flag, not an edit.

## 5. Flags

A flag names a symbol inside the scope that needed prose to be understood, and the change that would make it obvious: rename, extract, type, or reshape. One line each, with `path:line`. Don't invent flags for code that reads fine without the comment.

## 6. Check and report

Run the project's linter and type checker on the touched files. Deleting a comment can expose a suppression that is now unused, and a rename can break an import.

Report in a few lines:

- Comments deleted, rewritten and kept, as counts per file. List each kept comment with the keep reason in a few words.
- Suppressions removed or kept, with the rule name.
- Code changes made under step 4.
- Flags from step 5.
