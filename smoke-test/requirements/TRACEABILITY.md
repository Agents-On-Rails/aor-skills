# Traceability — Patient Appointment Booking (Smoke Test)

## Coverage Matrix

| UN | PR | SR | SR Clauses | TC | Coverage |
|----|----|----|-----------|----|----------|
| UN-001 | PR-001 | SR-001 | (single)  | TC-001 | ✅ covered |
| UN-001 | PR-001 | SR-002 | (single)  | TC-002 | ✅ covered |
| UN-001 | PR-003 [ADV] | SR-004 | (single)  | TC-005 | ✅ covered |
| UN-002 | PR-002 | SR-003 | (single)  | TC-003, TC-004 | ✅ covered |
| UN-002 | PR-004 [ADV] | SR-005 | (single)  | TC-006 | ✅ covered |
| UN-002 | PR-004 [ADV] | SR-006 | (single)  | TC-007, TC-008 | ✅ covered |

All SRs use single-statement form (no clause notation needed in this small spec).

## Gaps

None — every SR has at least one TC.

## Validation Plan

| UN | Validator | Scenario | Question |
|----|-----------|----------|----------|
| UN-001 | Patient (recruited usability tester) | Open the booking page, find an available slot 7 days out, complete a booking, verify confirmation email | Does this serve the user need described in UN-001 (self-service booking at any hour)? |
| UN-002 | Clinic receptionist | Run a 30-minute parallel booking drill with two devices targeting the same slot; review the `audit_log` for completeness | Does this serve the user need described in UN-002 (single-occupant guarantee + auditability)? |
