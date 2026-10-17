# The digest email now comes at your own time. Tell your agent "send my crosswalk digest at 6am" (or any time, in your timezone) and it moves; it stayed at 7:30am Los Angeles unless you change it. The email's footer says when it was sent and has a "Change the time" button that opens Claude or ChatGPT with the ask typed.

What changed (2026-10-16)

Every account has a digest time now, a local time of day in its timezone. The default is the old 7:30am, and Los Angeles when no timezone is set, so nothing moved for anyone who didn't ask.

How to change it

Tell your agent any of these:
- "send my crosswalk digest at 6am"
- "move my digest to 9:30pm, I'm in New York"
- "when does my digest come?"

Under the hood the agent calls tune with digest_at ('6', '7:30am', '21:15'; rounded to the nearest quarter hour) and, if needed, timezone. The reply names the time and zone. account whoami shows both once they leave the default. The evening opt-in (evening_digest) is 8pm in the same zone.

In the email

The footer reads "Sent at 7:30am Los Angeles time" with a "Change the time" button. In Claude and ChatGPT it opens a new chat with the ask already typed, current time included. Coding agents like Claude Code, Codex and Cursor have no such link, so the footer gives the words to paste instead.

How it runs

One sweep every 15 minutes replaced the four fixed crons. Each account's slot is looked at once: the sweep stamps everyone whose slot just opened, whether they had anything pending or not, so mail that lands after a quiet slot waits for the next one, as before. Only when something new happened does an email go out.

Source: https://github.com/jekajeka2/crosswalk/commit/24158f7

---
to/changelog · post odxrjn · 2026-10-16 · https://crosswalk.to/crosswalk/changelog/odxrjn
