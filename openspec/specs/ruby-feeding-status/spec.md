# ruby-feeding-status Specification

## Purpose

Derives, in a single human-readable sentence, whether the dog currently needs feeding. Every
feeding reminder — on the display and out loud — reads this one status rather than recomputing it,
so the visual and spoken reminders can never disagree.

Design rationale, including why an optimistic status makes this unsuitable as a rotation trigger,
lives in `docs/specs/ruby-care-system.md`.

## Requirements

### Requirement: Two Feeding Periods Per Day
The system SHALL divide the day into a morning and an evening feeding period, each with a
preferred window inside it and a maximum tolerable interval since the last feeding.

#### Scenario: Period boundaries
- **WHEN** the current time falls in the morning period
- **THEN** feeding status is evaluated against the morning window and interval

### Requirement: Status Reflects Whether Feeding Is Outstanding
The system SHALL report a status naming the responsible child whenever feeding is outstanding, and
a settled status otherwise.

#### Scenario: Already fed this period
- **WHEN** a feeding was recorded within the current period
- **THEN** the status reports that the dog has been fed

#### Scenario: Inside the preferred window and not yet fed
- **WHEN** the current time is inside the preferred window
- **AND** no feeding has been recorded this period
- **THEN** the status reports that it is time to feed, naming the responsible child

#### Scenario: Preferred window has passed
- **WHEN** the preferred window has passed
- **AND** no feeding has been recorded this period
- **THEN** the status reports the dog as unfed, naming the responsible child

#### Scenario: Maximum interval exceeded
- **WHEN** the time since the last recorded feeding exceeds the maximum interval for the period
- **THEN** the status reports the dog as unfed regardless of the window

#### Scenario: Before the window opens
- **WHEN** the current period has begun but its preferred window has not yet opened
- **AND** the maximum interval has not been exceeded
- **THEN** the status reports that the dog has been fed

### Requirement: Status Is Usable Before Any Feeding Is Recorded
The system SHALL produce a valid status when no feeding has ever been recorded, rather than
becoming unavailable.

#### Scenario: No feeding ever recorded
- **WHEN** no feeding timestamp exists
- **THEN** the status renders without error

### Requirement: Time Since Feeding Is Available For Display
The system SHALL expose a human-readable elapsed time since the last feeding alongside the status,
for use by dashboards.

#### Scenario: Reading elapsed time
- **WHEN** a feeding has been recorded
- **THEN** an elapsed-time description is available alongside the status
