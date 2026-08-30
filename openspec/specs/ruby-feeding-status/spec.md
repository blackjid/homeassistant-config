# ruby-feeding-status Specification

## Purpose

Derives, in a single human-readable sentence, whether the dog currently needs feeding. Every
feeding reminder — on the display and out loud — reads this one status rather than recomputing it,
so the visual and spoken reminders can never disagree.


## Why It Is Built This Way

**The status is deliberately optimistic before a window opens.** Between the start of a period and
the opening of its preferred window, the dog is reported as fed. This keeps the display quiet during
the hours when nobody would expect to feed her. The consequence is important and easy to miss: a
status of "fed" does not mean a feeding happened, so this status must never be used as a trigger for
anything that records or rotates. Use the feeding timestamp for that.

**Two failure modes were fixed and should not come back.** The status compared a null against a
number when no feeding had ever been recorded, which made the whole sensor unavailable on a fresh
install rather than merely uninformative. And the elapsed-time description exposed alongside the
status updates on its own cadence, producing update events where the status text has not changed —
consumers must compare values rather than reacting to every update.

**The elapsed-time description is a presentation concern that a dashboard depends on.** It is the
reason this capability cannot be expressed with a plain UI helper, which has no way to attach extra
values to a status. That constraint is worth knowing before anyone tries to simplify it.

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
