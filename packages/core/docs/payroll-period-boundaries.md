# Payroll Period Boundaries Specification (#877)

## Overview
A payroll period defines the temporal bounding box for work accrual, compensation calculation, pre-flight verification, and final disbursement. Enforcing deterministic period boundaries guarantees accounting accuracy, prevents double-claiming of work intervals, and ensures audit reconcilability.

---

## Period Structure & Timing Model

```
|--- Period Start ---------------- Cutoff Date ---------------- Period End --- Settlement Date --->
     [Accrual Window]                [Review & Proofs]           [Disbursement]
```

### Key Milestones:
1. **Period Start (`start`)**: The exact timestamp (epoch ms) marking the start of the pay cycle.
2. **Cutoff Date (`cutoff`)**: The deadline after which employee hours, adjustments, and bonus allocations are frozen. Must satisfy: $\text{start} < \text{cutoff} \le \text{end}$.
3. **Period End (`end`)**: The formal close of the pay cycle. Must satisfy: $\text{start} < \text{end}$.
4. **Settlement Date (`settlementDate`)**: The scheduled execution date for on-chain fund transfer. Must satisfy: $\text{settlementDate} \ge \text{end}$.

---

## Validation & Boundary Guardrails

- **Duration Bounds**: Minimum duration is 1 hour (3,600,000 ms); maximum duration is 366 days (leap year bound).
- **Non-Overlap Invariant**: Sequential payroll periods for the same employer and asset class cannot overlap:
  $$\text{Period}_B.\text{start} \ge \text{Period}_A.\text{end}$$
- **Lifecycle Immutability**: Once a period reaches `CLOSED` or `ARCHIVED` status, it cannot be modified or reopened without explicit cryptographic override.
- **Late Payroll Adjustment Escalation**: Adjustments submitted after `cutoff` but before `end` are not applied to the current period. They must be escalated as late adjustments and carried into the next period's accrual window, with the escalation recorded for audit. Adjustments submitted after `end` are rejected with an actionable error referencing the period identifier and the `cutoff` timestamp.
- **Escalation Recording**: Each escalation record must capture the originating period identifier, the adjustment submission timestamp, the `cutoff` timestamp of the originating period, and the target next-period identifier. Records are append-only and are retained for audit reconciliation.
- **Escalation Error Reporting**: Rejections for adjustments submitted after `end` must include the period identifier and `cutoff` timestamp so integrators can resubmit correctly. Escalation requests that fail validation must return an error identifying the failing rule (missing period identifier, invalid period status, or archived period reference).
- **Escalation Validation**: An escalation request must reference a period in `CLOSED` or pre-`CLOSED` status and must include the originating period identifier. Requests referencing an `ARCHIVED` period are rejected and must be resubmitted against the next open period. Requests missing the originating period identifier are rejected before any status check is performed.
- **Privacy Model**: Period boundaries disclose only temporal and scheduling metadata. No individual employee compensation details or headcount volumes are exposed. Escalation records reference periods and timestamps only and must not include adjustment amounts or employee identifiers in the period-boundary record.
