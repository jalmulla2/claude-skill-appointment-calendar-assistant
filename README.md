# Appointment & Calendar Assistant — Claude Skill

A [Claude skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that reads appointment messages, invitation photos, and notices, then adds them to your calendar with the right title, duration, and reminders — no manual entry needed.

Handles:
- Hospital / clinic appointments (yours or a family member's)
- Meetings and job interviews, with a discreet title option
- Wedding invitation photos (Arabic calligraphy cards)
- Dinner / gathering invitations
- Funeral / condolence (وفاة / عزاء) notices, with live prayer-time lookups and men's or women's عزاء schedules

Anything that starts inside your working hours can also send an invitation to your work email, so the slot shows as busy on your work calendar.

## Requirements

- Claude Code, Cowork, or claude.ai
- A connected calendar tool that can list calendars and create and read events
- Web search (used for prayer times on funeral notices)

## Install

**Claude Code / Cowork** — clone it into your skills directory (the folder must be named `appointment-calendar-assistant`), then restart:
```bash
git clone https://github.com/jalmulla2/claude-skill-appointment-calendar-assistant.git ~/.claude/skills/appointment-calendar-assistant
```

**claude.ai** — download the repo as a zip and upload it under Settings → Capabilities → Skills.

The first time you trigger the skill (e.g. "add this appointment to my calendar"), it asks which calendar(s) to use, your timezone, your city, which عزاء you attend, and optionally your work email and hours — then remembers the answers in a local `config.json`. It never holds up a clear request to run that interview.

## Personal overrides

Put anything specific to you (calendar IDs on claude.ai, names, house rules) in a `personal.md` next to `SKILL.md`. The skill reads it first and it overrides the defaults. Keep that file out of any public fork.

## Notes

This is a general-purpose version with no personal data baked in.
