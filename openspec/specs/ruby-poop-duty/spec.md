# ruby-poop-duty Specification

## Purpose

Derives whether the backyard needs picking up, on a daily expectation, escalating when it has been
skipped. Mirrors the shape of feeding status so both chores behave consistently.

Design rationale lives in `docs/specs/ruby-care-system.md`.

## Requirements

### Requirement: Daily Expectation With An Afternoon Reminder
The system SHALL treat the chore as due once per day, and SHALL only prompt for it inside a
configured reminder window rather than from the start of the day.

#### Scenario: Already done today
- **WHEN** a pickup was recorded today
- **THEN** the status reports the yard as clean

#### Scenario: Not done, inside the reminder window
- **WHEN** no pickup has been recorded today
- **AND** the current time is inside the reminder window
- **THEN** the status prompts for the chore, naming the responsible child

#### Scenario: Not done, before the reminder window
- **WHEN** no pickup has been recorded today
- **AND** the reminder window has not yet opened
- **THEN** the status reports the yard as clean

### Requirement: Escalation When Skipped
The system SHALL escalate the status when the chore has not been done for two or more days, and
SHALL do so irrespective of the time of day.

#### Scenario: Two days without a pickup
- **WHEN** the last recorded pickup was two or more days ago
- **THEN** the status reports the chore as overdue, naming the responsible child
- **AND** it does so whether or not the reminder window is open

### Requirement: Reminder Window Is Independently Adjustable
The system SHALL express the reminder window as its own entity so that its boundaries can be
inspected and changed without editing the status logic.

#### Scenario: Adjusting the window
- **WHEN** the reminder window's start time is changed
- **THEN** the status begins prompting from the new time, with no other change

### Requirement: Status Is Usable Before Any Pickup Is Recorded
The system SHALL produce a valid status when no pickup has ever been recorded.

#### Scenario: No pickup ever recorded
- **WHEN** no pickup timestamp exists
- **THEN** the status renders without error
