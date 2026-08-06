---
name: Exercise the Stream enrolment flow in the sandbox
description: >-
  Drive an end-to-end enrolment against the Wagestream Integrations API sandbox using the published dummy
  bank accounts and the test-enrolment trigger, without a real employee installing the app.
api: openapi/wagestream-integrations-api-openapi.yml
generated: '2026-08-05'
method: generated
source: openapi/wagestream-integrations-api-openapi.yml
operations:
  - POST /employees
  - POST /testenrollments
  - GET /enrollments
---

# Exercise the Stream enrolment flow in the sandbox

Use `https://publicapi.wagestream.io/pushapi-staging` for everything below. It takes the same
`x-api-key` header shape as production, but the key is issued separately by your Stream contact — there is
no self-serve sandbox signup.

## Step 1 — Seed a test employee with a bank account that will validate

`POST /employees` with an `EmployeeList`. The account details matter: Stream runs a modulus check and an
FPS check, and a made-up account fails both. Use one of the **published dummy accounts** from
`sandbox/wagestream-sandbox.yml`, for example:

| Bank | Sort code | Account number |
|---|---|---|
| Barclays Bank | 200052 | 75849855 |
| Lloyds TSB Bank | 309493 | 01273801 |
| Santander Bank | 090126 | 03367219 |

These are designed to pass Stream's validation checks. Poll `GET /employees?txn_id=<id>` and confirm no
`MODULUS_CHECK_FAILED` or `NOT_FPS_ENABLED` appears on the row.

## Step 2 — Trigger the enrolment

`POST /testenrollments?employee_ids=<comma,delimited,list>`.

Add `pending=true` **only** if you are implementing the `Enterprise` or `Instant Verification` enrolment
flows — it simulates the employee registering in the app and accepting terms, leaving them in `PENDING` so
you can practise the "poll for pending, then complete by POSTing the rest of the employee data" loop. For
the `Standard` flow the parameter is redundant.

## Step 3 — Watch the enrolment move

`GET /enrollments?changes_only=true` and follow the `action` field through `PENDING` → `TO_COMPLETE`.
Confirm your payroll-side logic writes the Stream-generated `bank_sort_code` / `bank_account_number` onto
the employee, then confirm with `POST /enrollments?employee_id=<id>`.

## Step 4 — Test the unwind

Set a `termination_date` on the test employee and re-push via `POST /employees`. Confirm the enrolment
record turns up with `action: TO_UNENROLL` and that your integration restores the employee's original bank
details. This is the path most integrations never test.

## Test the idempotency contract too

Submit the same `nonce` twice on any write and confirm your client treats the **409 "Nonce was already
used"** as success-of-the-original rather than as an error to retry.
