# Test Cases — Patient Appointment Booking (Smoke Test)

## TC-001: Booking page returns only unbooked future slots
**Format:** Integration / Given-When-Then

**Given** the database contains 5 slots within the next 30 days, of which 2 already have a booking row
**When** an authenticated patient sends `GET /booking/slots`
**Then** the response is HTTP 200 with a JSON array of exactly 3 slot objects (the 2 booked slots are excluded), each containing `slot_id`, `start_time`, `clinician_id`, and `available: true`

validates::TC->SR [SR-001]

## TC-002: Past slots are excluded from the list
**Format:** Integration / Given-When-Then

**Given** the database contains 1 slot with `start_time` 1 hour in the past and 1 slot with `start_time` 1 hour in the future, neither booked
**When** an authenticated patient sends `GET /booking/slots`
**Then** the response is HTTP 200 and contains only the future slot

validates::TC->SR [SR-002]

## TC-003: Concurrent booking — exactly one wins
**Format:** Integration / property under contention

**Given** an empty `bookings` table and a target `slot_id = S1`
**When** 50 patients send `POST /booking {slot_id: S1}` within a 50ms window
**Then** exactly 1 response is HTTP 201 and exactly 49 responses are HTTP 409; the `bookings` table contains exactly 1 row with `slot_id = S1`

validates::TC->SR [SR-003]

## TC-004: Sequential second booking for same slot returns 409
**Format:** Unit / Given-When-Then

**Given** patient A has already booked `slot_id = S1` (a row exists)
**When** patient B sends `POST /booking {slot_id: S1}`
**Then** the response is HTTP 409 Conflict with body `{"error":"slot_taken"}` and the database is unchanged

validates::TC->SR [SR-003]

## TC-005: Unauthenticated booking is rejected
**Format:** Security / negative assertion

REJECT [SR-004]: `POST /booking` without `Authorization` header → HTTP 401 with body `{"error":"unauthenticated"}`, no row inserted into `bookings`, no SQL query against `bookings` observed

validates::TC->SR [SR-004]

## TC-006: Accepted booking writes audit log within 100ms
**Format:** Integration / temporal assertion

**Given** an empty `audit_log` table
**When** an authenticated patient sends `POST /booking {slot_id: S1}` and receives HTTP 201 at time `T`
**Then** within 100ms of `T`, exactly one row appears in `audit_log` with `patient_id` matching the JWT subject, `slot_id = S1`, `outcome = "accepted"`, and `timestamp` between `T` and `T+100ms`

validates::TC->SR [SR-005]

## TC-007: 409 booking writes audit log with rejected_slot_taken
**Format:** Integration / Given-When-Then

**Given** an empty `audit_log` table and slot S1 already booked
**When** patient B sends `POST /booking {slot_id: S1}` (receives HTTP 409)
**Then** within 100ms, exactly one row appears in `audit_log` with `patient_id` = B's id, `slot_id = S1`, `outcome = "rejected_slot_taken"`

validates::TC->SR [SR-006]

## TC-008: 401 booking writes audit log with rejected_unauthenticated and null patient
**Format:** Integration / Given-When-Then

**Given** an empty `audit_log` table
**When** an unauthenticated request `POST /booking {slot_id: S1}` is sent (receives HTTP 401)
**Then** within 100ms, exactly one row appears in `audit_log` with `patient_id = null`, `slot_id = S1`, `outcome = "rejected_unauthenticated"`

validates::TC->SR [SR-006]
