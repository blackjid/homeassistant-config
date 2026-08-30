# ruby-display-sync Specification

## Purpose

Mirrors chore status onto the pixel display in the living room, so an outstanding chore is visible
without opening an app. Reads the chore status capabilities and never computes status itself.


## Why It Is Built This Way

**Persistent versus transient is the central decision, and it is about hardware.** The display is a
small microcontroller that re-renders every entry in its rotation on every pass. A persistent entry
is therefore a real running cost, spent only where persistence is the point: something that needs
attention until it is dealt with. Anything already resolved — a thank-you, an announcement — is
transient and leaves nothing behind.

**The display is driven over a message broker, not over HTTP.** The home automation system cannot
reliably reach the device directly: it holds an interface on the device's network, but source-based
routing keeps that subnet out of the routing table its own outbound connections consult, so they
leave by the wrong path and time out. The device holds a persistent outbound connection to the
broker, so that gap is never touched. This was not discovered by design — the previous firmware was
broker-driven, so the routing gap had always existed and had simply never been exercised.

**The device forgets its entries when it reboots.** Nothing else would re-send them, so a stale
entry could outlive its cause or a live one vanish silently. Resynchronising on start fixes both
directions; the thank-you is suppressed on that path so a restart does not congratulate anyone.

**Updating only on a real change is not an optimisation.** A status entity here emits update events
whose value has not changed, several times an hour. Without comparing values, the device is
re-rendered repeatedly for no reason.

**The payload contract is strict.** Keys are rejected wholesale rather than ignored — an unknown key
fails the entire message — and durations are milliseconds. Icons are files stored on the device and
referenced by name, so messages stay small; sending image data inline on every update would be far
more expensive for the device to handle.

**A deleted entry leaves its position behind.** The device remembers the slot so a returning entry
reappears in the same place. It shows up as an empty entry in the device's own interface. This is
cosmetic and expected, not a leak.

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
