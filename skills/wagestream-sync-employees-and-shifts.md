---
name: Sync employees and shifts into Stream (Wagestream)
description: >-
  Push the employee roster and worked shifts from an HR/WFM/payroll system into the Wagestream
  Integrations API so employees accrue earned wage access, then poll the async transaction status and
  resolve per-row validation errors.
api: openapi/wagestream-integrations-api-openapi.yml
generated: '2026-08-05'
method: generated
source: openapi/wagestream-integrations-api-openapi.yml
operations:
  - POST /employees
  - GET /employees
  - post_shifts
  - get_shifts
---

# Sync employees and shifts into Stream

This is the core feed. Employees define who exists; shifts define how much they have earned and when.
Without both, no employee can access earned wages.

## Before you start

- Base URL: `https://publicapi.wagestream.io/pushapi-prod` (sandbox: `.../pushapi-staging`).
- Every request must carry the header `x-api-key: <your key>`. Without it you get **403 Forbidden**.
  Keys are issued by a Stream Client Success Manager — they are not self-serve.
- Budget: **300 requests per client per day**, not per endpoint. Batch aggressively.

## Step 1 — Upsert the employee roster

`POST /employees` with an `EmployeeList` body: `{ "employees": [ ... ], "nonce": "<unique>" }`.

- `employee_id` is the only required field and is **your** payroll number, not a Stream id. It is the
  join key for every other entity.
- Send **one record per assignment** when an employee holds multiple positions, each with a distinct
  `assignment_id`.
- Include `bank_account_number` and `bank_sort_code` — an employee cannot complete enrolment without a
  valid, FPS-enabled UK account (or EU IBAN/BIC).
- Set `nonce` to a value unique to this request. Replaying the same nonce returns **409** — treat that as
  "already accepted", not as a failure.
- Re-sending an existing `employee_id` **updates** it. This is an upsert, so a full re-sync is safe.

The response is a `TransactionResponse`: `{ ok, rows, txn_id }`. Keep the `txn_id`.

## Step 2 — Upsert worked shifts

`post_shifts` — `POST /shifts` with a `ShiftList` body: `{ "shifts": [ ... ], "nonce": "<unique>" }`.

- Required per shift: `employee_id`, `shift_id`, `worked_on` (YYYY-MM-DD), `wages` (gross, 2dp),
  `hours`.
- `shift_id` exists specifically to de-duplicate resubmission — the same `shift_id` updates, a new one
  creates.
- Optional `type` is `STANDARD` or `OVERTIME`; `rate`, `started_at`, `ended_at`, `info` are informational.
- On first load, supply up to **45 days** of history, minimum one shift per day worked per employee.
- Batch: 30 shifts in one call every 15 minutes beats one shift a minute, and stays inside the daily cap.

Response is again `TransactionResponse` with a `txn_id`.

## Step 3 — Poll for the result (this API is asynchronous)

Nothing is validated per-field on the write. Call the matching GET with the `txn_id`:

- `GET /employees?txn_id=<id>`
- `get_shifts` — `GET /shifts?txn_id=<id>`

You get a `TransactionStatus`: `state` is `queued`, `processed` or `failed`. Processing usually completes
within **3 minutes**. `results[]` holds one `TransactionResponseItemStatus` per row, where `item_id` is
the record's own `employee_id`/`shift_id` and `status[]` carries the failure codes for that row.

## Step 4 — Resolve row errors

Look each code up in `errors/wagestream-error-codes.yml`. The ones that actually block earned wage access:

- `MODULUS_CHECK_FAILED` — the sort code / account number pair is not a real UK account.
- `NOT_FPS_ENABLED` — the account cannot receive Faster Payments, so wages cannot be streamed.
- `DUPLICATE_EMAIL_ADDRESS` — the email is already on another employee.
- `REQUIRED_PROPERTY`, `INVALID_DATE_FORMAT`, `INVALID_NUMBER_FORMAT`, `STRING_EXCEEDS_MAX_LENGTH` —
  mapping bugs in your extract; fix and resubmit the affected rows only.

## Step 5 — Run a daily full sync

Even if you send deltas, run a **full sync at least once a day**. A change that silently failed on an
incremental push is self-healed by the next full sync. Because every write is an upsert keyed on your own
ids, a full sync is idempotent by construction.
