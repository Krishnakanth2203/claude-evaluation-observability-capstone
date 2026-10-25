# Reflection Brief — Evaluation and Observability Capstone

**Name:** Krishna kanth E
**Date:** 25/10/2026

> Ground each answer in the recorded runs and evidence files. Distinguish passing offline tests from successful live integration. Do not treat model confidence as proof of correctness.

---

## 0. Environment

| Field | Value |
|---|---|
| Operating system | Linux x86_64 |
| Python version | 3.13.0 |
| pip version | 24.2 |
| Git version | 2.39.5 |
| Date of evidence review | 2026-10-25 |
| Live API status | The insurance pipeline's live API request returned HTTP 401, `Invalid bearer token`. Offline tests were run successfully afterward. |

---

## 1. Validated, routed insurance policy pipeline

| Evidence | Result |
|---|---|
| Full offline test suite | 45 passed, 3 skipped — `01-policy-pipeline/tests.txt` |
| Routing-specific tests | 9 passed — `01-policy-pipeline/routing-tests.txt` |
| Static checks | All checks passed — `01-policy-pipeline/static-checks.txt` |
| Calibration report | `01-policy-pipeline/calibration-report.txt` |
| Live pipeline run | Authentication failed with HTTP 401 — `01-policy-pipeline/pipeline-run.txt` |
| Routing decisions JSON | Not available from a successful live pipeline run |

### 1a. Retry boundary

The test `test_ac_01_04_missing_source_halts_immediately` evaluates an extraction response where `endorsements` is `None`. It checks that the result is a `RetryFutileEscalation`, with `field == "endorsements"` and `detected_pattern == "endorsements_absent"`.

The test uses a recorded client with one scripted response. Its passing result verifies that the missing-source scenario escalates without requesting another response from that fake client.

This behavior makes sense because retrying cannot recover information absent from the source document. Escalation allows a person to resolve the missing information instead of encouraging an unsupported guess.

**Evidence:** `01-policy-pipeline/tests.txt`.

### 1b. Reading the router

The live pipeline did not produce a routing-decision file because the API request failed authentication. However, the routing test `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` passed.

That test exercises a case where every field has confidence `0.99`, but the independent reviewer disagrees about `premium_amount`. The expected decision is `human_review`, with `reviewer_disagreements == ["premium_amount"]` and no fields below the confidence threshold.

The reviewer disagreement therefore overrides the apparently high confidence. If the system trusted confidence alone, the disagreement could be missed and an incorrect value could be approved.

**Evidence:** `01-policy-pipeline/tests.txt`.

### 1c. Where the aggregate hides a problem

The recorded calibration report contains:

```text
auto      premium_amount  n=3 conf=0.95 acc=1.00 brier=0.003
home      deductible      n=1 conf=0.90 acc=1.00 brier=0.010
umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
OVERALL brier=0.291
```

The umbrella/exclusions slice reports confidence `0.93` but accuracy `0.00` across two observations. This is a clear warning that confidence does not guarantee correctness. The overall Brier score of `0.291` does not reveal the entire problem on its own.

Breaking calibration results down by `policy_type × field` exposes a weak area that could be hidden by aggregate results. However, the sample sizes are small, so additional labeled examples are needed before drawing broad conclusions.

**Evidence:** `01-policy-pipeline/calibration-report.txt`.

---

## 2. Schema-enforced mortgage document extraction

| Evidence | Result |
|---|---|
| Passing test count | 25 passed — `02-mortgage-extraction/tests.txt` |
| Run evidence | `02-mortgage-extraction/document-runs.txt` |
| Static checks | `02-mortgage-extraction/static-checks.txt` |

### 2a. Two different guarantees

The income discrepancy run reported:

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

The structured extraction mechanism enforces the response's schema and expected types. That does not guarantee that the extracted values are mathematically consistent.

The independent validator recalculates the total from the line items and compares it with the stated total. It therefore detects a discrepancy that can exist even when the extracted data is structurally valid.

**Evidence:** `02-mortgage-extraction/document-runs.txt`.

### 2b. Refusing to fabricate missing values

The missing-bonus fixture demonstrates why absent information should remain `null` instead of being replaced with an invented number. A null value represents information not stated in the document; a numeric zero would imply that the document established a zero amount.

This distinction matters for downstream decisions because missing evidence and a confirmed zero are different facts.

The schema's nullable-field design supports representing missing values explicitly.

**Evidence:** `02-mortgage-extraction/document-runs.txt` and the passing tests in `02-mortgage-extraction/tests.txt`.

### 2c. Normalization

The appraisal fixture demonstrates the extraction of `gross_living_area_sqft` as the numeric value `2400` from informal measurement wording.

Normalizing values during extraction gives downstream validation and reporting components a consistent data type. It avoids requiring every consumer to independently parse informal strings.

**Evidence:** `02-mortgage-extraction/document-runs.txt`.

---

## 3. Multi-source supply-chain investigation

| Evidence | Result |
|---|---|
| Passing test count | 34 passed — `03-supply-chain/tests.txt` |
| Ordinary investigation | `03-supply-chain/investigation-run.txt` |
| Simulated timeout | `03-supply-chain/timeout-run.txt` |
| Static checks | `03-supply-chain/static-checks.txt` |

### 3a. Annotate, don't arbitrate

The ordinary investigation reports conflicting `on_time_delivery_rate` values:

- `95.0` percent from `supplier_audit`, as of `2026-04-10`.
- `78.0` percent from `logistics`, as of `2026-04-05`.

Preserving both values, their sources, and their dates is more informative than silently averaging them into one number. The difference is itself important evidence and can be escalated for investigation.

A synthesized average could hide the disagreement and make the metric appear more settled than it actually is.

**Evidence:** `03-supply-chain/investigation-run.txt`.

### 3b. When a source times out

The timeout run identifies logistics as unavailable and marks `late_shipment_count` as incomplete because the source timed out. The ordinary run instead reports `late_shipment_count` as `11.0 shipments` from logistics under the well-established section.

This difference shows why an unavailable source must not be interpreted as reporting zero. A timeout represents missing evidence, whereas conflicting reported values represent evidence that disagrees.

The investigation can still complete using the remaining available sources, while making the limitation visible to the reader.

**Evidence:** `03-supply-chain/timeout-run.txt` and `03-supply-chain/investigation-run.txt`.

### 3c. Dates as a guardrail

The ordinary briefing attaches source dates to reported values, including the supplier-audit delivery-rate reading dated `2026-04-10` and the logistics reading dated `2026-04-05`.

Dates help readers understand when each observation was made. They provide important context for determining whether differences represent a genuine conflict, a change over time, or both.

**Evidence:** `03-supply-chain/investigation-run.txt`.

---

## 4. Synthesis

### 4a. Evaluate the output, not the model's confidence

The clearest example is the insurance calibration report's umbrella/exclusions slice: confidence `0.93`, accuracy `0.00`, and Brier score `0.865`.

A system that trusted self-reported confidence without evaluating observed outcomes could miss this failure. Slicing the calibration results by policy type and field exposes the concern that an aggregate metric alone can hide.

The sample contains only two observations, so this finding should guide further testing rather than be treated as a definitive estimate of future performance.

### 4b. Confidence is not correctness

The insurance routing test demonstrates that a high-confidence extraction can still require human review when the independent reviewer disagrees. The calibration report independently shows a high-confidence slice with zero observed accuracy.

Together, these results support using independent validation and explicit routing rules rather than allowing confidence alone to determine whether an extraction is accepted.

### 4c. Applying these patterns to a real workflow

One practical example is extracting invoice details from a shared accounts-payable inbox. Invoices vary in layout, line items, and terminology, so an LLM can produce structured fields while deterministic checks verify totals and required information.

I would use schema validation and deterministic checks first, followed by independent review and routing for high-impact disagreements. Missing source information should be escalated rather than fabricated.

I would monitor schema-validation failures, arithmetic discrepancies, reviewer disagreements, escalation rates, calibration by vendor and field, source/API errors, and the percentage of records requiring human review. These measurements would help identify regressions and failures that a single overall accuracy figure could hide.

---

## 5. Limitations and next steps

1. Retest live insurance API behavior only after resolving the authentication problem. Do not include credentials in evidence files.
2. Investigate the umbrella/exclusions calibration result using additional labeled examples.
3. Preserve the actual offline test results and clearly identify the three skipped live API tests.
4. Check the final submission against the capstone README for required routing outputs, run artifacts, and screenshots.
5. Keep all reported results traceable to the saved files in `/workspace/capstone-submission`.
