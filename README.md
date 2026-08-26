# Appointment & Calendar Assistant (Claude Code Skill)

A [Claude Code](https://claude.com/claude-code) skill that reads appointment messages, invitation photos, and notices, then adds them to your calendar with the right title, duration, and reminders — no manual entry needed.

Handles:
- Hospital / clinic appointments
- Wedding invitation photos (Arabic calligraphy cards)
- Dinner / gathering invitations
- Funeral / condolence (وفاة / عزاء) notices, with live prayer-time lookups

## Requirements

- [Claude Code](https://claude.com/claude-code)
- A calendar MCP tool connected to your Claude Code setup (one that can list calendars, create events, and search the web for prayer times when needed)

## Install

1. Copy the `appointment-calendar-assistant` folder into your Claude skills directory:
   ```bash
   cp -r appointment-calendar-assistant ~/.claude/skills/appointment-calendar-assistant
   ```
2. Restart Claude Code / Claude Desktop.
3. The first time you trigger the skill (e.g. "add this appointment to my calendar"), it will ask you which calendar to use, your timezone, and your city — then remember your answers for next time. Nothing is pre-configured; it's the same skill for everyone until you set it up.

## Notes

This is a general-purpose version shared for anyone to use — it has no personal data baked in. Your calendar choice, timezone, and city are stored locally in a `config.json` file next to the skill, created the first time you use it.
