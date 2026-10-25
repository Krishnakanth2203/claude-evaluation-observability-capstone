# Perturbation Log

For each system, a deliberate input or configuration change was evaluated. The expected behavior was compared with the recorded test or run output.

---

### System 1 — Validated, routed insurance policy pipeline

- **Change I made (input + scenario):** Evaluated the missing-source-field scenario using the existing test `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately`. The test uses a recorded extraction response in which the required `endorsements` field is `None`, representing information that could not be found in the source document. This tests the existing retry and escalation logic without requiring a live API call.

- **Command I ran:**
  ```bash
  cd "/workspace/Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution"

  env -u ANTHROPIC_API_KEY .venv/bin/pytest \
    tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately -v
  ```

- **What I predicted:** A required field that is genuinely absent should trigger immediate escalation instead of repeated retries. Retrying a response cannot recover information that the source document does not contain and could encourage an unsupported guess.

- **What actually happened:** The test passed in the offline suite. The test asserts that the result is a `RetryFutileEscalation`, that `result.field == "endorsements"`, and that `result.detected_pattern == "endorsements_absent"`. It uses a recorded client seeded with one response, so the passing test verifies that this scenario escalates without requesting another response from that fake client.

  **Evidence:** `01-policy-pipeline/tests.txt` — 45 passed, 3 skipped. The routing-specific suite also passed all 9 tests in `01-policy-pipeline/routing-tests.txt`.

- **How this differs from the unperturbed scenario:** The existing test `test_ac_01_03_format_failure_retries_with_error_appended` covers a malformed response that can be retried with validation feedback. The missing-source scenario instead escalates immediately. This demonstrates the distinction between a potentially fixable formatting problem and a field for which the source provides no information.

- **Additional observation:** The calibration report at `01-policy-pipeline/calibration-report.txt` showed:
  ```text
  umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
  OVERALL brier=0.291
  ```
  This slice deserves further investigation because the observed accuracy is zero despite high confidence. The sample size is small, so more labeled examples are needed before drawing broader conclusions.

- **Limitation:** The live pipeline and live API tests could not be verified because the configured Anthropic credential returned HTTP 401 (`Invalid bearer token`). The offline tests passed, but this does not establish that live API integration works.

---

### System 2 — Schema-enforced mortgage document extraction

- **Change I made (input scenario):** Selected the supplied `fixtures/documents/income_sum_mismatch.txt` fixture, which contains a discrepancy between the listed earnings and the stated monthly total. The listed amounts sum to 9,642.17, while the document states 10,892.17. I compared this with the clean `fixtures/documents/appraisal_informal_sqft.txt` fixture.

- **Commands I ran:**
  ```bash
  cd "/workspace/Build a Resilient Mortgage Document Extraction System/04-validate-mathematical-consistency/solution"

  .venv/bin/mortgage-extract \
    fixtures/documents/income_sum_mismatch.txt --mode replay

  .venv/bin/mortgage-extract \
    fixtures/documents/appraisal_informal_sqft.txt --mode replay
  ```

- **What I predicted:** The extractor should preserve the total stated in the mismatched document rather than silently correcting it. The independent validation step should recompute the total from the individual amounts and report the difference. The clean document should pass its applicable consistency checks.

- **What actually happened:** The recorded discrepancy output reported:
  ```json
  {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0
      }
    ]
  }
  ```
  The clean appraisal run produced `"consistent": true` and an empty discrepancies list.

  **Evidence:** `02-mortgage-extraction/document-runs.txt`. The test suite also recorded 25 passed in `02-mortgage-extraction/tests.txt`.

- **How this differs from the clean scenario:** The mismatched income document produced a validation failure with a 1,250.00 difference. The clean appraisal document passed consistency validation. The extraction pipeline could preserve the source value while the separate validator identified the arithmetic inconsistency.

- **What this demonstrates:** Schema-constrained extraction and deterministic validation address different problems. A schema can enforce the response structure and types, while a mathematical validator can detect a well-formed but internally inconsistent total.

---

### System 3 — Multi-source supply-chain investigation

- **Change I made (configuration):** Used the CLI's `--simulate-timeout` option to simulate a timeout from the logistics source. Compared the timeout run with the ordinary offline investigation.

- **Commands I ran:**
  ```bash
  cd "/workspace/Investigate Supply Chain Risk with Multi-Source Synthesis/03-resilient-coordinator/solution"

  .venv/bin/supply-chain-investigate meridian --offline

  .venv/bin/supply-chain-investigate meridian \
    --offline --simulate-timeout
  ```

- **What I predicted:** The investigation should still produce a briefing when one source times out. Information that depends on the unavailable source should be identified as missing, rather than interpreted as zero risk or silently treated as confirmed.

- **What actually happened:** The saved timeout run recorded the logistics timeout and placed `late_shipment_count` under the incomplete section with a missing-source annotation. The ordinary investigation recorded a logistics value of 11.0 shipments under the well-established section.

  **Evidence:** `03-supply-chain/investigation-run.txt` and `03-supply-chain/timeout-run.txt`. The test suite recorded 34 passed in `03-supply-chain/tests.txt`.

- **How this differs from the unperturbed scenario:** In the ordinary run, `late_shipment_count` had a reported value from logistics. In the timeout run, the metric was marked incomplete because the source failed to respond. The ordinary run also recorded a conflict for `on_time_delivery_rate`: 95.0 percent from `supplier_audit` (as of 2026-04-10) versus 78.0 percent from `logistics` (as of 2026-04-05).

- **What this demonstrates:** A source timeout is different from a reported disagreement between sources. Preserving provenance and marking unavailable evidence helps downstream readers distinguish an unresolved conflict from missing information, while allowing the investigation to complete with partial coverage.

---

## Overall observations

1. A missing required field should be escalated rather than filled with an unsupported guess.
2. Structured extraction does not replace deterministic validation of mathematical consistency.
3. A source failure should be represented as incomplete evidence, not as a zero value.
4. Confidence and aggregate metrics should be examined alongside observed accuracy and narrower evaluation slices.
5. Offline test success and saved run evidence do not, by themselves, prove that live external API integration works.
