# Specification — Patient Appointment Booking (Smoke Test)

This is a toy specification authored to dogfood the aor-bundle skills. It is
intentionally small (2 UN, 4 PR, 6 SR, 8 TC) and not a real product spec.

## Personas

- `patient` — the end user booking an appointment
- `clinic-staff` — the receptionist managing the schedule

## User Needs (UN)

### UN-001: Patients can book a time that suits them
**Problem:** Patients today phone the clinic during business hours, often
get a busy line, and book whatever slot the receptionist offers. They want
to see available slots and pick one without speaking to anyone.
**Need:** Patients need a self-service way to view available appointment
slots and book one for themselves at any hour.
**Priority:** High
traces_to::UN->Persona [patient]

### UN-002: Clinic staff trust the schedule is internally consistent
**Problem:** When two patients call within seconds of each other for the
same slot, the receptionist can double-book by accident. Staff need
confidence that the system never schedules two patients into the same
clinician slot.
**Need:** Clinic staff need the booking system to enforce a single occupant
per slot and to log every booking attempt for audit.
**Priority:** Critical
traces_to::UN->Persona [clinic-staff]

## Product Requirements (PR)

### PR-001: Show available slots
When a patient opens the booking page, the system shall display all
appointment slots that are not yet booked, within the next 30 days.
derived_from::PR->UN [UN-001]
traces_to::PR->SR [SR-001, SR-002]

### PR-002: Single-occupant slot enforcement
When two booking requests target the same slot within any time window, the
system shall accept exactly one and reject the other.
derived_from::PR->UN [UN-002]
traces_to::PR->SR [SR-003]

### PR-003 [ADV]: Reject booking when patient is unauthenticated
If a booking request arrives without a valid patient session, then the
system shall reject the request with HTTP 401 and shall not create a
booking record.
derived_from::PR->UN [UN-001]
traces_to::PR->SR [SR-004]

### PR-004 [ADV]: Audit-log every booking attempt
The system shall record an audit entry for every booking attempt, whether
accepted or rejected, including patient identifier, slot identifier, and
outcome.
derived_from::PR->UN [UN-002]
traces_to::PR->SR [SR-005, SR-006]

## Software Requirements (SR)

### SR-001: Booking-page slot list
When the patient loads `GET /booking/slots`, the booking-service shall
return a JSON array of slot objects with `slot_id`, `start_time`,
`clinician_id`, and `available: true`, filtered to slots within the next
30 days where no booking row exists.
derived_from::SR->PR [PR-001]
traces_to::SR->TC [TC-001]

VERIFY: [SR-001] — `GET /booking/slots` returns only slots with no matching booking row and `start_time` within now+30d.

### SR-002: Slot list excludes past slots
While the current time exceeds a slot's `start_time`, the booking-service
shall not include that slot in the response of `GET /booking/slots`.
derived_from::SR->PR [PR-001]
traces_to::SR->TC [TC-002]

VERIFY: [SR-002] — slots with `start_time < now()` are absent from the API response.

### SR-003: Single-occupant slot enforcement (database constraint)
The booking-service shall enforce a UNIQUE constraint on `bookings.slot_id`
in the database such that any second `INSERT` for the same `slot_id` fails
with a constraint-violation error code, which the service shall translate
into an HTTP 409 Conflict response.
derived_from::SR->PR [PR-002]
traces_to::SR->TC [TC-003, TC-004]

VERIFY: [SR-003] — concurrent inserts on same slot_id: exactly one returns 201, the other returns 409.

### SR-004: Reject unauthenticated booking
If a `POST /booking` request lacks a valid `Authorization: Bearer <jwt>`
header, then the booking-service shall return HTTP 401 with body
`{"error":"unauthenticated"}` and shall not call the database.
derived_from::SR->PR [PR-003]
traces_to::SR->TC [TC-005]

VERIFY: [SR-004] — `POST /booking` without bearer token: 401 returned, no DB call observable.

### SR-005: Audit log on accepted booking
When a `POST /booking` request results in an HTTP 201 response, the
booking-service shall write an entry to `audit_log` with
`patient_id`, `slot_id`, `outcome: "accepted"`, and ISO-8601 `timestamp`,
within 100ms of the response.
derived_from::SR->PR [PR-004]
traces_to::SR->TC [TC-006]

VERIFY: [SR-005] — for every 201 response, an audit_log row exists within 100ms with outcome=accepted.

### SR-006: Audit log on rejected booking
When a `POST /booking` request results in an HTTP 401, 403, or 409
response, the booking-service shall write an entry to `audit_log` with
`patient_id` (or `null` if unauthenticated), `slot_id`,
`outcome: "rejected_<reason>"`, and ISO-8601 `timestamp`, within 100ms of
the response.
derived_from::SR->PR [PR-004]
traces_to::SR->TC [TC-007, TC-008]

VERIFY: [SR-006] — for every 401/403/409 response, an audit_log row exists within 100ms with outcome=rejected_<reason>.

## Out of Scope

- Multi-clinician scheduling conflicts (one clinician → multiple rooms)
- Cancellation flow (separate spec)
- SMS/email notifications

## Open Questions

- Should `audit_log` be append-only (no UPDATE/DELETE)? — pending compliance review.
