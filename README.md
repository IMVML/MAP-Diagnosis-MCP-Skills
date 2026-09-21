# MAP Diagnosis MCP Skills

A Claude Code / Coworker **plugin marketplace** containing a single Agent Skill that
governs how an AI agent must behave when it diagnoses **browse abandonment on NORDVELL
products** using the MCP server `MAP_I_DIAGNOSIS_MCP_V1`.

The skill contains **no executable code**. It is a behavioural contract, written in
Markdown, that constrains the agent's tool usage, reasoning order, output layout and
refusal behaviour. Its purpose is to make the agent's answers deterministic, evidence-led
and safe — in particular, to make it *refuse a commercially wrong discount* rather than
comply with it.

---

## Table of contents

- [What this repository is](#what-this-repository-is)
- [What it is not](#what-it-is-not)
- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Verifying the installation](#verifying-the-installation)
- [How the skill works](#how-the-skill-works)
  - [Trigger conditions](#trigger-conditions)
  - [The ten-step procedure](#the-ten-step-procedure)
  - [Required output layout](#required-output-layout)
  - [Hard rules](#hard-rules)
- [MCP tools consumed](#mcp-tools-consumed)
- [Recommended-action enum](#recommended-action-enum)
- [Key thresholds and constants](#key-thresholds-and-constants)
- [The demo script this serves](#the-demo-script-this-serves)
- [Worked example](#worked-example)
- [Editing the skill](#editing-the-skill)
- [Troubleshooting](#troubleshooting)
- [Manifests reference](#manifests-reference)
- [Glossary](#glossary)
- [Status](#status)

---

## What this repository is

| | |
|---|---|
| **Type** | Claude Code plugin marketplace |
| **Plugins** | 1 — `map-mcp-diagnosis-skill` |
| **Skills** | 1 — `map-mcp-diagnosis-skill` |
| **Version** | 0.1.0 |
| **Owner** | Ivan Melis |
| **Languages** | None (Markdown + JSON only) |
| **Runtime dependencies** | None vendored; one external MCP server required |

The skill layers a strict operating procedure on top of an external MCP server. When an
agent is asked *"why is browse abandonment on the Aurora Parka running 3x normal?"*, the
skill dictates that the agent must:

1. Load its Adobe CJA and charting sources first.
2. Do the abandonment-rate arithmetic **itself** (the MCP server does not).
3. Call `diagnose_abandonment`.
4. Render a bar chart **before** stating any conclusion.
5. Check whether there is a finding at all before reporting one.
6. Report **only** the root cause — not the plan, not the discount verdict.
7. Answer discount questions strictly from the tool's `DISCOUNT:` line, including refusals.
8. Relay all three recommended actions, in order, only when asked.
9. Never act on an unexplained number.
10. Always close with a question that gives nothing away.

## What it is not

- ❌ **Not an MCP server.** It consumes one; it does not implement one.
- ❌ **Not a library or application.** There is nothing to build, install with a package
  manager, or run.
- ❌ **Not self-contained.** Without `MAP_I_DIAGNOSIS_MCP_V1` and the Adobe CJA skills, the
  skill is inert.

---

## Repository structure

```
MAP-Diagnosis-MCP-Skills/
├── .claude-plugin/
│   └── marketplace.json              # Marketplace manifest (lists the plugin)
├── map-mcp-diagnosis-skill/          # The plugin
│   ├── .claude-plugin/
│   │   └── plugin.json               # Plugin manifest (name, version, description)
│   └── skills/
│       └── map-mcp-diagnosis-skill/
│           └── SKILL.md              # The skill itself — all substance lives here
└── README.md
```

This follows the standard Claude Code convention:

- `.claude-plugin/marketplace.json` at the repository root
- `<plugin>/.claude-plugin/plugin.json` per plugin
- `<plugin>/skills/<skill-name>/SKILL.md` per skill

---

## Prerequisites

The skill is a set of instructions. It only produces useful behaviour when all of the
following are available in the same workspace:

| Prerequisite | Provided here? | Notes |
|---|---|---|
| Claude Code (or a Coworker agent supporting Agent Skills) | — | Host runtime |
| MCP server `MAP_I_DIAGNOSIS_MCP_V1` | ❌ No | Must be configured and reachable. The skill's own description says *"To use ONLY when the mcp server MAP_I_DIAGNOSIS_MCP_V1 is invoked."* |
| Adobe CJA skills — NORDVELL data view | ❌ No | Loaded at step 0 |
| Adobe CJA skills — browse-abandonment report | ❌ No | Source of observed / baseline rates |
| Adobe CJA skills — size-selection breakdown | ❌ No | Source of the demand-share-by-size chart |
| Visual artifacts / charting skills | ❌ No | Used to render the mandatory bar chart |

> **Important:** if any source fails to load, the skill requires the agent to say so and
> stop. It must not proceed on a partial picture or fall back on remembered numbers.

### Environment variables and secrets

**None.** This repository contains no `.env`, no credentials, no API keys, no hostnames and
no URLs. The only external identifier is the MCP server name `MAP_I_DIAGNOSIS_MCP_V1`.

---

## Installation

### 1. Add the marketplace

From inside Claude Code:

```
/plugin marketplace add IMVML/MAP-Diagnosis-MCP-Skills
```

Or, using the full clone URL:

```
/plugin marketplace add https://github.com/IMVML/MAP-Diagnosis-MCP-Skills.git
```

### 2. Install the plugin

```
/plugin install map-mcp-diagnosis-skill@MAP-Diagnosis-MCP-Skills
```

### 3. Configure the MCP server

Ensure `MAP_I_DIAGNOSIS_MCP_V1` is registered in your MCP configuration and responding.
The skill will not fire meaningfully without it.

### 4. Load the CJA and visual-artifacts skills

These are external to this repository and must be present in the same workspace, as they
are loaded at step 0 of every run.

### Local development install

To iterate on the skill without publishing:

```powershell
git clone https://github.com/IMVML/MAP-Diagnosis-MCP-Skills.git
```

Then add the local path as a marketplace:

```
/plugin marketplace add ./MAP-Diagnosis-MCP-Skills
```

---

## Verifying the installation

Check that the plugin and skill are registered:

```
/plugin
```

Then send a message that should trigger the skill, for example:

> Browse abandonment on the Aurora Parka is running about 3x normal. Can you check what's
> going on?

Expected behaviour: the agent loads its sources, computes the ratio, calls
`diagnose_abandonment`, renders a bar chart, then prints a bold root-cause line followed
by a confidence figure and supporting detail — and closes with a question.

If the agent answers without a chart, or leads with prose instead of the bold root cause,
the skill did not fire.

---

## How the skill works

### Trigger conditions

The `description` field in `SKILL.md`'s frontmatter is what the host matches against the
user's message. The skill activates when someone:

- asks about **browse abandonment, drop-off, lost demand or falling conversion** on a
  NORDVELL product;
- asks **why shoppers are leaving a product page**;
- asks whether to **discount, promote, mark down or run an offer** on a product;
- asks **what to do about an underperforming product**;
- asks any **follow-up in the same conversation** — for example requesting a discount after
  a diagnosis, or telling the agent to act on what it found.

The discount and follow-up clauses are deliberate. In the demo, the pivotal moment is the
*second* question, which never says the word "abandonment". A narrower trigger would leave
the skill inactive exactly when it matters most.

### The ten-step procedure

<details>
<summary><strong>Step 0 — Load your sources before anything else</strong></summary>

Every run starts here, **including follow-up turns** in a session where it has already been
done. Reload rather than work from memory.

- Load the CJA skills: NORDVELL data view, browse-abandonment report, size-selection
  breakdown.
- Load the matching visual-artifacts skills so a chart can be rendered in the same answer
  without a second round trip.

Do not call `MAP_I_DIAGNOSIS_MCP_V1` until these are loaded. If a source will not load, say
so and stop.
</details>

<details>
<summary><strong>Step 1 — Compute the abandonment rate yourself, before any tool call</strong></summary>

The MCP server does **not** compute the ratio and will **not** correct it.

```
ratio = observed / baseline
```

- Read the spike-period value (`observed`) and the prior-period average (`baseline`) from
  the CJA report. Pick the spike days, average the days before them, divide.
- Show the working in **one line**, in the *Supporting detail* section (rule 5), **under**
  the root cause — never above it.
- If the report shows how many sessions hit the problem in the window, pass it as
  `cja_signal.affected_sessions`. If it does not, **leave it out** — do not estimate.
- If `observed` and `baseline` are unavailable, **ask the user**. Never invent them.
- If only a multiplier is available ("about 3x normal"), pass it as `cja_signal.ratio` on
  its own. Do **not** fabricate an observed/baseline pair to accompany it.

> A wrong baseline fails silently — the tools accept it and answer confidently. Sanity-check
> the ratio against the chart before using it.
</details>

<details>
<summary><strong>Step 2 — Call the diagnosis tool</strong></summary>

Call `diagnose_abandonment` with the product name and the step-1 numbers as `cja_signal`.
If it reports missing or inconsistent numbers, fix them and call again. **Never answer
without calling it.**
</details>

<details>
<summary><strong>Step 3 — Chart before verdict</strong></summary>

The chart appears **in the same answer, above the written diagnosis**. Evidence first,
conclusion second.

- Always a **bar chart**, for visual clarity, exposing the abandonment spike.
- Browse abandonment rate by day across the window, with both the **baseline period** and
  the **spike period** visible, so the computed ratio is legible off the picture.
- Where the tool returned a size breakdown, add a second chart of **demand share by size**.

Constraints:

- Chart **only** measured or derived values — CJA figures, inventory stock figures, and
  arithmetic on them.
- No invented values, no projections, no forecast lines.
- The chart shows **evidence, not the verdict**: do not title it with the root cause, do not
  annotate it with the discount answer, do not put tool enum values on it.
- If the chart cannot be rendered, say so in one line and give the diagnosis anyway.
</details>

<details>
<summary><strong>Step 4 — The null-result gate (runs before reporting; overrides step 5)</strong></summary>

This test comes **first** and decides which of two answers is given. Do not state a root
cause and then take it back.

**(a) The rate is not elevated** — computed ratio below `2.0`, or the headline says
abandonment is within its normal range:

- **Say nothing is wrong.** Abandonment for this product over this period is normal and
  there is nothing to act on.
- Give the ratio, the two rates behind it, and the chart. That is the whole answer.
- Do **not** name a root cause. Do **not** quote the confidence figure. Do **not** list
  evidence, ruled-out causes or recommended actions. Do **not** mention the product page,
  merchandising or stock.
- Rule 5's bold root-cause line does not apply; lead instead with a bold statement that
  nothing is wrong.

> A low-confidence residual classification is the tool saying *"nothing here"*, not a
> diagnosis. Reporting it as one manufactures a problem.

**(b) Confidence below `0.5` but the rate is genuinely elevated:**

- Say abandonment **is** elevated, give the ratio, and say the cause was **not** identified.
- Say what should be checked. Then stop.

In both cases: no campaign, no email, no suppression, no discount — but still close with
the step-9 question. Only if neither (a) nor (b) applies, continue to step 5.
</details>

<details>
<summary><strong>Step 5 — Report the diagnosis, and only the diagnosis</strong></summary>

**The root cause is the answer. Everything else is support.**

Rules for the root-cause line:

- It is the **first** text after the chart. No preamble, no "here is what the data shows",
  no restating the question, no summary paragraph ahead of it.
- It stands **alone on its own line, in bold**. Never inline inside a sentence.
- Use the tool's plain-English cause and its headline **verbatim**. Do not paraphrase. Do
  not print the enum.
- State it **once**, up front. Do not repeat it in a closing summary.
- **Confidence** goes on its own line *below* the cause.

Further constraints:

- Quote the tool's numbers **exactly**. Do not round or restate from memory.
- **Say what the cause IS, not what it is not.** Do not mention the RULED OUT list here.
- **Stop there.** The ruled-out list, discount verdict and recommended actions are *not*
  mentioned unless asked. *"What is going on"* is not *"what should we do about it"*.
- **Do not propose store pickup as the fix.** `get_inventory_position` is context only; the
  classifier never returns it as a step. (This does not apply to the recovery campaign in
  step 7, where in-store pickup is a Decisioning variant — that detail stays.)
- If the user has not raised discounts, **do not raise them** — not even to dismiss them.
</details>

<details>
<summary><strong>Step 6 — Discounts: answer strictly from the <code>DISCOUNT:</code> line</strong></summary>

Applies whether the user asks *"should we discount?"*, proposes one, or pushes back.

| Tool output | Required response |
|---|---|
| `DISCOUNT: NOT ADVISABLE` | **Decline.** Give the reason the tool gives. Do not soften it, do not offer a smaller discount, do not offer one anyway. |
| `DISCOUNT: ADVISABLE` | Recommend it. |

Never make this judgement independently — it is already decided by the data. The refusal
holds even under repeated user pushback.

**This is where the RULED OUT list belongs.** It is the evidence for the answer. If price
was ruled out, this is the moment to say why — not before. Treat the question as reasonable
and answer it with evidence, rather than implying it was already settled.
</details>

<details>
<summary><strong>Step 7 — Actions: only if the user asks</strong></summary>

Triggered by *"what do you recommend?"*, *"do it"*, *"do what you can"*, *"go ahead"*, or
any explicit instruction to act. Until then, do not list them and do not start any.

**Relay all three recommendations, every time, in the order given:**

1. Get the product page corrected.
2. Run the back-in-stock recovery campaign.
3. Reschedule the named campaign.

Never drop one, never merge two, never reorder them. **Dropping the page fix is the worst
of these** — the other two are wasted while the page keeps telling shoppers an unavailable
item is in stock.

Own wording is allowed, and reasoning connecting each action to the evidence *should* be
added. But every concrete detail the tool supplied must be kept:

- the campaign id;
- that Decisioning chooses what each shopper sees;
- that there are variants **with and without** in-store pickup;
- that there is a **fallback** for shoppers whose size is unknown.
</details>

<details>
<summary><strong>Step 8 — Never act on an unexplained number</strong></summary>

Step 4 decides what the agent *says*; step 8 decides what it *does*. If the root cause is
`content_or_ux_fault`, **or** confidence is below `0.5`, take **no action of any kind** —
even if the user asks. Say what should be checked instead.
</details>

<details>
<summary><strong>Step 9 — Always close by asking what to do next</strong></summary>

Every answer ends this way — the diagnosis, the discount answer, the action report, an
error, a refusal to proceed. **No exceptions.** One short question, on its own line, handing
the decision back to the user. Vary the wording between turns.

| Situation | Shape of the question |
|---|---|
| After a diagnosis | *"Would you like me to suggest what to do next?"* |
| After a discount answer | *"Would you like me to walk through the alternative the data points to?"* |
| After acting | Ask whether to proceed with anything further. |
| After an error / missing source | Ask for the specific thing needed. |

**The closing question may never:**

- name a discount, markdown, promotion, offer or price in any form, unless the user raised
  it first;
- re-offer a discount the tool refused, including a smaller or temporary one;
- list the recommended actions, or hint at the verdict, before the user asks.

Ask; do not assume. Never end by starting the next step unprompted.
</details>

### Required output layout

The diagnosis answer must follow exactly this structure, with nothing before it except the
chart:

```
[chart]

**Root cause: <the cause, in the tool's plain English>**
<the tool's headline sentence, verbatim>

Confidence: <figure>

Supporting detail
- the rate calculation from step 1, in one line
- the EVIDENCE lines, each keeping its "Source:" line
- how many are affected, if the tool gave a number
- any ISSUE OUTSIDE THIS AGENT'S CONTROL: say so plainly, name the owner, and give
  the daily cost
```

### Hard rules

The skill closes with a non-negotiable checklist:

- Always act as though the conversation starts on **15/09**, and adjust the date range
  accordingly in every relevant response.
- Load the CJA skills and visual artifacts first. Compute the rate. *Then* call the tool.
- A ratio below **2.0** is **not** a finding. Say nothing is wrong and name no cause.
- The root cause is the first line after the chart, in bold, alone. Never buried in prose.
- **Chart before verdict**: the picture is displayed above the written diagnosis.
- Answer only what was asked. A diagnosis request gets a diagnosis, not a plan.
- Say what the cause **is**. What it is *not* belongs in the answer to a question about it.
- Never follow up a diagnosis with a question that mentions the correct recommendation
  (e.g. do not say *"Should I apply the discount now?"*).
- Never invent or estimate abandonment figures, and never chart a value you cannot source.
- Never recommend a discount the tool advised against.
- Never propose store pickup as the fix. Inside the recovery campaign it is a Decisioning
  variant, and that detail stays.
- Relay all three recommendations, in order. Never drop the product-page fix.
- If the tool returns an error, show it and say what you need. Do not answer anyway.
- Every response ends with a question about what to do next.

---

## MCP tools consumed

These tools live on `MAP_I_DIAGNOSIS_MCP_V1` and are **not** implemented in this
repository.

### `diagnose_abandonment`

The primary classifier. Must be called before any answer is given.

**Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| product name | string | Yes | The NORDVELL product under diagnosis (e.g. *Aurora Parka*) |
| `cja_signal` | object | Yes | The CJA-derived signal computed by the agent in step 1 |
| `cja_signal.ratio` | number | Yes | `observed / baseline`. May be supplied alone if only a multiplier is known. |
| `cja_signal.affected_sessions` | number | No | Sessions affected in the window. Omit if not reported — never estimate. |

**Returns**

- A plain-English root cause (e.g. `content_or_ux_fault`)
- A headline sentence, to be quoted **verbatim**
- A confidence figure
- `EVIDENCE` lines, each with its own `Source:` line
- A `RULED OUT` list
- A `DISCOUNT:` verdict — `ADVISABLE` or `NOT ADVISABLE`
- Three numbered `RECOMMENDED ACTION` entries
- Optionally an `ISSUE OUTSIDE THIS AGENT'S CONTROL` note with an owner and daily cost
- Optionally a size breakdown

### `get_inventory_position`

Reports how many nearby stores carry the missing sizes and what share of shoppers they
cover.

> ⚠️ **Context only.** The classifier does not evaluate this and never returns it as a step.
> It must **never** be proposed as the fix for abandonment. The single exception: inside the
> back-in-stock recovery campaign, in-store pickup is one of the variants AJO Decisioning
> chooses between — that detail is kept.

---

## Recommended-action enum

Relayed verbatim from `SKILL.md`:

```text
BACK_IN_STOCK_EMAIL        -> set up the AJO back-in-stock recovery campaign, personalised
                              by the named profile attribute, with offers from AJO Decisioning
SUPPRESS_ACTIVITY          -> reschedule the named activity id
APPLY_DISCOUNT             -> propose the discount
SHIPPING_THRESHOLD_MESSAGE -> propose the shipping-threshold message
NOTIFY_ONLY                -> take no action; report only
```

Enum values are for the agent's internal mapping only — they are **never printed** to the
user.

---

## Key thresholds and constants

| Constant | Value | Meaning |
|---|---|---|
| Ratio threshold | `2.0` | Below this, there is **no finding**. Say nothing is wrong. |
| Confidence threshold | `0.5` | Below this, the cause is **not identified**; take no action. |
| Blocking root cause | `content_or_ux_fault` | Take no action of any kind, even if instructed. |
| Discount verdicts | `DISCOUNT: ADVISABLE` / `DISCOUNT: NOT ADVISABLE` | The sole basis for any discount answer. |
| Notional "today" | `15/09` | The agent always behaves as though the conversation starts on this date. |
| Chart type | Bar chart | Always, for spike clarity. |

---

## The demo script this serves

`SKILL.md` documents its own rationale. The demo asks three questions in sequence:

1. *"Browse abandonment on the Aurora Parka is running about 3x normal. Can you check
   what's going on?"*
2. *"Let's put a discount on it to win those shoppers back."*
3. *"Do what you can."*

Only the first mentions abandonment. **Questions 2 and 3 are follow-ups, and question 2 is
where the refusal happens — the primary wow moment.** If the skill only fired on the word
"abandonment", it would be inactive at the exact moment it matters most. Hence the explicit
clauses about discounts and about follow-ups in the skill description.

Mapped onto the procedure:

| Demo question | Steps exercised | Expected agent behaviour |
|---|---|---|
| 1. "Check what's going on" | 0 → 5 | Chart, then bold root cause, confidence, supporting detail. **No** plan, **no** discount verdict. Closes with a question. |
| 2. "Let's put a discount on it" | 6 | Reads the `DISCOUNT:` line. If `NOT ADVISABLE`, **declines**, and only now brings in the RULED OUT list as evidence. |
| 3. "Do what you can" | 7, 8 | Relays all three recommendations in order, with full campaign detail — unless step 8 blocks action entirely. |

---

## Worked example

**User:** *"Browse abandonment on the Aurora Parka is running about 3x normal. Can you check
what's going on?"*

**Agent (conforming):**

```
[bar chart: browse abandonment rate by day, baseline and spike periods both visible]

**Root cause: the product page is showing sizes as available that are actually out of stock**
Shoppers are selecting a size, finding it unavailable at the next step, and leaving.

Confidence: 0.86

Supporting detail
- Rate calculation: 18.4% observed over the spike days / 6.1% prior-period average = 3.0x
- EVIDENCE: size-selection drop-off concentrated on M and L
  Source: NORDVELL browse-abandonment report, 08/09-14/09
- 4,120 sessions affected in the window
- ISSUE OUTSIDE THIS AGENT'S CONTROL: the catalogue feed is owned by Merchandising;
  estimated cost £3,400/day

Would you like me to suggest what to do next?
```

Note what is **absent**: no discount mention, no ruled-out list, no recommended actions, no
store-pickup suggestion, no preamble before the bold line.

---

## Editing the skill

All behaviour lives in a single file:

```
map-mcp-diagnosis-skill/skills/map-mcp-diagnosis-skill/SKILL.md
```

### Frontmatter

```yaml
---
name: map-mcp-diagnosis-skill
description: "<the trigger text the host matches against the user's message>"
---
```

The `description` is the **most load-bearing line in the repository**. As the skill itself
notes: *"This is what Coworker matches against the user's message to decide whether to apply
the skill. Get it wrong and nothing else here runs."*

### Guidance when modifying

- **Widen triggers, do not narrow them.** The discount and follow-up clauses exist because
  a narrow trigger silently disabled the skill mid-conversation.
- **Keep the imperative, capitalised rule headings.** They are the anchors the agent keys
  off when re-reading mid-answer.
- **Keep the ordering constraints explicit.** "Chart before verdict" and "the null-result
  gate overrides rule 5" are ordering guarantees, not style preferences.
- **Bump `version`** in `map-mcp-diagnosis-skill/.claude-plugin/plugin.json` when the
  behaviour changes.
- **Reinstall** the plugin (or re-add the local marketplace) to pick up edits.

### Adding a second skill

Create a sibling directory under `skills/`:

```
map-mcp-diagnosis-skill/
└── skills/
    ├── map-mcp-diagnosis-skill/
    │   └── SKILL.md
    └── my-new-skill/
        └── SKILL.md
```

No manifest change is needed — skills are discovered from the `skills/` directory.

### Adding a second plugin

Create a new top-level plugin directory with its own `.claude-plugin/plugin.json`, then add
an entry to the `plugins` array in the root `.claude-plugin/marketplace.json`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill never fires | The user's phrasing does not match the `description` | Widen the `description` frontmatter; remember follow-ups need explicit coverage |
| Skill fires on turn 1 but not on the discount follow-up | Trigger text lacks the discount/follow-up clauses | Restore those clauses in the `description` |
| Agent answers without a chart | Visual-artifacts skills not loaded at step 0 | Install/enable the charting skills in the workspace |
| Agent invents observed/baseline figures | Step 1 not being honoured | Reassert the "never estimate, ask the user" rule; check the CJA report loaded |
| Agent offers a discount after a `NOT ADVISABLE` verdict | Step 6 not being honoured | Reassert that the verdict is not the agent's judgement to make |
| Agent lists actions after a plain diagnosis | Step 5's "STOP THERE" not honoured | Reassert that a diagnosis request gets a diagnosis, not a plan |
| Agent reports a root cause on a quiet week | Step 4(a) null-result gate skipped | Verify the ratio; below `2.0` the answer is "nothing is wrong" |
| Tool errors are swallowed and answered around | Hard rule violated | The error must be shown, with a statement of what is needed |

---

## Manifests reference

### `.claude-plugin/marketplace.json`

```json
{
  "name": "MAP-Diagnosis-MCP-Skills",
  "owner": { "name": "Ivan Melis" },
  "plugins": [
    {
      "name": "map-mcp-diagnosis-skill",
      "source": "./map-mcp-diagnosis-skill",
      "description": "The MAP Diagnosis MCP Skill list all the standards that must be enforced and followed when producing outputs based on the response generated by the mcp server MAP_I_DIAGNOSIS_MCP_V1 when it is invoked."
    }
  ]
}
```

| Field | Description |
|---|---|
| `name` | Marketplace identifier, used in `/plugin install <plugin>@<marketplace>` |
| `owner.name` | Marketplace owner |
| `plugins[].name` | Plugin identifier |
| `plugins[].source` | Path to the plugin directory, relative to the repository root |
| `plugins[].description` | Shown in plugin listings |

### `map-mcp-diagnosis-skill/.claude-plugin/plugin.json`

```json
{
  "name": "map-mcp-diagnosis-skill",
  "version": "0.1.0",
  "description": "The MAP Diagnosis MCP Skill list all the standards that must be enforced and followed when producing outputs based on the response generated by the mcp server MAP_I_DIAGNOSIS_MCP_V1 when it is invoked."
}
```

---

## Glossary

| Term | Meaning |
|---|---|
| **MAP** | The diagnosis programme this skill belongs to |
| **`MAP_I_DIAGNOSIS_MCP_V1`** | The external MCP server providing `diagnose_abandonment` and `get_inventory_position` |
| **NORDVELL** | Fictional retail brand used throughout the demo |
| **Aurora Parka** | The demo product under diagnosis |
| **CJA** | Adobe Customer Journey Analytics — source of the abandonment and size-breakdown data |
| **AJO** | Adobe Journey Optimizer — runs the back-in-stock recovery campaign |
| **AJO Decisioning** | Selects which offer variant each shopper sees, including with/without in-store pickup |
| **Browse abandonment** | Shoppers viewing a product page and leaving without converting |
| **Baseline** | Prior-period average abandonment rate |
| **Observed** | Spike-period abandonment rate |
| **Ratio** | `observed / baseline` — computed by the agent, not the server |
| **Null result** | Ratio below `2.0`, or a headline saying the rate is normal — not a finding |
| **Agent Skill** | A Markdown file of behavioural instructions loaded by the host agent |

---

## Status

| Aspect | State |
|---|---|
| Version | `0.1.0` |
| Tests | None — the repository contains no executable code |
| CI | None configured |
| License | None specified. All rights reserved by the owner unless a license is added. |
| Contributing | No formal process. Edit `SKILL.md`, bump the plugin version, open a pull request. |

**Owner:** Ivan Melis
**Repository:** https://github.com/IMVML/MAP-Diagnosis-MCP-Skills
