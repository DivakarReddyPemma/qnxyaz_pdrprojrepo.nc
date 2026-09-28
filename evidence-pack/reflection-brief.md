# Reflection Brief — Evaluation and Observability Capstone

**Name:** Divakar Reddy
**Date:** 2026-09-28

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux (Udacity cloud workspace, code-server) |
| Python version | 3.13.0 |
| Date run | 2026-09-26 to 2026-09-28 |
| Ran any system live? (which) | No. No ANTHROPIC_API_KEY was configured. The live command `policy-extractor pipeline data/policies/ --routing-out routing_decisions.json` failed with `TypeError: Could not resolve authentication method`. Offline fallbacks were used: System 1 via tests/test_us04_routing.py, System 2 via --mode replay, System 3 via --offline. |

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (01-policy/tests-output.txt) |
| Routing output file | 01-policy/routing-test-output.txt (9 passed in 0.03s). No routing_decisions.json, because the live pipeline needs an API key. |
| auto_approve / human_review / spot_check counts | Not produced (live run needs a key). The offline routing tests cover each route instead. |

**1a. Retry boundary.** From `tests/test_us01_retry.py -v` (13 passed, 1 skipped): `test_ac_01_02_validator_flags_null_required_as_missing_source PASSED` and `test_ac_01_04_missing_source_halts_immediately PASSED`. When a required field is null, the validator flags it as a missing source and the case escalates after exactly one API call, with no retry. Retrying is futile because the value is not in the document, so asking the model again cannot produce it. It wastes API calls and pushes the model toward inventing a value. Escalating sends the record to a human who can find the real data. By contrast, `test_ac_01_03_format_failure_retries_with_error_appended PASSED` shows that a fixable format error is worth a retry.

**1b. Reading the router.** The record is the case in `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` (routing-test-output.txt, PASSED). Model confidence was high, so the confidence signal did not send it to a human. The independent reviewer signal (reviewer disagreement) did. If the router had trusted confidence alone, this record would have been auto-approved even though the reviewer disagreed.

**1c. Where the aggregate lies.** From 01-policy/calibration-output.txt: `umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865`, against `OVERALL brier=0.291`. The overall number looks moderate, but slicing by policy_type x field exposes one cell that is 93% confident and 0% correct. A single aggregate hides that failure.

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed in 1.78s (02-motgage/tests-output.txt) |
| Document run | extract-appraisal.txt, extract-income-missing.txt, extract-income-mismatch.txt |
| Classified type | extract-appraisal.txt gives the typed appraisal result: `"property_type": "single_family"`, `"occupancy_type": "primary_residence"`, `"gross_living_area_sqft": 2400`, `"appraised_value": 410000.0` |

**2a. Two guarantees.** extract-income-mismatch.txt shows `"consistent": false`, `"calculated": 9642.17`, `"stated": 10892.17`, `"delta": -1250.0`. Tool use guarantees the output is valid, correctly typed JSON. It cannot check that numbers add up: a well-typed 10892.17 is still wrong. The validator recomputes the sum and catches that. The validator, in turn, cannot catch a malformed or wrongly typed field, which the schema prevents.

**2b. Refusing to fabricate.** extract-income-missing.txt contains `"bonus_monthly": null`. The other unstated income fields (`bonus_ytd`, `commission_monthly`, `overtime_monthly`, `other_monthly`, `stated_monthly_total`) are null too, while `"base_monthly": 5673.08` is filled because the document actually states it. The system returned null instead of a guess for two linked reasons. First, the schema fields are nullable, so null is a valid, type-correct answer and the model is not forced to produce a number to satisfy the schema. Second, the system prompt tells the model to return null for unstated fields and to normalize only what the source states (`test_ac_01_01_system_prompt_instructs_null_for_unstated_fields PASSED` in test_us01_retry.py). A guessed bonus would pass the schema and look real, so it would silently corrupt the income total downstream. The validation block shows `"consistent": true, "discrepancies": []` because there is nothing to compare, which is not a fabricated result.

**2c. Normalization.** extract-appraisal.txt shows `"gross_living_area_sqft": 2400`, produced from the informal source phrase "about 2,400 sq ft". Normalizing at extraction gives every downstream consumer one numeric type, instead of each consumer parsing free text.

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed in 149.86s (03-supply-chain/tests-output.txt) |
| Briefing file | 03-supply-chain/briefing-output.txt; timeout run in briefing-timeout-output.txt |
| Section the conflict landed in | Contested (on_time_delivery_rate, flagged ESCALATE) |

**3a. Annotate, don't arbitrate.** In briefing-output.txt, Contested section, `on_time_delivery_rate` lists `95.0 percent — supplier_audit (as of 2026-04-10)` and `78.0 percent — logistics (as of 2026-04-05)`. Both values are kept with their sources and dates. The system cannot know which is right: an audit may use a different definition or period than live logistics data. Picking one or averaging would invent a number nobody reported and hide the disagreement. Keeping both lets a reader check each source and decide, and the ESCALATE flag ("high-impact metric is contested across sources") tells them to look.

**3b. Source goes dark.** briefing-timeout-output.txt (run with --simulate-timeout) starts with `Sources unavailable: logistics unavailable (timeout)` and lists `late_shipment_count [missing source: timeout reading logistics]` under Incomplete. Unreachable is recorded as a missing source for that metric. That differs from nothing to report, as in `production_capacity_utilization [missing source: no source reported this metric]`. The run still finishes because one failed source is annotated, not treated as fatal. Note also that the Contested section becomes `_none_` and on_time_delivery_rate shows only `95.0 percent — supplier_audit`, because the logistics side of the conflict is gone.

**3c. Dates as a guardrail.** In briefing-output.txt, `port_disruption` is dated `(industry_news, 2026-03-17)` and `supplier_financial_distress` is dated `(industry_news, 2026-03-09)`. Each claim carries its own date, so a difference in time reads as two observations made at different times, not as a contradiction between sources.

## 4. Synthesis

**4a. One principle.** Evaluate the output, do not trust the model's word. The clearest catch is 01-policy/calibration-output.txt: `umbrella exclusions conf=0.93 acc=0.00` hidden behind `OVERALL brier=0.291`. A design that trusted the model's confidence, or the average, would have shipped that failure.

**4b. Confidence is not correctness.** System 1 is where it mattered most. In the same calibration cell, confidence was 0.93 while accuracy was 0.00. The routing test `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` shows a high-confidence record still going to human review because the reviewer disagreed.

**4c. Reliability principle shared by all three systems, applied to my own work.** The shared principle: never let uncertainty be silently absorbed into a confident answer. System 1 escalates to human_review, System 2 returns null or flags `"consistent": false` with a delta, and System 3 marks metrics Incomplete or keeps both conflicting values with sources. In my own workflow, extracting fields from messy customer support tickets, I would use validated retry with escalation first. Fixable format errors get one retry with the error appended. Missing fields stay null and escalate without retry. Sums and dates are checked in code, not trusted from the model. I would instrument the escalation rate, the null rate per field, and how often reviewers overrule the model's confidence, so I would know when the system started to break.
