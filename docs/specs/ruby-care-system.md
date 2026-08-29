# Ruby Care System

Status: implemented (2026-08-29), spec written retrospectively as the reference for future work.

## Problem Statement

Ruby is the family dog. Three kids share her care, and two chores recur: feeding her twice a
day, and picking up the yard.

Four things were wrong.

**Nobody knew whose turn it was.** `input_select.ruby_care_turn` was a dropdown a human had
to change by hand, so it never changed. Every reminder named the same kid indefinitely, which
made the reminders both unfair and easy to ignore.

**There was no poop reminder at all.** The chore existed socially but had no representation in
the house, so it happened when someone noticed, which is to say rarely and always the same
person.

**Reminders were invisible unless you went looking.** The feeding status existed as a sensor
and a dashboard tile. Nobody opens a dashboard to find out whether the dog has been fed.

**The display integration broke.** The kitchen's AWTRIX 3 clock was upgraded to AWTRIX NG
firmware, which changed every MQTT topic and every payload key. Each automation that drove the
display failed silently, and the custom integration the automations depended on was removed.

## Solution

A care system that decides three things on its own — is Ruby hungry, is the yard dirty, and
whose turn is it — and puts the answers on the Pixel Display in the living room where the
family already walks past.

The care turn advances by itself on a half-month rota, and whoever holds it owns Ruby for that
fortnight — both feeding and the yard. Anything needing attention occupies a slot in the
display's app loop until it is dealt with; anything already done says thank you once and
disappears. Either chore is cleared by pressing the centre button on the display while that
chore's app is on screen, so the kid who does the work closes the loop themselves without
touching a phone.

## User Stories

1. As a parent, I want the care turn to advance on its own, so that I am not the person who has
   to remember to change a dropdown twice a month.
2. As a parent, I want each kid to hold the turn for a predictable half-month block, so that the
   rota is obvious enough to argue with and hard to argue about.
3. As a kid, I want the display to name whose turn it is, so that I know without asking whether
   it is me.
4. As a kid, I want one person to own both chores for the whole fortnight, so that there is a
   single clear answer to "whose dog is it this week".
5. As a parent, I want the poop turn to follow the feeding rota automatically, so that there is
   no second rota to maintain or keep in step.
6. As a parent, I want the turn to survive a Home Assistant restart without skipping a kid, so
   that a reboot does not silently cost someone their turn.
7. As a parent, I want the turn to survive a config reload or several triggers in the same
   period without double-advancing, so that the rota stays honest.
8. As a family member, I want a glanceable reminder when Ruby has not been fed, so that I do not
   have to open an app to find out.
9. As a family member, I want the reminder to stay on screen until she is fed, so that it cannot
   be missed by looking away.
10. As a family member, I want the reminder to disappear the moment she is fed, so that the
    display never nags about work already done.
11. As the kid who fed her, I want a short thank-you naming me, so that the chore feels
    acknowledged.
12. As a family member, I want the reminder to grow more urgent the longer she goes unfed, so
    that "due soon" and "far too long" look different at a glance.
13. As a family member, I want feeding to be recorded by opening the food container, so that no
    one has to confirm the chore in an app afterwards.
14. As a family member, I want opening the container twice not to count as two feedings, so that
    the record stays accurate.
15. As a family member, I want a reminder to pick up the yard in the late afternoon, so that it
    gets done in daylight rather than remembered at bedtime.
16. As a family member, I want the yard reminder to escalate if it has been skipped for two
    days, so that a missed day does not quietly become a missed week.
17. As the kid on duty, I want to clear either chore by pressing the centre button on the
    display, so that finishing the work takes one press on the way past.
18. As a family member, I want that button to keep working normally the rest of the time, so
    that clearing a chore never collides with paging through apps.
19. As a family member, I want to be able to mark Ruby fed from the display, so that a feeding
    still gets recorded on the days the food container sensor does not fire.
20. As the kid who did it, I want a thank-you naming me, so that the chore is credited to the
    right person.
21. As a family member, I want the food and yard reminders to be visually distinct, so that I can
    tell them apart from across the room.
22. As a family member, I want the display to return to its normal clock when nothing needs
    doing, so that the house is not permanently shouting at us.
23. As a household, we want the display to be corrected after a Home Assistant restart, so that a
    stale reminder cannot outlive the condition that caused it.
24. As a household, we want the display to be corrected after the device itself reboots, so that
    the reminder returns rather than vanishing with the device's memory.
25. As a household, we want reminders delivered over a transport that does not depend on Home
    Assistant reaching the device directly, so that a network-layer problem cannot silence the
    system.
26. As an operator, I want display updates to be sent only when the underlying status actually
    changes, so that a low-powered device is not re-rendering on every attribute tick.
27. As an operator, I want transient messages sent as notifications rather than persistent apps,
    so that the app loop stays short and the device stays responsive.
28. As an operator, I want icons stored on the device as files, so that payloads carry a short
    identifier instead of an inline image on every push.
29. As an operator, I want the automations editable in the Home Assistant UI, so that I can
    review and adjust them without a text editor and a deploy.
30. As an operator, I want the derived sensors editable in the UI, so that thresholds and wording
    can be tuned in place.
31. As an operator, I want the dashboard tile to keep showing how long ago Ruby was fed, so that
    the existing view does not regress.
32. As an operator, I want the system to behave sanely before any chore has ever been recorded,
    so that a fresh install does not open by declaring everything overdue.
33. As an operator, I want a documented way to verify the whole system, so that a firmware or
    Home Assistant upgrade can be checked rather than hoped about.

## Implementation Decisions

### Layering

The system is three layers, and the split is deliberate:

- **State** — helpers holding facts: when Ruby was last fed, when the yard was last cleared,
  whose turn it is, and which care period has already been rotated for.
- **Derivation** — template sensors turning those facts plus the current time into a sentence a
  human can read. These have no side effects.
- **Behaviour** — automations, split again into *mutation* (record a chore, advance the rota)
  and *presentation* (mirror a status onto the display).

Presentation automations read status sensors and never compute status themselves. Mutation
automations write helpers and never talk to the display. This is what lets the display layer be
replaced without touching the rota, and vice versa.

### Care period and rotation

The care turn advances on half-month boundaries: days 1–15 are one kid, day 16 to month end the
next. With three kids the pairing walks forward through the months rather than repeating, so
nobody is permanently assigned to the same half of the month.

`sensor.ruby_care_period` renders the current block as a sortable, comparable token
(`YYYY-MM-H1` / `YYYY-MM-H2`). Rotation is driven by comparing that token against an
`input_text` marker recording the period already rotated for.

This marker is the whole idempotency mechanism, and it exists because the rotation automation
fires on a daily time trigger **and** on Home Assistant start. Without it, every restart inside a
period would advance the rota and quietly rob a kid of their turn. With it, the automation is
safe to fire arbitrarily often.

An empty marker means "never rotated" and is treated as a bootstrap: the marker is seeded to the
current period and the turn is left alone. A fresh install therefore does not jerk the rota
forward on first run.

The rotation was previously tied to feedings — advancing once per feeding period. That is
superseded. Feeding no longer influences whose turn it is.

### Poop turn is derived, not stored

`sensor.ruby_poop_turn` is derived from the care turn rather than being a second stored helper.
There is one rota, and poop duty is a function of it.

A stored second turn would need its own rotation, its own idempotency marker, and would drift out
of alignment the first time either side was corrected by hand.

**The current derivation is the identity: poop turn equals care turn.** Whoever holds the
fortnight owns both chores. An earlier revision offset it by one — the next kid in the list — so
the two chores never landed on the same person. That was the right shape when rotation happened
per feeding and the turn moved several times a day; with a half-month block it fragments
ownership of a single fortnight across two kids, so it was reverted.

The sensor is kept as a named indirection even though it is currently a pass-through. It is the
single place to change if the offset is ever wanted back, and nothing downstream needs touching
when it is: both the poop status wording and the thank-you read the sensor, never the
`input_select` directly.

### Feeding status

Retained from the existing implementation: morning and evening periods, each with a preferred
window, plus a maximum interval after which Ruby counts as starving regardless of window.

Two corrections were made. The status template raised when the last-fed helper was empty, because
a null was compared against a number — a fresh install would have produced an unavailable sensor
rather than a sensible first reading. And the status is reported optimistically before a window
opens, which is intentional but means the status changing to "fed" is **not** proof that anyone
fed her; it is not a valid rotation trigger. This is why rotation is time-driven rather than
status-driven.

### Poop status

Days elapsed since the last recorded pickup drives three states: clean (same day), a reminder
during the late-afternoon window, and an overdue escalation at two days or more. The escalation
deliberately ignores the time window — once it is genuinely overdue, the time of day stops
mattering.

The late-afternoon window is a `tod` helper rather than an hour comparison inside the template,
so the boundary is a first-class entity that can be seen, tested and adjusted independently.

### Display protocol

AWTRIX NG replaced AWTRIX 3's topics and payload vocabulary wholesale. The contract now in use:

- Commands publish under a device-prefixed `cmd/` tree; app pushes are addressed by app name.
- Payload keys are camelCase and durations are milliseconds with an `Ms` suffix.
- An empty payload deletes a pushed app or clears an indicator.
- The firmware rejects an entire payload with `422` naming the offending key if it contains any
  key it does not recognise. There is no partial application, so payloads must be exactly right
  rather than approximately right.
- Colour-by-palette replaced the old boolean rainbow flag.
- Icons are files in the device's icon store, referenced by bare name.

### Notification versus pushed app

The single most important display decision, and the one to preserve under future change:

- A condition **needing attention** is a pushed app. It occupies a slot in the loop and stays
  until the condition clears.
- A condition **already resolved** is a notification. It interrupts once, says thank you, and
  auto-expires leaving nothing behind.

The device is an ESP32. Every pushed app is a slot the firmware re-renders on every loop
traversal, so persistent apps are spent only where persistence is the point. Announcements,
acknowledgements and celebrations are notifications.

### Transport: MQTT, not the HTTP integration

The custom AWTRIX NG Home Assistant integration is HTTP-only, and Home Assistant cannot reliably
reach the device over HTTP in this deployment. The Home Assistant pod carries a macvlan interface
on the device's VLAN, but source-based routing puts the subnet route in a separate table matched
only on the macvlan source address. Pod-initiated connections consult the main table, find no
route, exit via the cluster network and time out. Pods *without* the macvlan attachment reach the
device normally.

AWTRIX 3 was MQTT-driven, so this gap existed all along and was never exercised until an
HTTP-only integration was introduced.

The system therefore drives the display over MQTT exclusively. The device holds a persistent
outbound connection to the broker, so the routing gap is never touched. The HTTP integration was
removed as redundant — the firmware's own Home Assistant MQTT discovery already publishes the
full entity set, including the buttons and indicators this system depends on.

Reinstating the integration is not required by anything here, and would reintroduce the
dependency on a network path that does not currently work.

### Button as chore completion

Either chore is acknowledged by the display's **centre (select) button**, guarded by the device's
currently-displayed app. On the food app it stamps last-fed; on the poop app it stamps
last-pickup; on anything else it does nothing and the button keeps its normal behaviour.

This is what makes a shared physical control safe to overload: the guard is the app context, not
a mode flag someone has to remember to set. It also means the guard has to be a real condition
rather than a default branch — a `choose` with no default, so an unrecognised app is a no-op
rather than an accidental acknowledgement of whichever chore was checked last.

For feeding this is a **fallback, not the primary path**. The food container lid sensor is the
intended trigger but does not always fire, and a missed feeding record produces a false "starving"
reminder that erodes trust in the whole system. Both paths write the same helper, so a feeding
recorded either way is indistinguishable downstream.

One automation handles both chores rather than one per chore: the behaviour is "acknowledge the
chore currently on screen", which is a single idea, and keeping it in one place means a third
chore is a new branch rather than a new automation.

The button writes a timestamp helper and nothing else. The status sensor recomputes, and the
presentation automation reacts to that. The button never touches the display directly.

### Presentation automations

Both display automations follow one shape: trigger on the status sensor changing, plus on Home
Assistant start; branch on whether the status is the resolved state; send the thank-you only when
the run came from a real status change, not from a start-up resync.

Two guards matter:

- **Start-up resync.** The device keeps pushed apps in RAM. A device reboot loses them and a Home
  Assistant restart would otherwise never re-send them, so a stale reminder could outlive its
  condition or a live one could silently vanish. Resync on start fixes both directions. The
  thank-you is suppressed on this path so a restart does not congratulate anyone.
- **Attribute-churn guard.** The feeding status sensor carries a human-readable "time since"
  attribute that updates on its own cadence, producing state-change events where the status text
  is unchanged. Without a guard comparing the previous and new state values, the display is
  re-pushed several times an hour for no reason. This is a template condition because Home
  Assistant has no native "value changed, ignoring attributes" trigger.

Indicator 1 is food, indicator 2 is poop, consistently. Urgency is carried by colour and by
blink, which cost nothing extra because they ride along in payloads already being sent.

### Configuration surface

Automations and derived sensors live in Home Assistant's UI-managed storage so they can be
reviewed and adjusted in the automation editor and the helpers screen. This is a deliberate move
away from version-controlled YAML, accepting the loss of git history in exchange for being
editable in place.

One exception: the feeding status sensor stays in package YAML because it exposes a custom
attribute that a dashboard tile renders, and UI template helpers cannot define custom attributes.
Converting it would silently drop a line from an existing card.

Timestamp and marker helpers also remain in YAML. Their *values* are already editable in the UI,
so moving their definitions would be churn without benefit.

### Two Home Assistant behaviours worth recording

**A UI template helper whose template references no entities never renders.** It has nothing to
subscribe to, so it sits at unknown indefinitely — a config-entry reload does not fix it. The
care period template is pure clock arithmetic and hit this exactly. Anchoring it to a
minute-ticking time entity gives it a subscription and it renders normally. Any future
clock-only helper needs the same anchor.

**A template that reads the clock only re-renders when something it subscribes to changes.** The
poop status template compares dates, so without a time anchor a midnight rollover would go
unnoticed until some other referenced entity happened to change — delaying an overdue escalation
by up to a day. It carries the same anchor for that reason.

## Testing Decisions

### What a good test looks like here

A good test drives the system the way the house does — by changing the facts — and asserts on
what the display is told. It never asserts on how a status sentence was computed, which helper
holds the marker, or which automation fired. Those are free to change.

Concretely: set a timestamp helper to a value relative to now, or simulate a button press, then
assert on the payload published to the display's command tree.

### The seam

**One seam: the MQTT command boundary.**

- **Drive** — Home Assistant service calls against the state helpers
  (`input_datetime.set_datetime`, `input_select.select_option`, `input_text.set_value`), plus
  publishing to the device's button state topic to simulate a physical press.
- **Assert** — messages published to the display's `cmd/` tree: app pushes, app deletions,
  indicator changes, and notifications.

This is the highest available seam and it covers all three layers in one place. It is also the
real external behaviour: the family experiences payloads arriving at a display, not sensor
states. Everything between the helpers and the broker is implementation detail this seam is
deliberately blind to.

Accepted cost: a failure reports a wrong or missing payload rather than naming the sensor that
computed it. Diagnosis starts by reading the derived sensors by hand.

### Handling time

Time is not frozen. Most branches are reachable by choosing input values relative to the current
time — a last-fed timestamp far enough back produces starving; a pickup timestamp two days back
produces overdue.

The half-month boundary is the exception, and is tested by writing the period marker to a
*different* period rather than faking the date. This is legitimate because the marker is the
actual rotation input; the date only reaches the rotation through it.

### Scenarios

Feeding: fed within the current period clears the app and the indicator and sends exactly one
thank-you; unfed inside the window pushes the reminder; long-unfed pushes the escalated colour
and blinking indicator; a start-up resync reproduces the correct display without sending a
thank-you.

Rotation: an empty marker seeds without advancing; a marker naming a previous period advances
once and updates the marker; a marker naming the current period is a no-op however many times it
fires.

Poop: same-day pickup reads clean; a pickup inside the reminder window pushes the reminder; two
days or more pushes the escalation regardless of time.

Acknowledgement: a centre press while the poop app is displayed clears the poop reminder and
thanks the right kid, leaving any food reminder untouched; a centre press while the food app is
displayed clears the food reminder the same way; a centre press while any other app is displayed
changes nothing. The two chores must be independently clearable — clearing one must not disturb
the other's app or indicator.

Derivation invariants: the poop turn is always a member of the care turn's own option list, and
tracks the care turn under the current identity derivation.

### Prior art

None — the repository has no automated tests today, so this seam is new. It is proposed at the
highest point precisely to avoid seeding a test suite that pins internals.

The manual sequence exercised during implementation is the working template for the harness: set
a helper, wait for propagation, read back the device's app list and indicator state, assert,
restore. It was run against the live device for every scenario above.

Until a harness exists, that sequence stands as a documented verification procedure and should be
re-run after any firmware or Home Assistant upgrade.

### A caution for whoever builds the harness

Tests here mutate real household state. A test that marks Ruby fed removes a live reminder, and
the family loses it. Any harness must capture and restore the helper values it touches, and
should ideally run against a display that is not the kitchen one.

## Out of Scope

- **Fixing the cluster routing gap.** Diagnosed but not repaired: the fix changes a shared
  network attachment used by several applications and is not required by this system.
- **Reinstating the HTTP integration.** Removed as redundant. Only worth revisiting if the
  routing gap is fixed and its services are actually wanted.
- **Deleting the dead integration config entries.** Two inert entries remain and need removing
  through the Home Assistant UI.
- **Reconciling the repository divergence.** The live configuration directory has commits and
  uncommitted work not present in the local checkout. This predates the system and needs sorting
  out on its own terms.
- **Multi-device support.** One display, addressed by its own prefix. Generalising is speculative
  until a second exists.
- **Voice announcements.** Speakers exist throughout the house; deliberately not used, because an
  unmissable audio reminder is a different and more intrusive product decision.
- **Dashboard redesign.** The existing tile is preserved as-is and constrains one implementation
  choice; improving it is separate.
- **Per-kid notifications.** Reminders are ambient and shared by design. Targeting a phone
  changes the social contract of the chore.
- **Tracking who fed her when the turn-holder did not.** The system credits the turn-holder. A
  sibling covering for another is invisible, and that is accepted.

## Further Notes

**The poop pickup timestamp was seeded with an invented value.** There was no true starting value
for a helper that had never existed. It was set to the previous evening so the yard reads clean
and the first reminder arrives naturally, rather than opening with an overdue escalation. Adjust
if the real history differs.

**Converting YAML entities to UI helpers requires clearing the registry first.** A YAML-defined
entity leaves an orphaned registry row when removed. Creating a helper of the same name while the
orphan exists yields a suffixed entity ID and silently breaks every reference. Remove the orphans,
then create, then confirm the identifiers were reclaimed.

**Icons live on the device and did not survive the firmware upgrade.** The icon store was empty
after flashing. Icons are uploaded once and referenced by name; a future reflash will need them
uploaded again. The poop icon is hand-drawn rather than sourced from the public gallery, whose
search API no longer responds — an icon whose appearance cannot be verified before shipping is
not worth referencing by ID.

**The removed integration is parked, not deleted,** alongside backups of the previous automations,
the previous package, and the storage files, in a backup directory inside the configuration
directory. That directory is untracked and should be either ignored or cleared once the system is
settled.

**One pre-existing broken automation is unrelated** and fails on every config check: a movie-night
automation referencing a device that no longer exists. It is noise in validation output, not a
symptom of this system.
