# Appointment & Calendar Assistant — Claude Skill

A [Claude skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that reads appointment messages, invitation photos, and notices, then adds them to your calendar with the right title, duration, and reminders — no manual entry needed.

Handles:
- Hospital / clinic appointments
- Wedding invitation photos (Arabic calligraphy cards)
- Dinner / gathering invitations
- Funeral / condolence (وفاة / عزاء) notices, with live prayer-time lookups

## Requirements

- [Claude Code](https://claude.com/claude-code)
- A calendar MCP tool connected to your Claude Code setup (one that can list calendars, create events, and search the web for prayer times when needed)

## Install

1. Clone it into your Claude skills directory (the folder must be named `appointment-calendar-assistant`):
   ```bash
   git clone https://github.com/jalmulla2/claude-skill-appointment-calendar-assistant.git ~/.claude/skills/appointment-calendar-assistant
   ```
2. Restart Claude Code / Claude Desktop.
3. The first time you trigger the skill (e.g. "add this appointment to my calendar"), it will ask you which calendar to use, your timezone, and your city — then remember your answers for next time. Nothing is pre-configured; it's the same skill for everyone until you set it up.

## Notes

This is a general-purpose version shared for anyone to use — it has no personal data baked in. Your calendar choice, timezone, and city are stored locally in a `config.json` file next to the skill, created the first time you use it.
