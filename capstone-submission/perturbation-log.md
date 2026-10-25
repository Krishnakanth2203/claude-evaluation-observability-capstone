# Controlled Perturbation Log

## System 1 — Insurance Policy Extraction

### Baseline
The existing test sets `endorsements` to `None`. The validator reports:
- Field: `endorsements`
- Category: `missing_source`
- Pattern: `endorsements_absent`

### Deliberate change
A separate temporary test changed `exclusions` to `None`, leaving `endorsements` populated.

### Command
From the policy pipeline solution directory:

    PYTHONPATH="$PWD" .venv/bin/pytest -q -s /tmp/test_policy_exclusions_perturbation.py

### Observed result
Two tests passed. The perturbation reported field `exclusions`, category `missing_source`, and pattern `exclusions_absent`. The baseline case reported `endorsements_absent`.

### Interpretation
The validator identifies the specific required field that is missing. The original project tests and source code were not modified by the temporary test.

## System 2 — Mortgage Extraction

### Baseline
With `DEFAULT_TOLERANCE_USD = 1.00`, the income mismatch fixture produced:
- Calculated monthly income: 9642.17 USD
- Stated monthly total: 10892.17 USD
- Delta: -1250.00 USD
- `consistent`: false
- Discrepancy: `total_monthly_income`

Evidence: `02-mortgage-extraction/discrepancy-run.txt`

### Deliberate change
The tolerance was temporarily increased to 1300.00 USD, the same replay fixture was rerun, and the original configuration was restored.

### Observed result
The perturbed run returned exit status 0, `consistent: true`, and no discrepancies. The configuration was restored to `DEFAULT_TOLERANCE_USD = 1.00`.

Evidence: `02-mortgage-extraction/tolerance-perturbation.txt`

### Interpretation
The 1250.00 USD difference fell within the deliberately increased tolerance. This demonstrates that changing the tolerance changes the validation result. The elevated threshold was experimental, not a recommended production setting.

## System 3 — Supply Chain Risk

### Baseline
The recorded investigation reported conflicting `on_time_delivery_rate` values:
- `supplier_audit`: 95.0%, dated 2026-04-10
- `logistics`: 78.0%, dated 2026-04-05

Evidence: `03-supply-chain/investigation-run.txt`

### Deliberate change
The investigation was run with the built-in simulated logistics timeout.

### Observed result
The recorded timeout run reported that logistics was unavailable due to timeout. The `late_shipment_count` result was marked incomplete because that source was unavailable; the baseline reported 11.0 late shipments from logistics.

Evidence: `03-supply-chain/timeout-run.txt`

### Interpretation
A source timeout reduced the completeness of the investigation. The unavailable source was surfaced rather than presenting the result as fully supported.
