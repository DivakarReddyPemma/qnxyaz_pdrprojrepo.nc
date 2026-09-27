# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** No file changed — ran the 
  offline routing test suite (`tests/test_us04_routing.py`) since I had 
  no live API key, per the project's offline fallback instructions.
- **Command I ran:** `.venv/bin/pytest tests/test_us04_routing.py -v`
- **What I predicted:** Records with low confidence or an integration 
  failure would be routed to human review rather than auto-approved.
- **What actually happened (paste the key output):** All 9 tests passed, 
  including `test_ac_04_02_low_confidence_routes_to_human_review PASSED` 
  and `test_ac_04_02_integration_failure_routes_to_human_review PASSED`.
- **How this differs from the unperturbed run:** The full pipeline run 
  (45 tests) confirms the same safe defaults; this test isolates and 
  proves the specific human-review escalation logic on its own.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Used the provided fixture 
  `income_sum_mismatch.txt`, which has a stated total that doesn't match 
  the sum of its line items.
- **Command I ran:** `.venv/bin/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:** The system would flag the mismatch rather than 
  trust the stated total.
- **What actually happened (paste the key output):** 
  `"consistent": false`, `"calculated": 9642.17`, `"stated": 10892.17`, 
  `"delta": -1250.0`.
- **How this differs from the unperturbed run:** Running 
  `appraisal_informal_sqft.txt` (a clean document) instead produces no 
  discrepancy entry and `"consistent": true`.

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



