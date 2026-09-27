# Reflection Brief — Evaluation and Observability Capstone

**Name:** Divakar Reddy
**Date:** 2026-09-27

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux (Udacity cloud workspace, code-server) |
| Python version | 3.13.0 |
| Date run | 2026-09-26 to 2026-09-27 |
| Ran any system live? (which) | No. No `ANTHROPIC_API_KEY` was configured. The System 1 live command (`policy-extractor pipeline data/policies/ --routing-out routing_decisions.json`) was attempted and failed with `TypeError: Could not resolve authentication method`. All three systems were run using their offline fallback modes instead: System 1 via `test_us04_routing.py` and `test_us01_retry.py`, System 2 via `--mode replay`, System 3 via `--offline`. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (full `tests/` suite); 9 passed (`test_us04_routing.py`); 13 passed, 1 skipped (`test_us01_retry.py`) |
| Routing output file | `routing-test-output.txt` (offline — `routing_decisions.json` was not generated since the live command failed on the missing API key) |
| auto_approve / human_review / spot_check counts | Not available from a live pipeline run (no API key). Evidence instead comes directly from the routing test suite: `test_ac_04_02_all_clear_routes_to_auto_approve PASSED`, `test_ac_04_02_low_confidence_routes_to_human_review PASSED`, `test_ac_04_02_integration_failure_routes_to_human_review PASSED` |

**1a. Retry boundary.** From `tests/test_us01_retry.py -v`: