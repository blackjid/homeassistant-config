# ruby-voice-reminder Specification

## Purpose

Speaks a personalised reminder when the camera recognises the child whose turn it is and a chore is
outstanding, and brings the matching chore onto the display so the spoken reminder has something to
point at. A supplement to the display, not a replacement — the display is ambient and easy to stop
seeing.

Design rationale — the sticky-sensor trigger, the two silent-failure traps, and why the quota reset
has no start-up trigger — lives in `docs/specs/ruby-voice-reminder.md`.

## Requirements

### Requirement: Announcements Are Aimed At The Responsible Child
The system SHALL announce only when the recognised person is the child currently holding the care
turn.

#### Scenario: The responsible child is recognised
- **WHEN** the recognised person matches the current care turn
- **AND** at least one chore is outstanding
- **THEN** a spoken reminder naming that child is played

#### Scenario: A different family member is recognised
- **WHEN** the recognised person does not match the current care turn
- **THEN** nothing is announced

#### Scenario: Someone outside the roster is recognised
- **WHEN** the recognised person is not on the care roster
- **THEN** nothing is announced

### Requirement: Announcements Only Happen When Work Is Outstanding
The system SHALL NOT announce when both chores are already settled.

#### Scenario: Responsible child recognised with nothing to do
- **WHEN** the responsible child is recognised
- **AND** no chore is outstanding
- **THEN** nothing is announced

### Requirement: Every Sighting Is Detected
The system SHALL detect a sighting of the responsible child even when the previously recognised
person was the same child.

#### Scenario: The same person is seen twice in a row
- **WHEN** the responsible child is recognised
- **AND** is recognised again with no other person in between
- **THEN** the second sighting is detected as an eligible trigger

### Requirement: Announcements Are Capped Per Period
The system SHALL allow at most two announcements per feeding period, and SHALL refill that
allowance at the start of each period.

#### Scenario: Third eligible sighting in one period
- **WHEN** two announcements have already been made in the current period
- **THEN** further eligible sightings produce no announcement

#### Scenario: A new period begins
- **WHEN** a feeding period starts
- **THEN** the allowance is refilled

### Requirement: The Allowance Is Not Refilled By A Restart
The system SHALL NOT refill the announcement allowance as a side effect of a restart.

#### Scenario: Restart mid-period with the allowance spent
- **WHEN** the allowance is spent
- **AND** the system restarts within the same period
- **THEN** the allowance remains spent

### Requirement: Announcements Are Spaced Apart
The system SHALL enforce a minimum interval between announcements, so a single busy moment cannot
consume the whole allowance.

#### Scenario: Two eligible sightings seconds apart
- **WHEN** an announcement has just been made
- **AND** another eligible sighting occurs before the minimum interval has elapsed
- **THEN** no second announcement is made

#### Scenario: Interval elapsed
- **WHEN** the minimum interval has elapsed
- **AND** allowance remains
- **THEN** a further eligible sighting produces an announcement

### Requirement: The House Stays Quiet Outside Waking Hours
The system SHALL NOT announce outside a configured daytime window, regardless of remaining
allowance.

#### Scenario: Eligible sighting at night
- **WHEN** an eligible sighting occurs outside the permitted window
- **THEN** nothing is announced

### Requirement: Wording Matches The Outstanding Chores
The system SHALL choose wording that names the outstanding chores, and SHALL combine both into a
single announcement rather than speaking twice.

#### Scenario: Only feeding outstanding
- **WHEN** only feeding is outstanding
- **THEN** the announcement refers to feeding alone

#### Scenario: Only yard duty outstanding
- **WHEN** only yard duty is outstanding
- **THEN** the announcement refers to the yard alone

#### Scenario: Both outstanding
- **WHEN** both chores are outstanding
- **THEN** one announcement refers to both

### Requirement: Each Child Has Their Own Voice
The system SHALL speak each child's reminders in a voice assigned to that child, so a reminder is
recognisable as personal before the words are parsed.

#### Scenario: An assigned child is announced to
- **WHEN** the responsible child has an assigned voice
- **THEN** the announcement uses that voice

#### Scenario: A child with no assigned voice
- **WHEN** the responsible child has no assigned voice
- **THEN** the announcement is still made, using a default voice

### Requirement: Names Are Pronounced As Written By The Family
The system SHALL speak a child's name in its correct written form, including any accent, without
altering the identifier used to match against the camera.

#### Scenario: A name that carries an accent
- **WHEN** the responsible child's name carries an accent
- **THEN** the spoken form includes the accent
- **AND** the value used to match the recognised person remains unaccented

### Requirement: A Chosen Voice Must Be Verified To Produce Audio
The system SHALL only use a voice confirmed to produce audio on the current subscription, because
an unavailable voice is rejected without any error reaching the caller.

#### Scenario: An unusable voice is configured
- **WHEN** a voice unavailable on the current plan is used
- **THEN** the announcement is silent and the failure appears only in the log

### Requirement: The Display Follows The Voice
The system SHALL bring the relevant chore onto the display when it announces, so the spoken
reminder can be checked visually.

#### Scenario: Announcing with feeding outstanding
- **WHEN** an announcement is made and feeding is outstanding
- **THEN** the feeding entry is brought to the front of the display

#### Scenario: Announcing with only yard duty outstanding
- **WHEN** an announcement is made and only yard duty is outstanding
- **THEN** the yard entry is brought to the front of the display

### Requirement: Announcements Reach An Audible Speaker
The system SHALL announce on a speaker that produces sound unaided, because a playback target that
depends on separate powered equipment reports success while remaining silent.

#### Scenario: Announcing to the living room
- **WHEN** an announcement is made
- **THEN** it plays on a self-powered speaker in the room the camera watches
