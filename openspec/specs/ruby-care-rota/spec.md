# ruby-care-rota Specification

## Purpose

Assigns responsibility for the dog's chores to exactly one child at a time, advancing on a
predictable schedule without anyone having to remember to change it. Every other Ruby capability
reads the current turn to decide who to name.


## Why It Is Built This Way

**Rotation is driven by the calendar, not by chores being done.** An earlier version advanced the
turn each time a feeding was recorded. That is wrong for a subtle reason: the feeding status
reports the dog as fed *optimistically*, before a feeding window opens, so a status of "fed" is not
evidence that anybody fed her. Anything keyed off that transition rotates on days nobody did
anything.

**The block marker is what makes rotation safe to evaluate often.** Rotation is checked daily and
again whenever the system starts. Without recording which block has already been rotated for, every
restart inside a block would advance the turn and quietly rob a child of their fortnight. With the
marker, evaluation is idempotent and can run as often as convenient.

**An empty marker means bootstrap, not "rotate now".** On a fresh install the marker is recorded and
the turn is left alone, so standing the system up does not jerk the rota forward.

**One child owns both chores.** An earlier version offset yard duty by one, so the two chores never
landed on the same person. That was the right shape when the turn moved several times a day; with a
half-month block it splits ownership of a single fortnight between two children, which is harder to
reason about than it is fair. Yard duty is now derived as the identity of the care turn, kept as a
named derivation rather than a direct read so the offset can be restored in one place.

## Requirements

### Requirement: Half-Month Rotation
The system SHALL advance the care turn to the next child on the roster at the start of each
half-month block, where the first block covers days 1 to 15 and the second covers day 16 to the
end of the month.

#### Scenario: A new block begins
- **WHEN** the current half-month block differs from the block already rotated for
- **THEN** the care turn advances to the next child on the roster
- **AND** the new block is recorded as rotated for

#### Scenario: Rotation re-evaluated within the same block
- **WHEN** rotation is evaluated again during a block already rotated for
- **THEN** the care turn is unchanged

#### Scenario: First ever evaluation
- **WHEN** no block has ever been recorded as rotated for
- **THEN** the current block is recorded
- **AND** the care turn is left unchanged

### Requirement: Rotation Is Not Advanced By Restarts
The system SHALL NOT advance the care turn as a side effect of a restart, reload, or repeated
evaluation, so that no child loses a turn to an operational event.

#### Scenario: Restart inside an already-rotated block
- **WHEN** the system restarts during a block that has already been rotated for
- **THEN** the care turn is unchanged

### Requirement: Chores Are Not Advanced By Being Performed
The system SHALL NOT advance the care turn when a chore is completed, so that a child holds the
turn for the whole block regardless of how often they act.

#### Scenario: Chore completed mid-block
- **WHEN** a chore is recorded as done
- **THEN** the care turn is unchanged

### Requirement: One Child Owns Both Chores
The system SHALL derive the yard-duty holder from the care turn so that a single child owns both
chores for the block.

#### Scenario: Deriving yard duty
- **WHEN** the care turn is a given child
- **THEN** the yard-duty holder is that same child

#### Scenario: Roster membership
- **WHEN** the yard-duty holder is derived
- **THEN** the result is always a member of the care-turn roster

### Requirement: Turn Is Manually Correctable
The system SHALL allow the care turn to be set by hand, and SHALL reflect a manual correction
immediately in every capability that names the responsible child.

#### Scenario: Turn corrected by hand
- **WHEN** the care turn is changed manually mid-block
- **THEN** subsequent reminders name the newly selected child
- **AND** the recorded block marker is unchanged, so the next boundary still advances normally
