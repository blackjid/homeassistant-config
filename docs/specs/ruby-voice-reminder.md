# Ruby Voice Reminder

Status: implemented (2026-08-29), spec written retrospectively.

Extends the [Ruby Care System](./ruby-care-system.md). That spec covers the chores themselves —
feeding status, the half-month care rota, poop duty, and the Pixel Display. This one covers the
spoken layer on top: catching the kid whose turn it is, in person, and telling them.

## Problem Statement

The Pixel Display works, and that is exactly its limitation. It is ambient. A reminder sits in the
app rotation being quietly correct while everyone walks past it, because a small display in the
corner of a room is easy to stop seeing after the first week.

The chore also has a specific owner at any moment — the kid holding the current half-month care
turn — and a display cannot tell whether that person is the one in the room. It shows the same
thing to whoever happens to look, which means the reminder is either aimed at nobody in
particular or aimed at someone who cannot act on it.

There is already a camera in the living room that recognises faces. The house knows who just
walked in. Nothing was using that to close the gap between "a chore is outstanding" and "the
person responsible for it is standing right here".

## Solution

When the living room camera recognises the kid whose care turn it currently is, and a chore is
genuinely outstanding, the house says so out loud — naming them, naming the chore — and pushes
the matching app onto the Pixel Display so the spoken reminder has something to point at.

It is deliberately hard to make it talk. It speaks at most twice per feeding period, never less
than half an hour apart, never outside waking hours, and never when there is nothing to do. An
announcement that fires whenever it technically could is one the family learns to tune out, which
would leave the system worse off than the silent display it was meant to reinforce.

## User Stories

1. As a parent, I want the house to speak up when a chore is outstanding, so that the reminder
   does not depend on someone happening to glance at a small display.
2. As a parent, I want the reminder aimed at the kid whose turn it actually is, so that it lands
   on someone who can act on it rather than on whoever is nearest.
3. As a kid, I want to be addressed by name, so that it is unambiguous the reminder is for me.
4. As a kid, I want to be told which chore is outstanding, so that I do not have to go and check
   the display to find out what is being asked.
5. As a kid, I want to hear about both chores in one sentence when both are outstanding, so that
   I am not interrupted twice in a row.
6. As a family member, I want the display to jump to the relevant app when the announcement
   plays, so that the spoken reminder and the visual one agree with each other.
7. As a family member, I want nothing said when both chores are already done, so that the system
   never nags about finished work.
8. As a family member, I want nothing said when the person recognised is not the one on duty, so
   that siblings are not blamed for each other's chores.
9. As a family member, I want at most two announcements per feeding period, so that the house
   does not become something to tune out.
10. As a family member, I want a minimum gap between announcements, so that a single busy moment
    in the living room cannot spend the whole day's allowance at once.
11. As a family member, I want the allowance refilled at the start of each feeding period, so
    that a morning that has been used up does not silence the evening as well.
12. As a family member, I want no announcements late at night or early in the morning, so that
    the house stays quiet when people are asleep.
13. As a parent, I want a restart not to refill the announcement allowance, so that a reboot
    cannot become a way to get extra reminders.
14. As a kid, I want a reminder to arrive as I walk in, so that it reaches me while I am in a
    position to act rather than after I have settled somewhere else.
15. As a family member, I want the same person walking past repeatedly to be recognised each
    time, so that the reminder is not lost because they were the last face seen.
16. As a family member, I want the announcement to play on a speaker that is actually audible, so
    that a reminder is never silently swallowed by hardware that is switched off.
17. As a parent, I want the spoken reminder to use the same chore state as the display, so that
    the two can never disagree about whether something is outstanding.
18. As a parent, I want the announcement wording to name the person from the live rota, so that a
    mid-period correction to the turn is reflected immediately.
19. As an operator, I want the announcement allowance visible as a helper, so that I can see how
    much of it has been used without reading logs.
20. As an operator, I want the cooldown visible as a helper, so that I can tell at a glance
    whether the system is currently muted and for how long.
21. As an operator, I want to be able to clear the allowance and the cooldown by hand, so that I
    can test the feature without waiting out a period.
22. As an operator, I want the announcement limits adjustable in the UI, so that the family can
    tune how talkative the house is without a config edit.
23. As an operator, I want the whole feature editable in the automation editor, so that the
    wording and the thresholds can be changed in place.
24. As an operator, I want to know that the face vocabulary and the rota vocabulary agree, so
    that a name mismatch cannot silently prevent the feature from ever firing.
25. As an operator, I want the feature to fail closed, so that any uncertainty results in silence
    rather than an unwanted announcement.
26. As a kid, I want my own voice on my reminders, so that I recognise one is aimed at me before I
    have parsed the words.
27. As a kid, I want my name pronounced correctly, so that the reminder does not sound like it was
    written by a machine that does not know me.
28. As a family member, I want the reminder wording to describe the chore accurately, so that it
    refers to the backyard rather than implying the dog owns it.
29. As a family member, I want a clear pause after my name, so that the first word is not lost
    before I have started listening.
30. As an operator, I want an unassigned or renamed kid to still get a spoken reminder in a
    default voice, so that editing the rota cannot silently mute someone.
31. As an operator, I want to know that a chosen voice actually produces audio, so that a
    subscription limit cannot mute the feature without any visible error.

## Implementation Decisions

### Trigger: the attribute, not the state

The living room face sensor is **sticky**. Its template preserves the previous value whenever an
incoming event is not a living-room person detection, so the state holds the last recognised name
indefinitely — potentially for hours. A state trigger therefore fires only when a *different*
person is recognised, and the most common real case, the same kid walking past twice, produces no
state change at all.

The sensor does update a `last_seen` attribute on every recognition. The automation triggers on
that attribute, which fires once per detection regardless of who was seen.

This is the single most important detail in the feature. A state trigger looks correct, passes a
casual test with two different people, and then under-fires in production in a way that is very
hard to notice.

### Two limits, doing different jobs

Both are required, and neither substitutes for the other.

- **A counter caps volume.** At most two announcements per feeding period. Refilled at the start
  of each period by a separate scheduled automation.
- **A timer enforces spacing.** A cooldown, started on every announcement, that must be idle
  before the next one.

Without the counter, a rolling cooldown alone permits roughly two dozen announcements a day —
technically rate-limited, in practice relentless.

Without the timer, both of the period's announcements fire within seconds of each other. The face
sensor flaps between names every few hundred milliseconds whenever several people are in frame,
so a single family gathering in the living room would spend the entire allowance before anyone
had a chance to act on the first reminder.

### The quota reset does not run on start-up

The reset automation is driven purely by the two period-boundary times. It deliberately has no
Home Assistant start trigger.

The two failure modes are not symmetric. A missed reset means fewer announcements than intended,
which is harmless. A reset on every restart means more than intended, which is the exact failure
this feature is built to avoid. Given restarts are common, the trigger is omitted on purpose.

### Fail closed

Every gate is a condition, not a branch. There is no default path, no fallback announcement, and
no "if in doubt, speak". Uncertainty — an unrecognised name, an unavailable sensor, an exhausted
allowance, an hour outside the window — produces silence.

### Conditions are native wherever they can be

The allowance check is a numeric-state condition, the cooldown a state condition, the quiet hours
a time condition, and the "is anything actually outstanding" test a nested or/not over state
conditions.

Only the face-to-rota comparison is a template, because it compares two entities against each
other and Home Assistant has no native construct for that.

### Message selection

Three sentences: both chores outstanding, feeding only, yard only. Chosen from the same status
sensors the display reads, so the spoken and visual reminders cannot disagree.

The name comes from the live rota rather than from the recognised face. These are equal at the
moment the automation runs — that equality is the trigger condition — but reading the rota means
a hand-corrected turn is reflected immediately, and it keeps a single source of truth for who
owns the chore.

### Display follows the voice

The announcement also forces the matching app onto the Pixel Display: the food app if feeding is
outstanding, otherwise the yard app. Feeding takes precedence when both are outstanding, on the
grounds that the dog's welfare outranks the lawn.

This makes the announcement pointable. Someone who half-hears it can look at the display and see
what was said, and the display is already where the acknowledgement button lives.

### Speaker selection is a correctness decision, not a preference

The announcement targets a **self-powered** speaker.

The living room's other media player is a Chromecast Audio: a dongle feeding an AV receiver. It
accepts a stream and reports `playing` for the full duration of the audio whether or not the
receiver downstream is powered on. When the receiver is off — its normal state — the result is a
completely silent failure. No error is logged, the service call succeeds, and the media player
state transitions look exactly like a working announcement.

This was not theoretical. It is how the feature first failed, and it cost a full diagnostic pass
to find, because every observable signal short of a human in the room said the reminder had
played.

Any future change of speaker must confirm the target is self-powered, or that whatever it feeds is
guaranteed to be on.

### Name vocabularies must be checked, not assumed

The face recogniser and the care rota are independent systems that happen to use people's names.
The feature only works where the two vocabularies agree exactly.

They were verified to agree before the feature was built. The recogniser also emits names that are
not rota members, which is harmless — those simply never match.

Worth recording: detection frequency is very uneven between family members, by more than an order
of magnitude. The feature will fire noticeably less often during the fortnight of a kid the camera
sees rarely. This is a property of the camera's position and each person's habits, not something
the automation can correct, and it makes the spoken layer a *supplement* to the display rather
than a replacement for it.

### Speech provider, and the voice tier that silently fails

Announcements go through a commercial neural speech engine rather than the free translate-based
one. The wording is short and fixed, so quality matters more than the per-character cost.

**The provider's voices are tiered, and the lower tier fails silently.** Voices marked *premade*
work on a free plan. Voices marked *professional*, and anything added from the provider's public
voice library, are rejected on a free plan with an HTTP 402 — and that rejection **does not reach
the caller**. The speech service call returns success, because generation is streamed after the
call returns; only the log records the failure. From every observable signal inside Home
Assistant, a 402 and a working announcement look identical.

This is the second silent-failure trap in this feature, structurally identical to the powered-off
amplifier: an outward-facing failure that looks like success from the inside. Both were found only
by reading the log after a human reported hearing nothing.

Consequences that must survive future edits:

- A voice may only be assigned after confirming it actually produces audio. Appearing in the
  integration's dropdown is not sufficient — the dropdown lists everything in the account,
  including voices the plan cannot use.
- Adding a library voice to the account does not make it usable on a free plan. That is precisely
  what the 402 rejects.
- At least one voice name exists in both tiers, distinguishable only by its description suffix.
  Picking by name alone can silently select the unusable one.

### One voice per kid

Each kid on the rota gets their own voice, chosen from the usable tier, set as a per-call override
so the household default is untouched. A name not in the map falls back to that default rather
than erroring, so an unmapped or renamed rota entry still speaks.

The map lives in a variable at the top of the automation rather than inline in the speech action,
and it re-reads the rota itself instead of depending on variable evaluation order.

One rule when assigning: do not give a kid the same voice as the fallback. If the fallback and an
assigned voice are identical, a name that has silently dropped out of the map is indistinguishable
from one that is working.

### The spoken name is not the rota value

One kid's name carries an accent. The accented form is what should be spoken, but the rota value
must stay unaccented, because that is the exact string the face recogniser emits and the trigger
compares the two directly. Accenting the helper would silently stop her ever matching.

So a second map translates the rota value into a spoken form, applied only at the point of speech.
Everything else — matching, the display, the chore logic — continues to use the plain value.

This generalises: the rota value is an identifier shared with another system, and anything that is
purely presentational belongs in a translation layer rather than in the identifier.

### Only the voice is adjustable per call

The integration accepts a voice override per call and nothing else. Speaking rate in particular is
rejected before the request leaves Home Assistant. Voice settings beyond the voice itself exist
only as integration-wide options, so they cannot be varied per kid or per announcement.

## Testing Decisions

### What a good test looks like here

A good test drives a face recognition and asserts on what the house was told to do — the sentence,
the speaker, the app. It never asserts on which condition rejected a run, how the counter is
stored, or that a particular automation fired. Those are free to change.

The interesting behaviour is almost entirely **gating and wording**, so both must be observable.

### The seam

**One seam: Home Assistant's `call_service` event bus.**

- **Drive** — publish synthetic camera events to make the face sensor report a chosen name, plus
  service calls against the rota, chore and limit helpers to set up each scenario.
- **Assert** — the `call_service` events the automation emits: the text-to-speech call with its
  message and target speaker, the display app-switch publish, the counter increment and the timer
  start.

This is the highest seam that can see the spoken sentence. The Ruby Care System spec's MQTT seam
is the right one for the display, but it is blind here: the message text never reaches the broker.

Driving the face sensor with synthetic camera events is proven — it is how the feature was
verified, including a negative case with a non-matching name.

**Accepted limitation:** this observes what the system *intended*, not what the device received.
That is acceptable because the only outbound message here is an app switch of an already-proven
shape. It would not be acceptable for the display spec, where malformed payloads are rejected
wholesale by the firmware.

**Explicitly rejected: asserting on media player state.** A test that waits for `playing` would
have passed throughout the period this feature was completely silent. The state transition is
evidence that audio was dispatched, not that anyone could hear it. It is worse than no assertion,
because it manufactures false confidence.

### Scenarios

Matching: a recognised name equal to the current turn, with a chore outstanding, produces exactly
one announcement naming that person, the correct sentence for which chores are outstanding, the
correct speaker, and a matching app switch.

Non-matching: a recognised name that is not the current turn produces nothing. A recognised name
that is not in the rota at all produces nothing.

Nothing to do: a match while both chores are already done produces nothing.

Wording: feeding only, yard only, and both outstanding each produce their own sentence, and the
combined case is one announcement rather than two. Each names the kid, and the accented name is
used where one applies.

Voice selection: each kid on the rota resolves to their own assigned voice; a rota value with no
assignment resolves to the fallback rather than failing; and the resolved voice reaches the speech
call as an override rather than being left to the integration default.

Quota: two announcements are permitted per period and the third is refused; a period boundary
refills the allowance; a restart does not.

Cooldown: a second match immediately after an announcement is refused while the cooldown is
running, and permitted once it has elapsed.

Quiet hours: a match outside the permitted window produces nothing, regardless of remaining
allowance.

Repeat detection: the same person recognised twice in a row is detected both times — the
regression test for triggering on the sensor's state instead of its attribute.

### Prior art

The repository has no automated tests. This is the second seam proposed across the two Ruby specs;
they are separate because the features have different risk profiles and different observable
surfaces, and neither seam can see what the other is for.

The manual sequence used during implementation is the working template: set the helpers, publish a
synthetic camera event, read back the emitted calls and the device state, assert, restore.

### The same caution as the care system spec

Tests here mutate real household state and make real noise. Any harness must capture and restore
the helpers it touches, and should target a speaker that will not wake anyone.

There is a sharper version of this hazard specific to this feature: forcing a chore app onto the
display puts that chore one button press from being marked done. During implementation a test
announcement did exactly that — the display switched to the yard app, someone pressed the
acknowledge button while it was showing, and a chore that had not been done was recorded as
complete. A harness must restore chore timestamps, not just the limit helpers.

## Out of Scope

- **Other cameras.** Only the living room drives this. Extending to the entrance or other rooms is
  a separate decision about how much of the house should talk.
- **Announcing to people who are not on duty.** Deliberate. A reminder aimed at the wrong sibling
  is worse than no reminder.
- **Escalation.** The overdue state changes the wording of the display but not the behaviour of
  the announcement. Speaking more insistently as a chore ages is a plausible next step and is not
  implemented.
- **Acknowledging by voice.** The chore is cleared by the display's button or the container
  sensor. Voice acknowledgement would need a wake word and a very different set of trade-offs.
- **Presence-based suppression.** No check for whether anyone else is in the room, asleep, or on a
  call. The quiet-hours window is the only concession to timing.
- **Volume management.** The announcement plays at whatever the speaker is currently set to, and
  restores nothing. Setting a floor for reminders is a reasonable improvement and is not done.
- **Language.** English, chosen deliberately and confirmed after the engine change made other
  languages available. The household is Spanish-speaking, so this is a preference rather than a
  constraint; the three sentences are the only thing that would need rewriting.
- **Speaking rate.** Not adjustable per announcement — the integration rejects it outright. A
  global voice-settings step may expose it, at the cost of applying to every use of the engine.
- **Fixing uneven face detection.** A camera-placement and training problem, not an automation one.

## Further Notes

**Speech is no longer free.** The earlier translate-based engine cached to disk, so a small set of
repeated sentences cost nothing after the first play. The current engine bills per character on
every call and does not cache the same way. At the capped rate of four announcements a day this is
negligible, but the economics now scale with how often the feature speaks — which is one more
reason the quota exists.

**The fastest model is the right one here.** The engine offers several models trading quality
against latency. The sentences are one line long and the audience is a child walking through a
room, so the fastest model is preferred; the quality difference is inaudible at this length and
the latency difference is not.

**End-to-end latency is dominated by the camera, not Home Assistant.** Measured during
implementation: three milliseconds from the face sensor updating to the display switching, and
roughly four seconds more before audio started, consistently, that being speech synthesis plus the
speaker's own startup. The display therefore always changes first and the voice follows. Any
further delay a person perceives is the camera's own detect-and-recognise pipeline, which is
upstream and not tunable from Home Assistant.

**Error counts in the log are retries, not distinct failures.** One rejected speech request
produced eight log lines through the streaming retry path. Counting log lines badly overstates how
many announcements actually failed; count distinct requests instead.

**The chosen speaker drops off the network briefly most nights** in the small hours, recovering
within about half a minute. Harmless while announcements are confined to waking hours; it would
matter if that window were ever widened.

**The trace buffer is not a reliable record here.** A single family gathering can generate enough
rejected runs to evict the successful one from the stored traces within seconds. Debugging should
lean on the counter, the cooldown and the automation's last-triggered timestamp rather than
expecting to find the interesting trace still present.
