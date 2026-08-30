# ruby-chore-acknowledgement Specification

## Purpose

Records that a chore has been done. Feeding is recorded automatically when the food container is
opened; either chore can also be acknowledged by hand from the display, which is the fallback for
when the container sensor does not fire.

Design rationale, including why the button is guarded by the displayed app rather than a mode
flag, lives in `docs/specs/ruby-care-system.md`.

## Requirements

### Requirement: Feeding Is Recorded From The Container
The system SHALL record a feeding when the food container is opened, so that no one has to confirm
the chore separately.

#### Scenario: Container opened
- **WHEN** the food container lid opens
- **THEN** the feeding timestamp is set to the current time

### Requirement: A Chore Can Be Acknowledged From The Display
The system SHALL acknowledge the chore whose app is currently on screen when the display's
acknowledge button is pressed.

#### Scenario: Acknowledging feeding
- **WHEN** the feeding app is the app on screen
- **AND** the acknowledge button is pressed
- **THEN** the feeding timestamp is set to the current time

#### Scenario: Acknowledging yard duty
- **WHEN** the yard app is the app on screen
- **AND** the acknowledge button is pressed
- **THEN** the pickup timestamp is set to the current time

#### Scenario: Button pressed on any other app
- **WHEN** neither chore app is the app on screen
- **AND** the acknowledge button is pressed
- **THEN** no chore is recorded
- **AND** the button retains its ordinary behaviour

### Requirement: The Fallback Is Indistinguishable Downstream
The system SHALL record a manually acknowledged feeding in the same place as an automatically
recorded one, so that nothing downstream can tell the two apart.

#### Scenario: Feeding recorded by button
- **WHEN** feeding is acknowledged from the display
- **THEN** the feeding status, elapsed time, and every reminder behave exactly as if the container
  had been opened

### Requirement: Chores Are Acknowledged Independently
The system SHALL leave the other chore untouched when one chore is acknowledged.

#### Scenario: Acknowledging one of two outstanding chores
- **WHEN** both chores are outstanding
- **AND** one is acknowledged
- **THEN** the other chore's status, app, and indicator are unchanged

### Requirement: Repeated Acknowledgement Does Not Distort Records
The system SHALL tolerate the same chore being acknowledged more than once in a period without
corrupting the rota or double-counting the chore.

#### Scenario: Container opened twice in one period
- **WHEN** the food container is opened twice within the same feeding period
- **THEN** the feeding is recorded
- **AND** the care turn is unchanged
