# REVIEW.md

Defects found in the starter `app/main.py`, and how each was fixed.

| # | Where | What is wrong | How you'd notice it | How I fixed it |
|---|---|---|---|---|
| 1 | `MongoClient(...)` | Naive datetimes come back from PyMongo; no `tz_aware`. | Every instant is silently interpreted in server-local time. | `tz_aware=True, tzinfo=UTC`; all conversion goes through `to_dt` / `to_ms`. |
| 2 | `GET /health` | Always returns 200, never touches MongoDB. | Stop MongoDB: health still says ok. | Ping MongoDB; 503 on failure. |
| 3 | `create_employee` | Check-then-insert race and no unique index: two parallel creates both succeed. | 10 parallel POSTs of the same `emp_code` give several 201s. | Unique index on `emp_code`; catch `DuplicateKeyError` -> 409. |
| 4 | `EmployeeIn` | No validation: bad `emp_code`, shifts, email, dates, `shift_start == shift_end` all accepted -> should be 422. | POST `emp_code="abc"` or `joined_on="2026-13-45"`. | Field patterns/lengths, real-date validator, model validator for equal shifts. |
| 5 | `create_employee` | `created_at = datetime.now()` is naive local time and is returned as a datetime, not epoch ms. | Response `created_at` is a string/datetime. | UTC, ms precision, serialized with `to_ms`. |
| 6 | `list_employees` | Offset is `page * page_size` (page 1 skips the first page). | `page=1` never returns the first rows. | `(page - 1) * page_size`. |
| 7 | `list_employees` | `total` counts the whole collection, ignoring the `department` filter (R10). | Filter by department: `total` is too large. | `count_documents(q)` with the same filter. |
| 8 | `list_employees` | No sort (contract: `emp_code` ascending); `page` / `page_size` not validated (0, negative, 10000). | `page_size=0`, `page=-3`. | `sort("emp_code")`; `Query(ge=1, le=100)`. |
| 9 | `compute_late_minutes` | (a) Uses `punch_in.replace(hour..)` in the datetime's own zone, not IST. (b) `minutes > 10` after flooring: 09:40:30 should be late (10), the code returns 0. (c) No second truncation. (d) Overnight shifts ignored. | 09:40:01 with shift 09:30 -> expected 10, got 0. | Compare against the shift start on the attendance date in IST, in whole seconds: late iff `secs > 600`, then `secs // 60`. |
| 10 | `compute_work_hours` | `round()` is banker's rounding on a binary float; R4 wants half-up. | 16218 s = 4.505 h -> returns 4.5, should be 4.51. This also flips `half_day`. | `Decimal` division and `ROUND_HALF_UP`. |
| 11 | `compute_overtime` | No 30-minute minimum (R3); takes shift end on `date_str`, wrong for overnight shifts; hard-coded naive handling. | Out 18:45 on 18:30 shift returns 15, expected 0. | `shift_end_dt` (next day when overnight), floor minutes, `>= 30` else 0. |
| 12 | `punch_in` | Unknown employee: `emp` is `None`, `emp["shift_start"]` -> `TypeError` -> 500 (must be 404). | Punch in for `EMP9999`. | 404 before use. |
| 13 | `punch_in` | `datetime.fromtimestamp(ms/1000)` uses server-local time, keeps sub-seconds, date is not the IST date, no overnight rule (R1). `if body.punched_at` treats `0` as "missing". | Server in UTC: a 23:30 IST punch gets the wrong date. | `to_dt` (UTC, truncated to the second) + `attendance_date` (IST, overnight aware); `is not None`. |
| 14 | `punch_in` | Check-then-insert race (find_one, then insert): two parallel requests both get 201. | 10 parallel identical punch-ins. | Unique index `(emp_code, date)`; `DuplicateKeyError` -> 409. |
| 15 | `punch_in` / `PunchInIn` | `status` is any string (ABSENT/LEAVE/garbage accepted); `punched_at` not validated (seconds, floats, strings, bool). | `status="LEAVE"`, `punched_at=1783312500`. | `Literal[PRESENT, WFH, ON_DUTY]`; strict-int `EpochMs` with range. |
| 16 | `punch_in` response | Leaks `id` (Mongo `_id`) and returns raw datetimes instead of epoch ms. | Response contains `"id"`. | `serialize_attendance`. |
| 17 | `list_attendance` | `find(q)` loads the entire collection into memory, sorts and slices in Python. | 100k docs: slow, memory heavy. | `find().sort().skip().limit()` with indexes. |
| 18 | `list_attendance` | Sort is `date` only (needs `date` desc, then `emp_code` asc); unstable pages. | Same date, different order between calls. | Sort `[(date,-1),(emp_code,1)]`, backed by index `(date -1, emp_code 1)`. |
| 19 | `list_attendance` | `date_from > date_to` and malformed dates / invalid status not rejected (422); page and page_size unvalidated. | `date_from=2026-08-01&date_to=2026-07-01`. | Pattern + real-date + range checks; `Literal` status; `Query` bounds. |
| 20 | `list_attendance` | Returns `_id` as `id`, datetimes raw, legacy docs missing `half_day`/`history` leak through. | Fetch the legacy sample record. | `serialize_attendance` defaults (`0`, `False`, `[]`) and ms conversion. |
| 21 | module level | No indexes at all. | `explain` shows COLLSCAN. | Idempotent `ensure_indexes()` at startup. |

**Looked at and judged fine:** `load_dotenv()` (does not override real environment variables, which is what the contract asks); returning plain dicts from sync endpoints (FastAPI runs them in a thread pool and `MongoClient` is thread-safe); `doc.pop("_id")` after insert (works, but replaced by explicit serializers).
