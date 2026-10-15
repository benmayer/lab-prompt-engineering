# Improved Prompts

Final prompts for TechFlow Solutions, with the reasoning behind each change. Numbers are consistency over 15 runs at the default temperature (1.0), model `gpt-4o-mini`;
full tables are in `prompt_engineering_lab.ipynb` and `failure_analysis.md`. Prompts below are rendered directly from the notebook source.

## Summary

| Task | v1 → v3 consistency | Winning technique | Final prompt |
|---|---|---|---|
| Sentiment analysis | 33% → 100% | Closed label set + few-shot (5 examples) | v3 |
| Product description | 0% → 93% (100% for v3.1) | Exact template + few-shot (2 examples) + length target | v3.1 |
| Data extraction | 33% → 100% | Schema + Chain-of-Thought with a `FINAL JSON:` delimiter | v3.1 |

## 1. Sentiment analysis

**v1 (baseline)**
```text
Classify this customer message: "{message}"
```
Fails because nothing tells the model *what the answer looks like*: it replies with sentences and invents categories ("Positive Feedback", "complaint", "Order Update").

**v2 — instruction + format + constraint**
```text
You are a sentiment classifier for customer-service messages.
Classify the sentiment of the customer message below as exactly one of: positive, negative, neutral.
Respond with ONLY that one lowercase word - no punctuation, no explanation.

Customer message: "{message}"
```
Why: a closed label set and "ONLY that one lowercase word" removed the format variance (27% → 100% on the easy message).

**v3 — few-shot + rules (final)**
```text
You are a sentiment classifier for customer-service messages.
Classify each message as exactly one of: positive, negative, neutral.

Rules:
- positive: the customer expresses satisfaction or thanks.
- negative: the customer expresses dissatisfaction, a complaint or frustration. Sarcasm that is really a complaint is negative.
- neutral: no emotion (questions, factual statements) OR clearly mixed praise and complaint with no dominant side.
Respond with ONLY the one lowercase label - no punctuation, no explanation.

Message: "Terrible service, I waited 40 minutes and nobody helped me."
Sentiment: negative

Message: "Thanks, the replacement arrived and it works perfectly."
Sentiment: positive

Message: "I'd like to know the store opening hours on Sunday."
Sentiment: neutral

Message: "The camera is stunning but the battery dies by lunchtime."
Sentiment: neutral

Message: "Wow, three weeks to ship a pencil. Truly impressive."
Sentiment: negative

Message: "{message}"
Sentiment:
```
Why: the base message is easy, so v2 already worked there. The rules and examples exist for the hard cases: neutral questions, mixed reviews and sarcasm. On five unseen messages v3 was 100% correct and consistent; v1 was correct 0-70% of the time (0% on four of the five) because it often answered with topics ("complaint", "Order Update", "mixed feedback") instead of a sentiment.
Judgement call encoded in the prompt: mixed praise-and-complaint maps to `neutral` (the client only has three labels).

**What worked best:** instruction + closed output set did most of the work; few-shot examples are what make edge cases (sarcasm, mixed) reliable.

## 2. Product description

**v1 (baseline)**
```text
Create a product description for {product and price}.
```
Fails because length, structure and tone are all unspecified: 220-347 words, bold markdown, invented brand names and specs, exclamation marks.

**v2 — structure + length + style**
```text
Write a product description for the item below, for an online store.

{product info}

Requirements:
- Start with a short headline of at most 8 words on its own line.
- Then a description of exactly 2 sentences.
- Then exactly 3 feature bullets, each starting with "- ", at most 8 words each.
- End with a line stating the price exactly as given.
- 50-80 words in total. Confident, plain tone. No exclamation marks, emojis or markdown formatting (no bold, no headings).
```
Why: every dimension that varied in v1 is now a stated constraint. This alone reached 100% spec-pass.

**v3.1 — few-shot + template + length target (final)**
```text
You write product descriptions for an online store. Follow the template exactly.

Target length: 55-75 words in total.

Template:
<Headline, max 8 words>
<Exactly 2 sentences of 18-25 words each, plain confident tone, no exclamation marks, no emojis, no markdown>
- <feature, max 8 words>
- <feature, max 8 words>
- <feature, max 8 words>
Price: <price exactly as given>

Example 1
Product: mechanical keyboard, hot-swappable switches
Price: $89.00
Output:
Typing that feels as good as it sounds
Built around hot-swappable switches, this keyboard lets you tune the feel of every single key to match your own taste. Its sturdy aluminium frame keeps each keystroke steady and quiet through long, demanding work days.
- Hot-swappable mechanical switches
- Sturdy aluminium frame
- Comfortable full-size layout
Price: $89.00

Example 2
Product: USB-C hub with 7 ports
Price: $45.50
Output:
One cable, seven connections
This compact hub turns a single USB-C port into everything your desk could possibly need for the day ahead. Plug in displays, drives and chargers at the same time without ever hunting for adapters.
- Seven ports in one slim body
- 4K HDMI video output
- 100W pass-through charging
Price: $45.50

Now write the description.
{product info}
Output:
```
Why: v3 (template + 2 short examples) scored 100% then 93% in the fresh batch - one output was 38 words, below the 40-word floor, because the examples were only ~45 words. v3.1 lengthens the examples to ~57 words and adds an explicit 55-75 word target; outputs moved to 48-65 words and all 45 runs across three products passed.

**What worked best:** explicit structure + length limits (v2) gave the big jump. Examples add consistency of *voice* and guard the edges of the spec. Content still differs run to run (15 unique outputs in 15 runs), which is the desired behaviour for generation.

## 3. Data extraction

**v1 (baseline)**
```text
Extract information from this customer feedback: "{feedback}"
```
Fails because there is no schema: right facts, but bullets/prose, different labels each run, raw `#12345` and `March 15th`.

**v2 — schema + format requirement**
```text
Extract structured data from the customer feedback below.

Return a JSON object with exactly these keys:
- "order_id": string, the order/item number without '#', or null if absent
- "order_date": string in the form "Month D" (e.g. "March 15"), or null
- "delivery_sentiment": "positive", "negative", "neutral" or null if delivery is not mentioned
- "packaging_sentiment": "positive", "negative", "neutral" or null if packaging is not mentioned
- "issues": array of zero or more of: "damaged_packaging", "late_delivery", "wrong_item", "missing_item", "damaged_product", "other"

Return only the JSON object.

Feedback: "{feedback}"
```
Why: fixed keys, types, enums and null rules. 80% valid; the failures were all the same defect - the model wrapped the JSON in a ```json fence so `json.loads` raised (content was 100% correct).

**v3.1 — Chain-of-Thought + worked example + scope rule (final)**
```text
You extract structured data from customer feedback. Work through it step by step, then output JSON.

Schema (exactly these keys):
- "order_id": string without '#', or null if no order/item number is stated
- "order_date": "Month D" (e.g. "March 15"; convert "3 January" -> "January 3", drop ordinal suffixes), or null
- "delivery_sentiment": "positive" | "negative" | "neutral" | null (null if delivery is not mentioned)
- "packaging_sentiment": "positive" | "negative" | "neutral" | null (null if packaging is not mentioned)
- "issues": array drawn only from: "damaged_packaging", "late_delivery", "wrong_item", "missing_item", "damaged_product", "other"


Scope rules (important):
- "delivery_sentiment" is ONLY about shipping/delivery (speed, lateness, whether it arrived). "packaging_sentiment" is ONLY about the box/packaging.
- Complaints about customer service, unanswered emails, price or the product itself are NOT about delivery or packaging -> use null for those fields
  (list the problem under "issues" if a label fits, e.g. wrong_item).

Procedure - write it out under a line "REASONING:":
1. Quote any order/item number, or write "none".
2. Quote any date and rewrite it as "Month D", or write "none".
3. Say what is stated about shipping/delivery itself and whether that is positive, negative or neutral (or "not mentioned" - service complaints do not count).
4. Say what is stated about packaging, the same way.
5. List which allowed issue labels apply.
Then write a line "FINAL JSON:" followed by the raw JSON object only (no code fences, nothing after it).

Example
Feedback: "Item #777 bought on 9th June never showed up, but the seller was polite."
REASONING:
1. Order number: 777
2. Date: "9th June" -> "June 9"
3. Delivery: the item never showed up -> negative
4. Packaging: not mentioned
5. Issues: missing_item
FINAL JSON:
{"order_id": "777", "order_date": "June 9", "delivery_sentiment": "negative", "packaging_sentiment": null, "issues": ["missing_item"]}

Feedback: "{feedback}"
REASONING:
```
Why: reasoning happens first and is kept apart from the machine-readable part (`FINAL JSON:`), which removes the code-fence failure and forces explicit normalisation. The scope rule was added in v3.1 after the `missing_fields` variation showed the model inventing a delivery sentiment from a customer-service complaint (50% correct → 100%).

**What worked best:** CoT for extraction (multi-field, normalisation, null handling). A cheaper alternative worth testing in production is `response_format={"type": "json_object"}`, which would fix the fence problem without the extra tokens.

## Technique selection

- **Few-shot** when the difficulty is *what the output should look like or where the label boundaries are* (classification, templated generation).
- **Chain-of-Thought** when the difficulty is *multi-step interpretation* (several fields, normalisation, deciding "mentioned or not").
- **Constraints and schema first** - for two of three tasks a precise v2 already got most of the gain.
- **Lower temperature is not a substitute**: v1 product copy was 0% spec-compliant at every temperature from 0 to 1.0.
