---
name: "appointment-calendar-assistant"
description: "Adds calendar events from appointment messages, interviews, wedding cards, dinner and funeral/aza (عزاء) notices. Use whenever event details are shared to save, with routing and reminders."
compatibility: "Claude Code, Cowork, claude.ai · needs a calendar connector (list calendars, create and read events) and web search · Sonnet or Opus"
---

# Appointment & Calendar Assistant

Extracts key details from invitations and appointment messages and adds them to your calendar with the right duration and reminders — no manual entry needed.

If `personal.md` exists in this folder, read it first; it overrides the defaults above.

## When to use

Use this skill whenever an appointment message, invitation image, funeral notice or event details are shared for the calendar — including any photo of a wedding or event card. Always use it rather than calling the calendar tool directly: it applies the routing, durations, prayer-time lookups, reminder and working-hours rules.

Trigger phrases include: "add this to calendar", "book this", "save this appointment", "hospital appointment", "clinic appointment", "interview", "wedding invitation", "dinner invitation", "save the date", "توفي", "وفاة", "عزاء", "دفن", "جنازة".

## Install

- **Claude Code / Cowork:** clone or copy this folder to `~/.claude/skills/appointment-calendar-assistant/` (the folder name must match), then restart.
- **claude.ai:** zip the folder and upload it under Settings → Capabilities → Skills.
- No packages to install. It needs a connected calendar tool and web search (for prayer times on funeral notices).

---

## First-Run Setup

Before doing anything else, check whether `~/.claude/skills/appointment-calendar-assistant/config.json` exists.

**If it does not exist**, this is the first use — run this interview once:

1. Call the calendar tool's list-calendars function and show the user the calendars available to them.
2. Ask: "Do you want everything on one calendar, or do you separate your own appointments, family appointments, and social events?" Then:
   - **One calendar** — ask which, and record it as `personal`. Leave `family` and `events` unset.
   - **Separate calendars** — ask which calendar to use for each of the three roles. Any the user doesn't keep separate is left unset.
3. Ask for their timezone as a UTC offset (e.g. `+03:00`). If the calendar tool exposes a timezone for the account/calendar, propose it and just ask them to confirm.
4. Ask for the city they live in. Explain why: it's only used to look up accurate prayer times when logging a funeral/burial notice — skip this question if the user says they won't need the funeral-notice feature.
5. Ask which عزاء they attend — **men's or women's**. Funeral notices list separate venues and often separate hours for each, and the answer also decides whether the burial is added at all (see Funeral Notices below). Skip this question too if they said they won't need the funeral-notice feature.
6. Ask whether their **work calendar already subscribes to a personal calendar** (then `workBlocking: "subscribed"` plus which calendars it shows, and no invitations are ever sent); otherwise ask for their **work email address and their working hours**, and explain why: anything landing inside those hours is invited to the work address so the slot also blocks their work calendar. Leave `workEmail` unset if they decline; default `workHours` to 07:00–14:00 if they don't specify.
7. Write the answers to `~/.claude/skills/appointment-calendar-assistant/config.json`:
   ```json
   {
     "calendars": {
       "personal": { "id": "<calendar ID>", "name": "<display name>" },
       "family":   { "id": "<calendar ID>", "name": "<display name>" },
       "events":   { "id": "<calendar ID>", "name": "<display name>" }
     },
     "timezone": "+03:00",
     "city": "<city, or null if skipped>",
     "azaAttends": "men" | "women",
     "workEmail": "<work email, or null if skipped>",
     "workHours": { "start": "07:00", "end": "14:00" },
     "workDays": ["Sun", "Mon", "Tue", "Wed", "Thu"],
     "workBlocking": "invite" | "subscribed",
     "workVisibleCalendars": ["personal"]
   }
   ```
   Omit `family` and `events` entirely if the user keeps everything on one calendar.

**If it does exist**, read it silently and proceed — never re-run the interview or ask these questions again. If the user later says "use a different calendar" or similar, update the relevant field(s) in `config.json` and confirm the change.

**No writable skill folder (claude.ai):** the config file can't be saved there. Look for the same values in `personal.md` instead; if they aren't there, run the interview once for the conversation and offer the answers as a block the user can paste into `personal.md`.

**Never let setup block a clear instruction.** If the config is missing and the user has asked for something to be added, do not stall the task with the interview: pick the most sensible calendar available, create the event, write the config from what was inferred, and state in one line which calendar was used and that it can be changed. The interview is a convenience, not a gate.

### Resolving a calendar

Each event-type rule below names a **role**: `{personal}`, `{family}` or `{events}`. Resolve a role to a calendar ID like this:

- If that role is set in `config.json`, use it.
- If it isn't, fall back to `{personal}` — which is always set.

So a single-calendar user gets every event on their one calendar with no extra prompting, and a user with separate calendars gets each event routed to the right one. **Never ask which calendar to use at event-creation time** — the config already answers it.

`{timezone}`, `{city}`, `{workEmail}`, `{workHours}`, `{workDays}`, `{workBlocking}` and `{workVisibleCalendars}` come from the same file; funeral/burial prayer-time lookups use `{city}`.

---

## The Working-Hours Rule (applies to EVERY event type)

This rule is not tied to any one category. Check it for every single event before creating it, whatever kind it is — hospital appointment, interview, wedding, dinner, burial, aza.

**First check `{workBlocking}`.** If it is `"subscribed"`, the work calendar already shows the calendars listed in `{workVisibleCalendars}`, so **never invite the work address, for any event**. Instead, an event inside work hours that should block work time goes on a visible calendar (usually Personal). If it belongs on a calendar the work side can't see (Family, Personal Events), say in one line that it won't show at work and offer to put it on a visible calendar; don't move it silently. The invitation steps below do not apply, and the report says "shows at work via subscription" rather than "invitation sent".

**Otherwise (`{workBlocking}` unset or `"invite"`):** if the event's start time falls inside `{workHours}` (default 07:00–14:00 local) on a working day, and `{workEmail}` is set, invite the work address:

```
attendees: [{ email: "{workEmail}", responseStatus: "needsAction" }]
notificationLevel: "ALL"
visibility: "private"
```

The purpose is simple: anything in those hours collides with work, so the work calendar must show the slot as taken and nobody should book over it.

**Both parts are required.** The work calendar is normally a different system (Outlook / Microsoft 365), and the only thing that puts an event there is an invitation email the user accepts. Adding the attendee silently — `responseStatus: "accepted"` with `notificationLevel: "NONE"` — marks them as attending inside the personal calendar and **nothing ever reaches the work calendar**. Never do that: it looks correct in the API response and fails in practice.

**Boundary:** an event starting at or after `{workHours}.end` is outside the window. A 2:00 PM start with a 14:00 end time is out; 1:30 PM is in. Judge by the start time, not the end.

**Free-time events are not exempt.** Burial and aza entries stay `AVAILABILITY_FREE` on the personal calendar — that setting is about not cluttering personal time. If one falls inside working hours, the work invitation still goes out, because the user genuinely cannot be at work. The two settings are not in conflict: free on personal, blocked at work.

**Set `visibility: "private"`** on any event carrying a work invitation, so colleagues with access to the work calendar see the time as taken, not what it is.

**Title discretion.** `visibility: "private"` hides the detail from colleagues, but it does **not** hide the invitation email sitting in the work mailbox, and its subject line carries the title. For a job interview — or anything else the user would rather not have visible in work mail — offer a neutral title (`Personal appointment`, `External meeting`) with the real detail in the description. Say why in one line and let the user choose; don't silently pick either way.

**After creating it**, say that an invitation has gone to the work address and that accepting it is what blocks the time.

Events starting outside `{workHours}` — an evening wedding, a dinner, an aza at 5:00 PM — get no work invitation at all.

---

## Event Type Rules

### 1. Hospital / Clinic Appointments

**Calendar:** `{personal}` if the appointment is for the user themself; `{family}` if it is for a family member (partner, child, parent). If it is genuinely unclear who it is for, ask — this is the one routing question worth asking, because it also determines the title format.

**Defaults:**
- Duration: **1 hour**
- Reminders: **1 day before** + **2 hours before**

**Extract from message:**
- Patient name (the person the appointment is for)
- Doctor / department / specialty
- Date and time
- Hospital / clinic name and location (use as event location)
- Any reference/booking number (add to event description)

**Title format:** `[Patient First Name]: [Appointment Type]` — e.g. `Lina: Dental`. If the appointment is for the user themself, use `[Specialty] Appointment – Dr. [Name]` or `[Hospital] Appointment` instead. Doctor name and clinic details go in the description, not the title.

If it's genuinely unclear who the appointment is for, ask before adding.

---

### 2. Meetings, Interviews & Professional Appointments

Covers job interviews, recruiter and agency meetings, external business meetings, advisory or consultancy calls — anything scheduled that belongs to the user personally rather than to their employer's own calendar.

**Calendar:** `{personal}`. These are the user's own commitments, and interviews in particular are confidential — they never go on an employer calendar.

These almost always fall inside `{workHours}`, so the working-hours rule above will apply. Apply it — don't re-derive it here.

**Description:** plain text with real line breaks. Never `<br>` tags — they come back escaped and render literally.

**Defaults:**
- Duration: **1 hour** unless the message states one
- Reminders: **1 day before** + **2 hours before**

**Extract:**
- Organization, and the role or subject of the meeting
- Date and time
- Full street location (use as event location), or the video-call link, meeting ID and passcode for a remote interview — put the join link in `location` so it is tappable, and the ID and passcode in the description
- Arrival instructions — who to ask for at reception, floor, building, parking, ID or badge needed; for a remote call, any "join early" instruction
- Who arranged it, the panel or interviewers if named, with email or phone if given

**Title format (when discretion isn't needed):** `Interview – [Organization] ([Role])` for interviews; `Meeting – [Organization]` or `[Subject] – [Organization]` otherwise.

---

### 3. Wedding Invitations (Photo / Image)

Wedding invitations in the Gulf are typically portrait-format cards with Arabic calligraphy. They contain:
- Family/groom name (often in large decorative script)
- Date in Arabic (الاثنين الموافق dd/mm/yyyy)
- Venue name and hall number
- Time reference (usually "بعد صلاة العصر" = after Asr prayer, or "من ساء" = from evening)

**Calendar:** `{events}`

**Time defaults:**
- If time says "بعد صلاة العصر" (after Asr) or is unspecified → set **5:00 PM – 8:00 PM** (3 hours)
- If a specific time is given → use it, default duration 2 hours

**Reminders:** **1 day before** + **1 hour before**

**Extract:**
- Groom's first name (usually the largest text on the card)
- Full hosting family name
- Date (convert Arabic date format dd/mm/yyyy to ISO)
- Venue (hall name + hall number if present)

**Title format:** `عرس [First Name] [Father Name] [Last Name]`
**Location:** Venue name and hall number

---

### 4. Dinner / Gathering Invitations (Text Message)

**Calendar:** `{events}` — or `{family}` if it is clearly a family gathering rather than a social one.

**Defaults:**
- Duration: **2 hours**
- Reminders: **1 day before** + **1 hour before**

**Extract:**
- Host name or organizer
- Date and time
- Venue / restaurant / location
- Any dress code or notes (add to description)

**Title format:** `Dinner – [Host Name]` or `Dinner at [Venue]`

---

### 5. Funeral / Condolence Notices (وفاة / عزاء)

These messages are typically forwarded WhatsApp text. They mention the deceased, burial (دفن), and عزاء (condolence gathering) details.

**Calendar:** `{events}` (all entries)

**What gets created depends on `{azaAttends}` from config:**

| `azaAttends` | Burial | عزاء days |
|---|---|---|
| `men` | ✅ created | ✅ 3 days, men's venue and hours |
| `women` | ❌ not created | ✅ 3 days, women's venue and hours |

#### A) Burial (جنازة / دفن) — only when `{azaAttends}` is `men`

The burial prayer is attended by men, so **skip this event entirely when `{azaAttends}` is `women`**. Do not create it and do not mention it as something omitted — just create the عزاء days.

**Look up the burial prayer time either way.** Even when no burial event is created, the aza Day 1 counting rule below keys off when the burial prayer falls, so the lookup is still required to place the three عزاء days correctly.

When creating it:

- Use `web_search` to look up exact prayer times in `{city}` for that specific date — never guess or use a fixed table, prayer times shift daily and by location
- Add **15 minutes** after the prayer time as the event start (time for prayer + walk to graveyard)
- Duration: **1 hour**
- Location: the cemetery name mentioned in the message
- Title: `دفان [First Name] [Father Name] [Last Name]`
- Reminders: **at the event time only** — no advance reminders
- Availability: **free** (`AVAILABILITY_FREE`) — must not block time on the personal calendar
- A Dhuhr burial commonly falls inside `{workHours}` — apply the working-hours rule (invite the work address, or rely on the subscription when `{workBlocking}` is `subscribed`)

#### B) Condolence / عزاء (3 consecutive days)

**Aza day counting rule:**
- If burial is **after Asr or later** (Maghrib or Isha) → burial day does NOT count as Day 1; Day 1 = next day
- If burial is **Asr or earlier** (Fajr, Dhuhr, or Asr) → burial day IS Day 1

**Aza hours:** use the hours the notice gives for the `{azaAttends}` side — men's and women's sittings often run at different times. Fall back to **4:00 PM – 8:00 PM** only when the notice gives no hours for that side.

Create **3 separate calendar entries**, one per day, labeled:
- `عزاء [Last Name] – اليوم الأول`
- `عزاء [Last Name] – اليوم الثاني`
- `عزاء [Last Name] – اليوم الثالث`

**Location:** the venue for the `{azaAttends}` side. Notices commonly list men's and women's venues separately — use the configured one in the `location` field and ignore the other; don't clutter the entry with a venue the user won't go to. If the notice gives only one venue, use it. If a Google Maps link is given for that side, use the link as the `location` value — the calendar app renders it as a tappable link.

**Reminders:** **at the event time only** — no advance reminders (for each day entry)

**Availability:** **free** (`AVAILABILITY_FREE`) — aza entries must not block time on the personal calendar. Most aza sittings run in the afternoon and evening, so the working-hours rule usually won't fire; if one does start inside `{workHours}`, it applies as normal.

**Extract from message:**
- Deceased's name and age (if mentioned)
- Burial day and prayer name (e.g., بعد صلاة العشاء)
- Cemetery name
- عزاء venue and hours for the `{azaAttends}` side (notices often give both sides — read the one that applies)
- Google Maps link if present → use as `location` for aza entries (not just description)

---

## Step-by-Step Workflow

1. **First run only:** complete the setup interview above before proceeding — but never let it block a clear instruction.
2. **Identify event type** from the input (image = likely wedding; hospital keywords = appointment; interview, recruiter, or a scheduled visit to another organization = meeting; dinner/meal keywords = dinner; توفي/وفاة/عزاء = funeral notice)
3. **Extract all details** — be precise with dates, convert Arabic dates if needed
4. **Funerals — check `{azaAttends}` before creating anything.** If it is `women`, create only the three عزاء days, on the women's schedule and at the women's venue; no burial event. If it is `men`, create the burial plus the three days. Either way, look up the burial prayer time — Day 1 counting depends on it.
5. **Date anchoring for funerals:** Burial and aza always refer to the same day or recent past — never assume "next week". If the message says "اليوم الاثنين" or just "الاثنين" and today is Monday or Tuesday, it means this Monday just passed or today. If the burial prayer has already occurred relative to now, that's fine — still create the entries (aza days may still be upcoming).
6. **Apply the defaults** from the relevant section above
7. **Check the working-hours rule for every event** — does its start time fall inside `{workHours}`? If yes: with `{workBlocking}` = `subscribed`, make sure it is on a visible calendar and send no invitation; otherwise, if `{workEmail}` is set, it carries a work invitation. This check is per event: in a three-day aza, one day may qualify and the others not.
8. **Confirm with the user** only if a critical detail is genuinely ambiguous (e.g., unclear who a hospital appointment is for)
9. **Create the event(s)** using the calendar tool's create-event function with:
   - `calendarId`: the role resolved for this event type (see **Resolving a calendar**)
   - Correct `startTime` and `endTime` (ISO 8601, using `{timezone}` from config)
   - `location` field populated — street address, map link, or video-call join link
   - `description` in plain text with real line breaks — meeting IDs and passcodes, reference numbers, arrival instructions, deceased info. Never `<br>` tags.
   - `overrideReminders` array with the correct reminders
   - When the working-hours rule applies: `attendees` carrying `{workEmail}` as `needsAction`, `notificationLevel: "ALL"` so the invitation is actually sent, and `visibility: "private"`
   - `availability`: `AVAILABILITY_FREE` for burial and aza entries; leave it at the default (busy) for everything else
10. **Verify what was created.** Read each event back and check calendar, date and time (in `{timezone}`), location, reminders, availability, attendee list and description rendered as intended — don't report success from the create call alone. **If anything differs, correct it with the calendar tool's update function and return to this step.**
11. **Report** one line per event: title, date and time, calendar, and how it reaches work: "shows at work via subscription", "invitation sent" (accepting it is what blocks the time), or "not at work".

### Checklist

Copy this into your reply and tick it off as you go:

```
- [ ] Config read (or setup run without blocking the request)
- [ ] Event type identified, details extracted, Arabic dates converted
- [ ] Funeral only: azaAttends checked, prayer time looked up for {city} on that date, Day 1 placed
- [ ] Calendar role resolved for each event
- [ ] Defaults applied: duration, reminders, availability
- [ ] Working-hours rule checked per event (attendee + ALL + private when it fires)
- [ ] Events created
- [ ] Each event read back and matches; fixed and re-checked if not
- [ ] Reported to the user
```

---

## Reminder Templates

### Hospital Appointments, Meetings & Interviews
```
overrideReminders: [
  { minutes: 1440, method: "popup" },  // 1 day before
  { minutes: 120,  method: "popup" }   // 2 hours before
]
```

### Wedding & Dinner Invitations
```
overrideReminders: [
  { minutes: 1440, method: "popup" },  // 1 day before
  { minutes: 60,   method: "popup" }   // 1 hour before
]
```

### Burial
```
overrideReminders: [
  { minutes: 0, method: "popup" }   // at the event time
]
availability: "AVAILABILITY_FREE"
```

### Aza (each day)
```
overrideReminders: [
  { minutes: 0, method: "popup" }   // at the event time
]
availability: "AVAILABILITY_FREE"
```

### Work invitation — any event starting inside `{workHours}`
```
attendees: [{ email: "{workEmail}", responseStatus: "needsAction" }]
notificationLevel: "ALL"      // required — NONE means the work calendar never sees it
visibility: "private"
```

Burial and aza entries are the **only** events created as free time on the personal calendar. Hospital, meeting, interview, wedding and dinner events stay busy (the calendar default) and keep their advance reminders. Free time on the personal calendar does not cancel a work invitation.

---

## Arabic Date Parsing

Common patterns in Gulf invitations and notices:
- `الاثنين الموافق 27/4/2026م` → Monday, April 27, 2026
- `يوم الجمعة 3 مايو` → Friday, May 3 (infer year from context)
- `بعد صلاة العصر` → After Asr → prayer time + 15 min
- `بعد صلاة العشاء` → After Isha → prayer time + 15 min
- `بعد صلاة المغرب` → After Maghrib → prayer time + 15 min
- `من العصر` / `من ساء يوم` → from Asr onward → use 5:00 PM for aza

All times use the `{timezone}` offset from config.

---

## Example Outputs

### Job Interview, 10:00 AM (inside work hours)
- Calendar: `{personal}` | Duration: 1 hr | Reminders: 1 day + 2 hrs
- Work email invited with `needsAction` and `notificationLevel: "ALL"` — the invitation email is what puts it on the work calendar
- `visibility: "private"` so colleagues see busy, not detail
- Neutral title offered (`Personal appointment`) with the real detail in the description, since the invitation's subject line lands in the work mailbox

### Dental Appointment, 9:30 AM (inside work hours)
- Calendar: `{personal}` or `{family}` | Duration: 1 hr | Reminders: 1 day + 2 hrs
- Work invitation sent — the rule is about the time, not the kind of event

### Hospital Appointment (for a family member)
- Calendar: `{family}` | Title: `[First Name]: [Type]` e.g. `Lina: Dental` | Duration: 1 hr | Reminders: 1 day + 2 hrs
- Working-hours rule still applies if the user is going with them

### Wedding, 5:00 PM (outside work hours)
- Calendar: `{events}` | Title: عرس [Groom Name] | Time: 5:00–8:00 PM | Reminders: 1 day + 1 hr
- No work invitation

### Funeral Notice — `azaAttends: men`
- Burial: Monday بعد صلاة العشاء (Isha, looked up for `{city}` that date) → start 15 min after Isha, at the named cemetery. Isha is outside work hours, so no work invitation. A Dhuhr burial would get one.
- If Isha is after Asr → Monday does NOT count as Day 1
- Aza Day 1: Tuesday, men's hours from the notice (default 4:00–8:00 PM), men's venue, calendar `{events}`
- Aza Day 2: Wednesday, same
- Aza Day 3: Thursday, same
- All four entries: reminder at the event time only, marked free

### Funeral Notice — `azaAttends: women`
Same notice, four events become three:
- **No burial event.**
- Isha still looked up, because Day 1 counting depends on it — Isha is after Asr, so Monday is not Day 1.
- Aza Day 1: Tuesday, women's hours from the notice, women's venue, calendar `{events}`
- Aza Day 2: Wednesday, same
- Aza Day 3: Thursday, same
- All three entries: reminder at the event time only, marked free

### Dinner Invitation
- Calendar: `{events}` | Duration: 2 hrs | Reminders: 1 day + 1 hr | No work invitation (evening)
