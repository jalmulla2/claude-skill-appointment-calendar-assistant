---
name: appointment-calendar-assistant
description: Intelligently add calendar events from various message formats — hospital/clinic appointments, wedding invitation photos (Arabic calligraphy cards), dinner/gathering invitations, and condolence/funeral/aza notices (وفاة / عزاء / دفن messages). Trigger whenever the user shares an appointment message, invitation image, funeral notice, or event details and wants it added to the calendar. Trigger phrases include: "add this to calendar", "book this", "save this appointment", "hospital appointment", "clinic appointment", "wedding invitation", "dinner invitation", "save the date", "توفي", "وفاة", "عزاء", "دفن", "جنازة", or whenever an image of a wedding/event card is shared. Always use this skill — do not just call the calendar tool directly — because this skill applies the correct routing logic, default durations, prayer-time lookups, and reminder rules.
---

# Appointment & Calendar Assistant

Extracts key details from invitations and appointment messages and adds them to your calendar with the right duration and reminders — no manual entry needed.

---

## First-Run Setup

Before doing anything else, check whether `~/.claude/skills/appointment-calendar-assistant/config.json` exists.

**If it does not exist**, this is the first use — run this interview once:

1. Call the calendar tool's list-calendars function and show the user the calendars available to them.
2. Ask: "Which single calendar should I use for all appointments and events (hospital visits, weddings, dinners, funerals — everything goes on one calendar unless you tell me otherwise)?"
3. Ask for their timezone as a UTC offset (e.g. `+03:00`). If the calendar tool exposes a timezone for the account/calendar, propose it and just ask them to confirm.
4. Ask for the city they live in. Explain why: it's only used to look up accurate prayer times when logging a funeral/burial notice — skip this question if the user says they won't need the funeral-notice feature.
5. Write the answers to `~/.claude/skills/appointment-calendar-assistant/config.json`:
   ```json
   {
     "calendarId": "<the chosen calendar's ID>",
     "calendarName": "<its display name>",
     "timezone": "+03:00",
     "city": "<city, or null if skipped>"
   }
   ```

**If it does exist**, read it silently and proceed — never re-run the interview or ask these questions again. If the user later says "use a different calendar" or similar, update the relevant field(s) in `config.json` and confirm the change.

Every rule below refers to `{calendarId}` and `{timezone}` from this config file. Funeral/burial prayer-time lookups use `{city}`.

---

## Event Type Rules

### 1. Hospital / Clinic Appointments

**Calendar:** `{calendarId}`

**Defaults:**
- Duration: **1 hour**
- Reminders: **1 day before** + **2 hours before**

**Extract from message:**
- Patient name (the person the appointment is for)
- Doctor / department / specialty
- Date and time
- Hospital / clinic name and location (use as event location)
- Any reference/booking number (add to event description)

**Title format:** `[Patient First Name]: [Appointment Type]` — e.g. `Sara: Dental`. If the appointment is for the user themself, use `[Specialty] Appointment – Dr. [Name]` or `[Hospital] Appointment` instead. Doctor name and clinic details go in the description, not the title.

If it's genuinely unclear who the appointment is for, ask before adding.

---

### 2. Wedding Invitations (Photo / Image)

Wedding invitations in the Gulf are typically portrait-format cards with Arabic calligraphy. They contain:
- Family/groom name (often in large decorative script)
- Date in Arabic (الاثنين الموافق dd/mm/yyyy)
- Venue name and hall number
- Time reference (usually "بعد صلاة العصر" = after Asr prayer, or "من ساء" = from evening)

**Calendar:** `{calendarId}`

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

### 3. Dinner / Gathering Invitations (Text Message)

**Calendar:** `{calendarId}`

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

### 4. Funeral / Condolence Notices (وفاة / عزاء)

These messages are typically forwarded WhatsApp text. They mention the deceased, burial (دفن), and عزاء (condolence gathering) details.

**Calendar:** `{calendarId}` (all entries)

**Creates TWO types of events:**

#### A) Burial (جنازة / دفن)

- Use `web_search` to look up exact prayer times in `{city}` for that specific date — never guess or use a fixed table, prayer times shift daily and by location
- Add **15 minutes** after the prayer time as the event start (time for prayer + walk to graveyard)
- Duration: **1 hour**
- Location: the cemetery name mentioned in the message
- Title: `دفان [First Name] [Father Name] [Last Name]`
- Reminders: **2 hours before** + **1 hour before**

#### B) Condolence / عزاء (3 consecutive days)

**Aza day counting rule:**
- If burial is **after Asr or later** (Maghrib or Isha) → burial day does NOT count as Day 1; Day 1 = next day
- If burial is **Asr or earlier** (Fajr, Dhuhr, or Asr) → burial day IS Day 1

**Aza hours:** **4:00 PM – 8:00 PM** for 3 consecutive days

Create **3 separate calendar entries**, one per day, labeled:
- `عزاء [Last Name] – اليوم الأول`
- `عزاء [Last Name] – اليوم الثاني`
- `عزاء [Last Name] – اليوم الثالث`

**Location:** the عزاء venue name mentioned. If a Google Maps link is present in the message, use it as the `location` field value — the calendar app renders it as a tappable link.

**Reminders:** **1 day before** + **1 hour before** (for each day entry)

**Extract from message:**
- Deceased's name and age (if mentioned)
- Burial day and prayer name (e.g., بعد صلاة العشاء)
- Cemetery name
- عزاء location(s)
- Google Maps link if present → use as `location` for aza entries (not just description)

---

## Step-by-Step Workflow

1. **First run only:** complete the setup interview above before proceeding.
2. **Identify event type** from the input (image = likely wedding; hospital keywords = appointment; dinner/meal keywords = dinner; توفي/وفاة/عزاء = funeral notice)
3. **Extract all details** — be precise with dates, convert Arabic dates if needed
4. **Date anchoring for funerals:** Burial and aza always refer to the same day or recent past — never assume "next week". If the message says "اليوم الاثنين" or just "الاثنين" and today is Monday or Tuesday, it means this Monday just passed or today. If the burial prayer has already occurred relative to now, that's fine — still create the entries (aza days may still be upcoming).
5. **For funerals:** use `web_search` to look up exact prayer times in `{city}` for that specific date
6. **Apply the defaults** from the relevant section above
7. **Confirm with the user** only if a critical detail is genuinely ambiguous (e.g., unclear who a hospital appointment is for)
8. **Create the event(s)** using the calendar tool's create-event function with:
   - `calendarId`: `{calendarId}` from config
   - Correct `startTime` and `endTime` (ISO 8601, using `{timezone}` from config)
   - `location` field populated
   - `eventDescription` with extra details (map links, reference numbers, deceased info, etc.)
   - `nudges` array with the correct reminders

---

## Reminder Templates

### Hospital Appointments
```
nudges: [
  { minutesBefore: 1440, method: "notification" },  // 1 day before
  { minutesBefore: 120,  method: "notification" }   // 2 hours before
]
```

### Wedding & Dinner Invitations
```
nudges: [
  { minutesBefore: 1440, method: "notification" },  // 1 day before
  { minutesBefore: 60,   method: "notification" }   // 1 hour before
]
```

### Burial
```
nudges: [
  { minutesBefore: 120, method: "notification" },  // 2 hours before
  { minutesBefore: 60,  method: "notification" }   // 1 hour before
]
```

### Aza (each day)
```
nudges: [
  { minutesBefore: 1440, method: "notification" },  // 1 day before
  { minutesBefore: 60,   method: "notification" }   // 1 hour before
]
```

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

### Funeral Notice
- Burial: Monday بعد صلاة العشاء (Isha, looked up for `{city}` that date) → start 15 min after Isha, at the named cemetery
- If Isha is after Asr → Monday does NOT count as Day 1
- Aza Day 1: Tuesday 4:00–8:00 PM
- Aza Day 2: Wednesday 4:00–8:00 PM
- Aza Day 3: Thursday 4:00–8:00 PM

### Wedding
- Title: عرس [Groom Name] | Time: 5:00–8:00 PM | Reminders: 1 day + 1 hr

### Hospital Appointment
- Title: `[First Name]: [Type]` e.g. `Sara: Dental` | Duration: 1 hr | Reminders: 1 day + 2 hrs

### Dinner Invitation
- Duration: 2 hrs | Reminders: 1 day + 1 hr
