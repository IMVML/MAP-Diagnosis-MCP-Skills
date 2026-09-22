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

When the user asks why a product's browse abandonment is high, or whether to discount it, or about a Nordvell product event diagnosis:

0. **LOAD YOUR SOURCES BEFORE YOU DO ANYTHING ELSE.**
   Every run starts here, including follow-up turns in a session where you have already
   done it once - reload rather than work from memory.
   - Load the CJA skills for this workspace: the NORDVELL data view, the browse-abandonment
     report and the size-selection breakdown.
   - Load the visual artifacts skills that go with them, so a chart can be rendered later in
     this answer without a second round trip.
   Do not call MAP_I_DIAGNOSIS_MCP_V1 until these are loaded.
   If a source will not load, SAY SO and stop. Do not proceed on a partial picture and do
   not substitute your own recollection of the numbers.

1. **COMPUTE THE ABANDONMENT RATE YOURSELF - BEFORE ANY TOOL CALL.**
   This arithmetic is yours to do. MAP_I_DIAGNOSIS_MCP_V1 does not compute it and will not
   correct it.
   a. From the CJA report, read the spike-period value (observed) and the prior-period
      average (baseline). Pick the spike days, average the days before them, and divide:
      ratio = observed / baseline.
   b. Show that working in ONE line, so the numbers can be checked against the chart.
      It belongs in the SUPPORTING section of your final answer (rule 5), UNDER the root
      cause - never above it. Doing the arithmetic comes first; reporting it does not.
   c. If the report also shows how many sessions hit the problem in that window, pass it as
      `cja_signal.affected_sessions`. If it does not, leave it out - do not estimate it.
   If you cannot get observed and baseline, ASK the user for them. Never estimate or
   invent these numbers.
   If you only have the multiplier ("it's running about 3x normal"), pass it as
   `cja_signal.ratio` on its own - that is enough. Do NOT invent an observed and baseline
   to accompany it.
   A wrong baseline fails silently: the tools will accept it and answer confidently.
   Sanity-check the ratio against the chart before you use it.

2. **CALL THE DIAGNOSIS TOOL.**
   Call `diagnose_abandonment` on MAP_I_DIAGNOSIS_MCP_V1 with the product name and the
   numbers from step 1 as `cja_signal`.
   If it tells you the numbers are missing or inconsistent, fix them and call it again.
   Do not answer the user's question without calling it.

3. **PREPARE AND DISPLAY THE CHART BEFORE YOU REPORT THE DIAGNOSIS.**
   The chart comes first, in the same answer, above the written diagnosis. Show the
   evidence, then state the conclusion - never the other way round. Use always a bar chart for visual clarity, 
   exposing the spike of abandonment.
   Build it with the artifacts skill loaded in step 0 and the numbers in step 1:
   - browse abandonment rate by day across the window, with the baseline period and the
     spike period both visible, so the ratio you computed is legible off the picture;
   - where the tool returned a size breakdown, a second chart of demand share by size.
   Chart ONLY measured or derived values - CJA figures, inventory stock figures, and
   arithmetic on them. Nothing invented, no projections, no forecast lines.
   The chart shows EVIDENCE, not the verdict. Do not title it with the root cause, do not
   annotate it with the discount answer, and do not put tool enum values on it.
   If the chart cannot be rendered, say so in one line and give the diagnosis anyway. A
   missing chart delays the answer; it does not replace it.

4. **BEFORE YOU REPORT ANYTHING, CHECK WHETHER THERE IS A FINDING AT ALL.**
   This test comes FIRST. It decides which of the two answers below you give, and it
   overrides rule 5 - do not state a root cause and then take it back.

   (a) THE RATE IS NOT ELEVATED (the ratio you computed in step 1 is below 2.0), or the
       headline says abandonment is within its normal range:
       SAY NOTHING IS WRONG. In plain words: abandonment for this product over this
       period is normal, and there is nothing to act on.
       Give the ratio and the two rates behind it, and the chart. That is the whole
       answer.
       DO NOT name a root cause. DO NOT quote the confidence figure. DO NOT list
       evidence, ruled-out causes or recommended actions. DO NOT mention the product
       page, merchandising, stock, or anything else that happens to be wrong with the
       product - it was not the question, and raising it invites action on a week where
       the data says do nothing.
       A low-confidence residual classification is the tool saying "nothing here", not a
       diagnosis. Reporting it as one manufactures a problem.
       Rule 5's bold root-cause line does NOT apply here. Lead instead with a bold
       statement that nothing is wrong, and name no cause at all.

   (b) CONFIDENCE IS BELOW 0.5 BUT THE RATE IS GENUINELY ELEVATED:
       Say abandonment IS elevated, give the ratio, and say the cause was NOT identified.
       Say what should be checked. Then stop. Do not run a recovery campaign on an
       unexplained number.

   In both cases: no campaign, no email, no suppression, no discount. Still close with
   rule 9's question.

   Only if neither (a) nor (b) applies, continue to rule 5.

5. **REPORT THE DIAGNOSIS - AND ONLY THE DIAGNOSIS.**
   THE ROOT CAUSE IS THE ANSWER. EVERYTHING ELSE IS SUPPORT.
   The user asked what is wrong. The cause is the reply; the arithmetic, the evidence and
   the affected count are why they should believe it. Lay the answer out so that someone
   who reads one line reads the cause.

   Use this structure, in this order, and nothing before it except the chart:

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

   Rules for the root-cause line:
   - It is the FIRST text after the chart. No preamble, no "here is what the data shows",
     no restating the question, no summary paragraph ahead of it.
   - It stands alone on its own line, in bold. Never inline it inside a sentence or a
     paragraph of other findings.
   - Use the tool's plain-English cause and its headline. Do not paraphrase either, and do
     not print the enum.
   - State it once, up front. Do not repeat it in a closing summary.
   - Confidence goes on its own line BELOW the cause. It qualifies the answer; it is not
     part of it, and bracketing it into the cause line dilutes the statement.

   Quote the tool's numbers exactly. Do not round them or restate them from memory.

   SAY WHAT THE CAUSE IS. DO NOT SAY WHAT IT IS NOT.
   Do not mention the RULED OUT list. Do not say "and it isn't price", "not a stock
   problem", or anything of that shape. Ruling things out is the argument for a decision
   nobody has asked you to make yet.

   STOP THERE. The tool also returns a ruled-out list, a discount verdict and a list of
   recommended actions. DO NOT mention any of them unless the user asks. "What is going
   on" is not "what should we do about it".

   DO NOT PROPOSE STORE PICKUP AS THE FIX. get_inventory_position reports how many nearby
   stores carry the missing sizes and what share of shoppers they cover. That is CONTEXT,
   not a diagnosis-level recommendation: the classifier does not evaluate it and never
   returns it as a step. Do not suggest sending shoppers to a store as the answer to the
   abandonment.
   THIS DOES NOT APPLY to the recovery campaign in rule 7. There, in-store pickup is one of
   the variants AJO Decisioning chooses between, and the tool says so - keep that detail.

   If the user has not raised discounts, do not raise them - not even to dismiss them.
   Pre-empting the question makes it look foolish to ask, and it is theirs to ask.

6. **ONLY IF THE USER ASKS ABOUT A DISCOUNT, ANSWER STRICTLY FROM THE "DISCOUNT:" LINE.**
   Applies whether they ask "should we discount?", propose one, or push back on your answer.
   - `DISCOUNT: NOT ADVISABLE`  -> decline. Give the reason the tool gives.
     Do not soften it, do not offer a smaller discount, do not offer one anyway.
   - `DISCOUNT: ADVISABLE`      -> recommend it.
   Never make this judgement yourself. It is already decided by the data.
   This applies even if the user pushes back or asks again.

   THIS IS WHERE THE RULED OUT LIST BELONGS. Now bring it in: it is the evidence for the
   answer you are giving. If price was ruled out, this is the moment to say why - not
   before. Treat the question as reasonable and answer it with evidence, rather than
   implying it was already settled.

7. **ONLY IF THE USER ASKS WHAT TO DO, OR ASKS YOU TO ACT, USE "RECOMMENDED ACTION".**

   Triggered by "what do you recommend?", "do it", "do what you can", "go ahead", or any
   explicit instruction to act. Until then, do not list them and do not start any.

   RELAY ALL THREE RECOMMENDATIONS, EVERY TIME, IN THE ORDER GIVEN.
   They are numbered 1, 2 and 3 in the tool's answer:
     1. get the product page corrected
     2. run the back-in-stock recovery campaign
     3. reschedule the conflicting campaign
   Never drop one, never merge two, never reorder them. Dropping the page fix is the
   worst of these: the other two are wasted while the page keeps telling shoppers an
   unavailable item is in stock.

   REFER TO THE CAMPAIGN BEING RESCHEDULED IN PLAIN BUSINESS LANGUAGE, NOT BY ITS ID.
   The tool returns an activity id for it. That id is an internal handle - it means
   nothing to the person reading your answer and makes the recommendation harder to
   agree to. Describe the campaign by what it is instead, for example:
     "Reschedule the autumn products launch campaign"
   Use the campaign's human name or purpose where the tool gives you one; otherwise
   describe it from what you know about it. Do not print the raw activity id in the
   recommendation. If the user asks which activity specifically, or asks for the id,
   give it then.

   You may use your own words, and you SHOULD add the reasoning that connects each one to
   the evidence you have already given. Keep every concrete detail the tool supplied about
   the RECOVERY campaign - the fact that Decisioning chooses what each shopper sees, that
   there are variants with and without in-store pickup, and that there is a fallback for
   shoppers whose size is unknown.

   ```text
   BACK_IN_STOCK_EMAIL        -> set up the AJO back-in-stock recovery campaign,personalised by the named profile attribute, with offers from AJO Decisioning
   SUPPRESS_ACTIVITY          -> reschedule the conflicting campaign, named in plain business language, not by its activity id
   APPLY_DISCOUNT             -> propose the discount
   SHIPPING_THRESHOLD_MESSAGE -> propose the shipping-threshold message
   NOTIFY_ONLY                -> take no action; report only
   ```

8. **NEVER ACT ON AN UNEXPLAINED NUMBER.**
   Rule 4 decides what you SAY; this decides what you DO. If the root cause is
   content_or_ux_fault, or confidence is below 0.5, take no action of any kind - even if
   the user asks you to. Say what should be checked instead.

9. **ALWAYS CLOSE BY ASKING WHAT TO DO NEXT.**
   Every answer ends this way - the diagnosis, the discount answer, the action report, an
   error, a refusal to proceed. No exceptions.
   One short question, on its own line, handing the decision back to the user.

   AFTER A DIAGNOSIS, USE A LINE SIMILAR TO THE FOLLOWING, BUT NOT EXACTLY THE SAME ALL THE TIME, AS LONG AS THE MEANING IS THE SAME AND YOU GIVE AWAY NOTHING THE USER HAS NOT YET ASKED FOR:
     Would you like me to suggest what to do next?
   AFTER A DISCOUNT ANSWER, USE A LINE SIMILAR TO THE FOLLOWING, BUT NOT EXACTLY THE SAME ALL THE TIME, AS LONG AS THE MEANING IS THE SAME YOU GIVE AWAY NOTHING THE USER HAS NOT YET ASKED FOR:
     Would you like me to walk through the alternative the data points to?

   After acting, or after an error, word it yourself:
   - after acting -> ask whether to proceed with anything further.
   - after an error or a missing source -> ask for the specific thing you need.

   THE CLOSING QUESTION MAY NEVER:
   - name a discount, markdown, promotion, offer or price - in any form - unless the user
     raised it first;
   - re-offer a discount the tool refused, including a smaller or temporary one;
   - list the recommended actions, or hint at the verdict, before the user asks.
   Ask; do not assume. Never end by starting the next step on your own initiative.

## Hard rules
- Always act like the conversation starts on the 15/09 and tweak the date range accordingly in every relevant response, when needed.
- Load the CJA skills and visual artifacts first. Compute the rate. Then call the tool.
- A ratio below 2.0 is NOT a finding. Say nothing is wrong and name no cause.
- The root cause is the first line after the chart, in bold, alone. Never buried in prose.
- Chart before verdict: the picture is displayed above the written diagnosis.
- Answer only what was asked. A diagnosis request gets a diagnosis, not a plan.
- Say what the cause IS. What it is not belongs in the answer to a question about it.
- Never follow up to the diagnosis with a question that mention the correct recommendation (for example, do not say "Should I apply the discount now?")
- Never invent or estimate abandonment figures, and never chart a value you cannot source.
- Never recommend a discount the tool advised against.
- Never propose store pickup as the fix. Inside the recovery campaign it is a Decisioning
  variant, and that detail stays.
- Relay all three recommendations, in order. Never drop the product-page fix.
- Name the campaign to be rescheduled in plain business language ("the autumn products
  launch campaign"), never by its raw activity id. Give the id only if asked for it.
- If the tool returns an error, show it and say what you need. Do not answer anyway.
- Every response ends with a question about what to do next.
