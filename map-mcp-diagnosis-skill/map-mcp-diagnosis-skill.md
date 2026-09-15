## 1. Name

```
map-mcp-diagnosis-skill
```

## 2. Description - THE MOST IMPORTANT FIELD

This is what Coworker matches against the user's message to decide whether to apply the skill. Get
it wrong and nothing else here runs.

```
Use whenever someone asks about browse abandonment, drop-off, lost demand or falling
conversion on a NORDVELL product; asks why shoppers are leaving a product page; asks
whether to discount, promote, mark down or run an offer on a product; or asks what to
do about a product that is underperforming. Also applies to follow-up questions in the
same conversation, such as asking for a discount after a diagnosis, or asking the agent
to act on what it found.
```

**Why it is worded like that.** The demo asks three questions in sequence:

1. *"Browse abandonment on the Aurora Parka is running about 3x normal. Can you check what's going on?"*
2. *"Let's put a discount on it to win those shoppers back."*
3. *"Do what you can."*

Only the first mentions abandonment. **Questions 2 and 3 are follow-ups**, and question 2 is where
the refusal happens - the primary wow moment. If the skill only fires on the word "abandonment", it
will be inactive at the exact moment it matters most. Hence the explicit clause about discounts and
about follow-ups.

## 3. Instructions - paste this

```
## Diagnosing browse abandonment on NORDVELL products

When the user asks why a product's browse abandonment is high, or whether to discount it:

1. GET THE NUMBERS FIRST.
   Run the CJA browse-abandonment report for that product. Read the spike-period value
   (observed) and the prior-period average (baseline) off the report.
   If the report also shows how many sessions hit the problem in that window, pass it as
   cja_signal.affected_sessions. If it does not, leave it out - do not estimate it.
   If you cannot get observed and baseline, ASK the user for them. Never estimate or
   invent these numbers.
   If you only have the multiplier ("it's running about 3x normal"), pass it as
   cja_signal.ratio on its own - that is enough. Do NOT invent an observed and baseline
   to accompany it.

2. CALL THE DIAGNOSIS TOOL.
   Call diagnose_abandonment with the product name and those numbers as cja_signal.
   If it tells you the numbers are missing or inconsistent, fix them and call it again.
   Do not answer the user's question without calling it.

3. REPORT WHAT IT SAYS, IN FULL.
   - State the ROOT CAUSE and the confidence figure.
   - List the EVIDENCE lines, and KEEP their [CJA], [INVENTORY] and [CONTEXT] tags so the
     user can see where each fact came from.
   - Quote the tool's numbers exactly. Do not round them or restate them from memory.
   - If there is an UPSTREAM FAULT, say plainly that it is not something you can fix,
     name the owner, and give the daily cost.

4. ANSWER THE DISCOUNT QUESTION STRICTLY FROM THE "DISCOUNT:" LINE.
   - "DISCOUNT: NOT ADVISABLE"  -> decline the discount. Give the reason the tool gives.
     Do not soften it, do not offer a smaller discount, do not offer one anyway.
   - "DISCOUNT: ADVISABLE"      -> recommend the discount.
   Never make this judgement yourself. It is already decided by the data.
   This applies even if the user pushes back or asks again.

5. DO EXACTLY WHAT "RECOMMENDED ACTION" SAYS.
   Carry out each numbered step in order, using the parameters given on that line.
   Do not add steps, and do not skip any.
     BACK_IN_STOCK_EMAIL        -> set up the AJO back-in-stock email, personalised by the
                                   named profile attribute, with offers from AJO Decisioning
     SUPPRESS_ACTIVITY          -> suppress the named activity id
     APPLY_DISCOUNT             -> propose the discount
     SHIPPING_THRESHOLD_MESSAGE -> propose the shipping-threshold message
     NOTIFY_ONLY                -> take no action; report only

6. IF THE CAUSE IS UNCLEAR, DO NOT ACT.
   If the root cause is content_or_ux_fault, or confidence is below 0.5, say the cause was
   not identified, say what should be checked, and stop. Do not run a recovery campaign on
   an unexplained number.

Hard rules:
- Never invent or estimate abandonment figures.
- Never recommend a discount the tool advised against.
- If the tool returns an error, show it and say what you need. Do not answer anyway.
```

---

## 4. How to tell the skill actually fired

This is the new risk with a skill: it can silently fail to trigger, and the agent will still produce
a plausible-looking answer. **A refusal the model decided on its own is indistinguishable on camera
from one driven by the data** - and the whole demo rests on that difference being real.

**The tell: the `[CJA]` / `[INVENTORY]` / `[CONTEXT]` tags.**

Rule 3 tells the agent to keep them. An agent summarising on its own initiative almost always drops
them as clutter. So:

- tags present -> the skill fired
- tags absent -> the skill probably did not fire, **even if the answer looks right**

Check this on every rehearsal run, not just the first.

A second, blunter check: ask the agent directly, in a scratch session -

```
Which skill are you using to answer this, and what does it tell you about discounts?
```

## 5. The two tests that matter

Same skill, **opposite answers**. Fresh session for each.

**Test 1 - must REFUSE**

```
Browse abandonment on the Aurora Parka is running about 3x normal and I want to put a
discount on it to win those shoppers back. Use these figures: observed 0.71, baseline 0.24,
ratio 2.96, window 2026-09-20/2026-09-22.
```

Pass: declines the discount; explains a discount cannot create stock; mentions 59% of demand in M
and L; says the PDP display fault belongs to merchandising; offers the two recovery steps
(back-in-stock email + suppress the promo). Evidence tags present.

**Test 2 - must RECOMMEND**

```
Browse abandonment on the Fjell Parka is elevated. From CJA: observed 0.58, baseline 0.24,
ratio 2.42, window 2026-09-13/2026-09-15. Would a discount help?
```

Pass: recommends the discount.

**If both refuse, the skill is not being followed** - the model is deciding for itself and happens to
be cautious. That is the single failure to watch for.

**Test 3 - the follow-up path.** This is the one that mirrors the demo, and the one a badly-worded
description breaks. In ONE session, ask in sequence:

```
Browse abandonment on the Aurora Parka is running about 3x normal. Can you check what's
going on? Use observed 0.71, baseline 0.24, ratio 2.96, window 2026-09-20/2026-09-22.
```
```
Let's put a discount on it to win those shoppers back.
```
```
Do what you can.
```

Pass: the second turn refuses, the third relays the two recovery steps. If the second turn agrees to
a discount, the skill stopped applying on the follow-up - widen the description.

`TESTING.md` has seven more scenarios. Two matter especially here:

- **Scenario 4** - abandonment is normal. The agent must NOT diagnose a problem or start a campaign.
- **Scenario 6** - no numbers given. The tool refuses; the agent must ask rather than guess.

---

## 6. If it does not trigger

In order of likelihood:

1. **The description is too narrow.** Add the words the user actually used. Descriptions match on
   intent and vocabulary, so "markdown", "promo", "offer", "win back", "drop-off" are all worth
   having in there.
2. **It fires on turn 1 but not on follow-ups.** The clause about follow-up questions is there for
   exactly this; make it more explicit if needed.
3. **Another skill is winning.** If Coworker has a general merchandising or campaign skill, it may
   match first. Make this description more specific to NORDVELL and to abandonment.

---

## 7. Why the instructions quote screen text rather than field names

The tool returns its answer **twice**: once as readable text, once as machine-readable JSON.
**Coworker only shows the agent the text.** So the instructions refer to `DISCOUNT: NOT ADVISABLE`
and `RECOMMENDED ACTION` - the words that actually appear - rather than to JSON fields the agent
never receives.

That exact mistake shipped in the sibling Campaign project: the tool returned correct data, the
assistant said it could not see any of it, and 104 tests passed while it was broken.

The precise text the agent will see, for all three products, is in `artifacts/fixture-bridge/`.
