# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Edited `policy_extractor/routing.py` line 21, 
  raising `DEFAULT_CONFIDENCE_THRESHOLD` from `0.90` to `0.99`.
- **Command I ran:** `.venv/bin/pytest tests/test_us04_routing.py -v`
- **What I predicted:** Records with confidence between 0.90 and 0.99 that previously cleared 
  the bar for auto-approval would now fail to clear it and route to human review instead.
- **What actually happened (paste the key output):** 4 tests failed, 5 passed (baseline was 
  9 passed, 0 failed). `test_ac_04_02_all_clear_routes_to_auto_approve` failed with 
  `AssertionError: assert 'human_review' == 'auto_approve'` — a record that used to auto-approve 
  now routes to human review because its confidence score no longer clears the stricter threshold.
- **How this differs from the unperturbed run:** At the original threshold (0.90), this same 
  test suite passes 9/9 (routing-test-output.txt). Raising the threshold shows the router is 
  sensitive to this exact setting, and that a stricter bar pushes more records toward human 
  review rather than silently approving them.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Edited `mortgage_extractor/config.py` line 10, 
  raising `DEFAULT_TOLERANCE_USD` from `1.00` to `2000.00`.
- **Command I ran:** `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:** With the tolerance raised above the document's actual discrepancy 
  (1250.0), the same mismatched document would now pass validation instead of being flagged.
- **What actually happened (paste the key output):** `"consistent": true, "discrepancies": []`. 
  Before the change, the same document produced `"consistent": false`, `"calculated": 9642.17`, 
  `"stated": 10892.17`, `"delta": -1250.0` (see extract-income-mismatch.txt).
- **How this differs from the unperturbed run:** At the original tolerance (1.00), any 
  discrepancy above one dollar is flagged. Raising the tolerance to 2000 makes the validator 
  blind to a real $1,250 error — showing the tolerance setting directly controls how much 
  arithmetic disagreement the system will silently accept.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** No file changed — added 
  the `--simulate-timeout` flag to force one data source to fail.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:** The affected metric would be marked incomplete 
  rather than causing the whole briefing to fail.
- **What actually happened (paste the key output):** 
  `late_shipment_count` appeared under "## Incomplete" with 
  `missing source: timeout reading logistics`.
- **How this differs from the unperturbed run:** Without 
  `--simulate-timeout`, that same metric appears normally under 
  Well-Established/Contested instead of Incomplete.
  
---



