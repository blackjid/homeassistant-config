# ruby-display-sync Specification

## Purpose

Mirrors chore status onto the pixel display in the living room, so an outstanding chore is visible
without opening an app. Reads the chore status capabilities and never computes status itself.

Design rationale — the notification-versus-app distinction, the transport choice, and the
resync-on-restart requirement — lives in `docs/specs/ruby-care-system.md`.

## Requirements

### Requirement: Outstanding Chores Are Persistent On Screen
The system SHALL represent an outstanding chore as a persistent entry in the display's rotation,
so it remains visible until the chore is done.

#### Scenario: A chore becomes outstanding
- **WHEN** a chore status changes to outstanding
- **THEN** a persistent entry for that chore appears in the display's rotation
- **AND** the chore's indicator is lit

### Requirement: Completed Chores Are Announced Once, Then Leave Nothing Behind
The system SHALL represent a completed chore as a transient notification, and SHALL remove the
persistent entry and clear the indicator.

#### Scenario: A chore is completed
- **WHEN** a chore status changes to settled
- **THEN** a transient thank-you naming the responsible child is shown once
- **AND** the persistent entry is removed
- **AND** the chore's indicator is cleared

### Requirement: The Two Chores Are Visually Distinct
The system SHALL give each chore its own indicator so the two can be told apart across a room.

#### Scenario: Both chores outstanding
- **WHEN** both chores are outstanding
- **THEN** each has its own persistent entry and its own indicator

### Requirement: Urgency Is Visible
The system SHALL vary the colour, and where applicable the blink, of the entry and indicator
according to how overdue the chore is.

#### Scenario: Chore becomes badly overdue
- **WHEN** a chore reaches its escalated state
- **THEN** its entry and indicator use the escalated appearance

### Requirement: The Display Is Resynchronised After A Restart
The system SHALL restore the display to match current chore status when the home automation
system starts, because the display does not retain pushed entries across its own reboot.

#### Scenario: Restart while a chore is outstanding
- **WHEN** the system starts
- **AND** a chore is outstanding
- **THEN** the persistent entry and indicator are restored

#### Scenario: Restart while all chores are settled
- **WHEN** the system starts
- **AND** no chore is outstanding
- **THEN** any stale entry and indicator are cleared
- **AND** no thank-you is announced

### Requirement: The Display Is Not Updated Without Cause
The system SHALL update the display only when a chore's status value actually changes, and not in
response to incidental churn in the underlying data.

#### Scenario: Underlying data updates without a status change
- **WHEN** a status entity emits an update whose status value is unchanged
- **THEN** nothing is sent to the display
