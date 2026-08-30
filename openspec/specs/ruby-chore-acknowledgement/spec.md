# ruby-chore-acknowledgement Specification

## Purpose

Records that a chore has been done. Feeding is recorded automatically when the food container is
opened; either chore can also be acknowledged by hand from the display, which is the fallback for
when the container sensor does not fire.


## Why It Is Built This Way

**The acknowledge button is guarded by what is on screen, not by a mode.** Overloading a shared
physical control is only safe if the guard is something already visible to the person pressing it.
The displayed chore is exactly that: you press the button while looking at the thing you are
acknowledging. A mode flag someone has to set first would be worse than no shortcut at all.

**There is no default branch.** An unrecognised app is a no-op rather than falling through to
acknowledge whichever chore was checked last. Getting this wrong would silently mark chores done at
random.

**The manual path exists because the automatic one is unreliable.** The food container sensor does
not always fire, and a missed feeding record produces a false reminder — which erodes trust in every
other reminder the system makes. Both paths write the same timestamp so nothing downstream can tell
them apart; this is a fallback, not a parallel system.

**One automation covers both chores.** "Acknowledge whatever is on screen" is a single idea, so a
third chore is a new branch rather than a new automation.

**A caution learned the hard way.** Because the display can be driven to a chore programmatically,
any automated test that does so leaves that chore one button press from being marked done. This has
already happened once: a chore that had not been done was recorded as complete during an unrelated
test. Anything that forces the display must restore chore timestamps afterwards, not just its own
state.

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
