I built three zero-shot prompts (sentiment, product description, extraction) and ran each 5, 10 and 15 times with `gpt-4o-mini`, and the repeats showed what a single call hid: 
- v1 sentiment answered with sentences and invented categories ("Positive Feedback", "complaint") instead of a label (27% consistency, 0% format-valid), 
- v1 product copy was 220-347 words of markdown with invented brand names and specs (0% spec-pass), 
- and v1 extraction was correct but never valid JSON (20% consistency). 

Most of the improvement came from a precise v2 (closed label set and one-word output, explicit template and length limits, a JSON schema), which took sentiment and product to 100%; 
extraction stalled at 73-80% because the model sometimes wrapped its JSON in a code fence, and Chain-of-Thought with a `FINAL JSON:` delimiter fixed that (100%), while few-shot examples are what made sentiment robust on sarcasm, mixed reviews and questions (100% correct on five unseen messages versus 0-70% for v1). 

All v3 prompts cleared the 80% bar on 15 fresh runs (sentiment 100%, extraction 100%, product 93%), but the variation tests found real gaps that a single base input would not have: 

v3 extraction invented a delivery sentiment from a customer-service complaint (50% correct) and v3 product outputs hugged the minimum word count, so I added a scope rule and longer examples in a v3.1 that reached 100% on every case. 

The temperature experiment showed that lowering temperature cannot rescue a vague prompt (v1 product was 0% spec-pass at every temperature) and that "consistent" is not "correct" (v1 extraction was 100% repeatable at T=0.3 yet 0% valid JSON); next time I would test v3.1 on more held-out variations (my fix was tuned on the case that exposed it), use `response_format` for JSON, and use larger batches because 10-15 runs can swing by 10-20 points.
