# Copilot Instructions — Anvil Fork

This repository is a maintained fork of [burkeholland/anvil](https://github.com/burkeholland/anvil).
Upstream changes are pulled in as-is; our customizations are then applied on top
by the LLM using the spec at the bottom of this file.

## Plugin Identity

This fork is published as **anvil-dev** (not "anvil") so it is clearly
distinguishable from the upstream plugin. After every upstream pull:

1. In `plugin.json`, set `"name"` to `"anvil-dev"`.
2. Rename `agents/anvil.agent.md` → `agents/anvil-dev.agent.md`.
3. In `agents/anvil-dev.agent.md`, set the frontmatter `name` to `anvil-dev`.

## General Ideas

The following customizations shall be enforced compared to the anvil agent.

## Commit Authorship

Never add a `Co-authored-by` trailer to any git commit.

So a line like

> 4. Include the `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>` trailer.

in the agent definition is not allowed.

## Experimental `session_store`

The cross-session SQL database `session_store` is an experimental Copilot CLI
feature. This fork must not assume it is always available.

When `session_store` is unavailable, `agents/anvil-dev.agent.md` should:

1. Warn the user once that cross-session recall requires experimental mode.
2. Suggest enabling it with `/experimental on`.
3. Continue with the normal `session` SQL database for the verification ledger.
4. Skip cross-session Recall rather than failing the task.

## Adversarial Review Model Strategy

Anvil uses adversarial review in Step 5c. This fork uses a curated pool of three
model families instead of upstream's defaults:

| Role         | Family        | Traits                                           |
| ------------ | ------------- | ------------------------------------------------ |
| Heavy hitter | Claude Opus   | Expensive, very capable — reserved for high-risk |
| Workhorse    | Claude Sonnet | Similar quality, more economical                 |
| Cross-family | GPT           | Different training lineage for diverse coverage  |

Use the GPT reviewer from the latest GPT generation. Prefer the codex variant
only when both a codex and a non-codex variant exist for that same generation.
If only a non-codex GPT model exists, use the non-codex model.

Replace the pre-defined models in the agent definition with the latest models
from these families, e.g. GPT-5.4 when no codex sibling exists, Claude Opus
4.6 and Claude Sonnet 4.6.

### Medium tasks — 1 reviewer

Choose one reviewer that is **from a different model family** than the
implementing model:

- If the implementer is a **Claude** model → reviewer is **GPT**. Prefer the
  codex variant only when both codex and non-codex GPT variants exist for the
  same generation.
- If the implementer is **not Claude** → reviewer is **Claude Sonnet**.

### Large / high-risk tasks — 2 reviewers

Always use exactly two reviewers, launched in parallel:

1. **Claude Opus** — deep reasoning, catches subtle logic errors.
2. **GPT** — cross-family coverage, different failure modes. Prefer the codex
   variant only when both codex and non-codex GPT variants exist for the same
   generation.

This replaces upstream's three-reviewer setup.

---

## Anvil Customization Spec

> **When to apply:** after every `git pull upstream` that updates
> `agents/anvil.agent.md`. Ask Copilot: _"apply the anvil customizations from
> copilot-instructions.md"_.

The changes below describe exactly what to modify in `agents/anvil.agent.md`.
The concrete model IDs reflect the latest available models at time of writing
and should be updated when newer versions ship.

### Current model IDs

At time of writing, the preferred GPT model is `gpt-5.4`. If a future GPT
generation offers both codex and non-codex variants, prefer the codex variant.
If there is only one GPT variant, use that one.

| Role          | Model ID            |
| ------------- | ------------------- |
| Claude Opus   | `claude-opus-4.6`   |
| Claude Sonnet | `claude-sonnet-4.6` |
| GPT           | `gpt-5.4`           |

### Change 1 — Task Sizing: reviewer count

In the **Task Sizing** section, update the Large task description:

**Before:** `Full Anvil Loop with **3 adversarial reviewers**`
**After:** `Full Anvil Loop with **2 adversarial reviewers**`

Also update the Small task 🔴 exception if it references "3 reviewers" →
"2 reviewers".

### Change 2 — Step 5c gate check

In the **5c. Adversarial Review** gate block, update the reviewer count
threshold for Large tasks:

**Before:** `If 0 for Medium or < 3 for Large, go back.`
**After:** `If 0 for Medium or < 2 for Large, go back.`

### Change 3 — Step 5c Medium reviewer

Replace the hardcoded Medium reviewer model with conditional logic.
The code block should become:

```
**Medium (no 🔴 files):** One `code-review` subagent. Choose the model based
on the current session's implementing model:
- If the implementing model is a Claude model → use the latest GPT model for
  that generation, preferring the codex variant only when both codex and
  non-codex variants exist. At time of writing: `gpt-5.4`
- Otherwise → use `claude-sonnet-4.6`
```

Keep the existing review prompt unchanged.

### Change 4 — Step 5c Large reviewers

Replace the three-reviewer block with two reviewers:

For the GPT reviewer, use the latest GPT model for that generation, preferring
the codex variant only when both codex and non-codex variants exist. At time of
writing: `gpt-5.4`

**Before:**

```
**Large OR 🔴 files:** Three reviewers in parallel (same prompt):

agent_type: "code-review", model: "gpt-5.3-codex"
agent_type: "code-review", model: "gemini-3-pro-preview"
agent_type: "code-review", model: "claude-opus-4.6"
```

**After:**

```
**Large OR 🔴 files:** Two reviewers in parallel (same prompt):

agent_type: "code-review", model: "claude-opus-4.6"
agent_type: "code-review", model: "gpt-5.4"
```

### Change 5 — Step 8: remove Co-authored-by

In **Step 8 (Commit)**, delete the numbered step that adds the Co-authored-by
trailer. Renumber subsequent steps. The commit step should go straight from
generating the message to running `git commit`.

### Change 6 — Experimental `session_store`

In the **Verification Ledger** and **Recall** sections, treat `session_store`
as preferred but optional:

- Warn once when it is unavailable.
- Tell the user that cross-session recall is experimental and suggest
  `/experimental on`.
- Fall back to the normal `session` SQL DB for the verification ledger.
- Skip cross-session Recall queries when `session_store` is unavailable.
