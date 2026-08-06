---
name: Reconcile Stream enrolments back into payroll
description: >-
  Read the Stream-generated enrolment banking records from the Wagestream Integrations API and apply them
  to the employer's payroll system so an enrolled employee's wages route through Stream, and unwind them
  cleanly when the employee opts out or leaves.
api: openapi/wagestream-integrations-api-openapi.yml
generated: '2026-08-05'
method: generated
source: openapi/wagestream-integrations-api-openapi.yml
operations:
  - GET /enrollments
  - post_enrollments
---

# Reconcile Stream enrolments back into payroll

This is the one flow that runs **from** Stream **to** you. Getting it wrong means an employee is paid to
the wrong account, so treat it as the highest-consequence integration in the set.

## What an enrolment record is

When an employee enrols, Stream mints a reconciliation bank account for them. Your payroll system must
**replace** that employee's own `bank_sort_code` and `bank_account_number` with the Stream-generated pair
so their salary lands at Stream, which then streams it to them. `bank_name` reads `Stream`/`wagestream`
once enrolled, and `Unknown Bank` while Stream is still waiting on the employee's personal bank details.

## Step 1 — Poll the enrolment feed

`GET /enrollments` with `x-api-key`. Recommended cadence: **every 4 hours**.

Query parameters:

- `changes_only=true` — return only records that need you to act. Use this for the routine poll; it is the
  cheapest call and keeps you inside the 300/day cap.
- `banking_only=true` — return only fully enrolled employees that have banking details.
- `limit` + `page` — paging. `page` is **1-indexed, not 0**. The envelope returns `total_records` and
  `page`; records are ordered by `requested_on` ascending (oldest first).

## Step 2 — Act on `action`

Each `enrollment` carries an `action` of `NONE`, `PENDING`, `TO_COMPLETE` or `TO_UNENROLL`.

- `TO_COMPLETE` — write the Stream `bank_sort_code` / `bank_account_number` onto the employee in payroll.
- `TO_UNENROLL` — restore the employee's original personal bank details. This is the step people forget;
  leaving a leaver pointed at Stream strands their final pay.
- `PENDING` — the employee has started but Stream does not yet hold their personal bank details. Make sure
  those fields are populated on the employee record via `POST /employees`.
- `NONE` — no action.

The dates `requested_on`, `enrolled_on` and `unenrolled_on` let you audit the transition.

## Step 3 — Confirm completion

`post_enrollments` — `POST /enrollments?employee_id=<id>` notifies Stream that you have completed the
enrolment for that employee. Enrolment is only finished once the employee's record in the `/employees`
feed carries their Stream reconciliation account.

## Retention

Leavers drop out of the enrolment feed **60 days** after becoming a leaver. Reconcile before then — after
that the record is gone and you cannot recover the unenrol instruction from this API.

## Errors

`403` means the key is missing or wrong. This endpoint has no 409/422 contract because it is a read plus a
single-parameter notify. Row-level problems surface on the `/employees` feed, not here.
