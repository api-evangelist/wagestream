---
name: Send absences and off-cycle payments to Stream
description: >-
  Keep an employee's earned wage balance accurate by pushing pay-impacting leave into the Wagestream
  Integrations API, and notify Stream of payments made outside the normal pay run.
api: openapi/wagestream-integrations-api-openapi.yml
generated: '2026-08-05'
method: generated
source: openapi/wagestream-integrations-api-openapi.yml
operations:
  - post_absences
  - get_absences
  - post_offcycle_payment
  - get_expenses
---

# Send absences and off-cycle payments to Stream

Both feeds exist to stop an employee accessing wages they will not actually be paid. Skipping either one
produces over-accrual, which becomes a payroll deduction the employee did not expect.

## Absences — pause or adjust accrual

`post_absences` — `POST /absences` with an `AbsenceList` body: `{ "absences": [ ... ], "nonce": "<unique>" }`.

Required per record: `employee_id`, `absence_id`, `type`, `reason`.

- `type` is `Paid` or `Unpaid`. **Unpaid leave typically pauses salary accrual entirely.**
- `reason` is the leave category — Unauthorised, Training, Disciplinary, Sabbatical, Maternity, Jury
  Service, Long Term Sick — and is what lets Stream pause on a specific reason only.
- `started_at` / `ended_at` are YYYY-MM-DD. `ended_at` also drives un-pausing.
- `absence_id` is the upsert key: same id updates the record, a new id creates one.
- Supply up to **45 days** of historical absence on first load.

Poll with `get_absences` — `GET /absences?txn_id=<id>`.

## Off-cycle payments — tell Stream about money paid outside the pay run

`post_offcycle_payment` — `POST /off-cycle-payments` with an `off-cycle-payment-list` body:
`{ "payments": [ ... ], "nonce": "<unique>" }`.

Required per record: `employee_id`, `amount` (2dp), `expected_date`.

This is the highest-consequence write in the API: it moves real money expectations. Send it once, send it
with a `nonce`, and if you get **409 Nonce was already used**, the first submission was accepted — do
**not** retry with a fresh nonce, or you will double-book the payment.

Poll with `get_expenses` — `GET /off-cycle-payments?txn_id=<id>`.

## Shared rules

- Header `x-api-key` on every call, else 403.
- 422 means the envelope did not parse. Row-level problems come back asynchronously in
  `TransactionStatus.results[]`, keyed by `absence_id` or `employee_id`.
- Check codes against `errors/wagestream-error-codes.yml`; `INVALID_ENUM_VALUE` on an absence almost
  always means `type` was not `Paid`/`Unpaid`.
- Batch. The 300 requests/day cap is per client across all endpoints.
