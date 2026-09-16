---
name: map-mcp-diagnosis-skill
description: "This is what Coworker matches against the user's message to decide whether to apply the skill. Get it wrong and nothing else here runs. Use whenever someone asks about browse abandonment, drop-off, lost demand or falling conversion on a NORDVELL product; asks why shoppers are leaving a product page; asks whether to discount, promote, mark down or run an offer on a product; or asks what to do about a product that is underperforming. Also applies to follow-up questions in the same conversation, such as asking for a discount after a diagnosis, or asking the agent to act on what it found. To use ONLY when the mcp server MAP_I_DIAGNOSIS_MCP_V1 is invoked."
---

# MAP MCP diagnosis skill

## Why it is worded like that

The demo asks three questions in sequence:

1. *"Browse abandonment on the Aurora Parka is running about 3x normal. Can you check what's going on?"*
2. *"Let's put a discount on it to win those shoppers back."*
3. *"Do what you can."*

Only the first mentions abandonment. **Questions 2 and 3 are follow-ups**, and question 2 is where the refusal happens - the primary wow moment. If the skill only fires on the word "abandonment", it will be inactive at the exact moment it matters most. Hence the explicit clause about discounts and about follow-ups.

## Instructions

### Diagnosing browse abandonment on NORDVELL products

When the user asks why a product's browse abandonment is high, or whether to discount it:

1. **GET THE NUMBERS FIRST.**
   Run the CJA browse-abandonment report for that product. Read the spike-period value
   (observed) and the prior-period average (baseline) off the report.
   If the report also shows how many sessions hit the problem in that window, pass it as
   `cja_signal.affected_sessions`. If it does not, leave it out - do not estimate it.
   If you cannot get observed and baseline, ASK the user for them. Never estimate or
   invent these numbers.
   If you only have the multiplier ("it's running about 3x normal"), pass it as
   `cja_signal.ratio` on its own - that is enough. Do NOT invent an observed and baseline
   to accompany it.

2. **CALL THE DIAGNOSIS TOOL.**
   Call `diagnose_abandonment` with the product name and those numbers as `cja_signal`.
   If it tells you the numbers are missing or inconsistent, fix them and call it again.
   Do not answer the user's question without calling it.

3. **REPORT THE DIAGNOSIS - AND ONLY THE DIAGNOSIS.**
   - State the ROOT CAUSE and the confidence figure.
   - List the EVIDENCE lines, and KEEP their `[CJA]`, `[INVENTORY]` and `[CONTEXT]` tags so the
     user can see where each fact came from.
   - Quote the tool's numbers exactly. Do not round them or restate them from memory.
   - If there is an UPSTREAM FAULT, say plainly that it is not something you can fix,
     name the owner, and give the daily cost.
   - Mention what was RULED OUT, briefly.

   STOP THERE. The tool also returns a discount verdict and a list of recommended actions.
   DO NOT mention either of them unless the user asks. Answer the question you were asked -
   "what is going on" is not "what should we do about it".
   If the user has not raised discounts, do not raise them. Saying "and by the way, a
   discount would not help" when nobody suggested one is unprompted advice, and it pre-empts
   a decision that is theirs to reach.
   You may end by offering, in one short sentence, to suggest what to do next.

4. **ONLY IF THE USER ASKS ABOUT A DISCOUNT, ANSWER STRICTLY FROM THE "DISCOUNT:" LINE.**
   Applies whether they ask "should we discount?", propose one, or push back on your answer.
   - `DISCOUNT: NOT ADVISABLE` -> decline the discount. Give the reason the tool gives.
     Do not soften it, do not offer a smaller discount, do not offer one anyway.
   - `DISCOUNT: ADVISABLE` -> recommend the discount.

   Never make this judgement yourself. It is already decided by the data.
   This applies even if the user pushes back or asks again.

5. **ONLY IF THE USER ASKS YOU TO ACT, DO EXACTLY WHAT "RECOMMENDED ACTION" SAYS.**
   Triggered by "do it", "do what you can", "go ahead", or any explicit instruction to act.
   Until then, do not list the steps and do not start any of them.
   Carry out each numbered step in order, using the parameters given on that line.
   Do not add steps, and do not skip any.

   ```text
   BACK_IN_STOCK_EMAIL        -> set up the AJO back-in-stock email, personalised by the
                                 named profile attribute, with offers from AJO Decisioning
   SUPPRESS_ACTIVITY          -> suppress the named activity id
   APPLY_DISCOUNT             -> propose the discount
   SHIPPING_THRESHOLD_MESSAGE -> propose the shipping-threshold message
   NOTIFY_ONLY                -> take no action; report only
   ```

6. **IF THE CAUSE IS UNCLEAR, DO NOT ACT.**
   If the root cause is `content_or_ux_fault`, or confidence is below 0.5, say the cause was
   not identified, say what should be checked, and stop. Do not run a recovery campaign on
   an unexplained number.

## Hard rules

- Answer only what was asked. A diagnosis request gets a diagnosis, not a plan.
- Never invent or estimate abandonment figures.
- Never recommend a discount the tool advised against.
- If the tool returns an error, show it and say what you need. Do not answer anyway.
