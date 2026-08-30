# ruby-voice-reminder Specification

## Purpose

Speaks a personalised reminder when the camera recognises the child whose turn it is and a chore is
outstanding, and brings the matching chore onto the display so the spoken reminder has something to
point at. A supplement to the display, not a replacement — the display is ambient and easy to stop
seeing.


## Why It Is Built This Way

**The trigger is a sighting, not a change of identity.** The camera's recognition result is sticky:
it holds the last recognised name, potentially for hours. Reacting to that value changing means
reacting only when a *different* person appears — and the most common real case, the same child
walking past twice, produces no change at all. The automation reacts to the per-sighting timestamp
instead. This is the single most important detail here: keying off the identity looks correct,
passes a casual test with two people, and then under-fires in production in a way nobody notices.

**Two limits exist because they do different jobs.** A cap on announcements per period controls
total volume. A minimum interval controls bunching. Neither substitutes for the other: a minimum
interval alone permits dozens of announcements a day, and a cap alone is spent within seconds of a
family gathering, because the recognition result flips between names several times a second when
more than one person is in frame.

**The allowance is not refilled on start-up, deliberately.** The two failure modes are not
symmetric. A missed refill means fewer announcements, which is harmless. A refill on every restart
means more announcements than intended, which is the exact failure this feature exists to avoid.
Restarts are common, so the trigger is omitted.

**Everything is a gate; nothing is a fallback.** Uncertainty produces silence. There is no default
path and no "announce anyway" branch.

**Two silent-failure traps, both of which look like success from the inside.** They are the reason
this capability insists on a verified, self-powered speaker:

- A playback target that is only a streaming endpoint will accept audio and report itself as playing
  for the full duration, even when the equipment it feeds is switched off. Every observable signal
  short of a person in the room says the announcement worked.
- A speech voice outside the current subscription tier is rejected *after* the request has already
  returned success, because generation is streamed. The rejection reaches the log and nothing else.

Both were found only by reading logs after a human said they had heard nothing. Neither can be
caught by asserting that playback occurred — which is why no test here should do so.

**The spoken name is not the identifier.** One child's name carries an accent. The value used to
match against the camera must stay unaccented, because that is exactly the string the camera
produces; accenting it would silently stop her ever matching. The accent is applied only at the
point of speech. Generally: an identifier shared with another system should never carry
presentational changes.

**Recognition frequency is very uneven between family members**, by more than an order of magnitude,
depending on where each person tends to be. The spoken layer will fire noticeably less during the
fortnight of a child the camera sees rarely. That is a camera-placement problem, not something the
automation can correct, and it is the reason this remains a supplement to the display rather than a
replacement.

**Practical notes for anyone debugging this.** The display changes within milliseconds of a sighting
and speech follows about four seconds later, that gap being synthesis plus the speaker waking; a
person perceiving more delay than that is seeing the camera's own recognition pipeline, which is
upstream. Error counts in logs overstate failures badly, because one rejected request retries
several times. And a busy room can generate enough rejected runs to push the interesting one out of
the stored execution history within seconds, so prefer the counter, the interval timer and the
last-run timestamp over hunting for a trace.

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
