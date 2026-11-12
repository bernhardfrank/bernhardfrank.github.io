# Ignite 2026 Planner — Hybrid & Infrastructure

A static, single-page planner for Microsoft Ignite 2026 sessions on Hyper-V, Windows Server,
Azure Local, Azure Arc, hybrid and Azure infrastructure.

- Pick sessions (star rating, search, sort by rating or time)
- See them on a 4-day calendar, with time clashes highlighted
- Switch time zone (event time, your local time, CET, UK, US Eastern, UTC)
- **Download .ics**: imports into Outlook, Google Calendar and Apple Calendar (with a 10-minute reminder)
- **Copy share link**: the selection is encoded in the URL (`#s=BRK418,BRK419,…`)

No build step, no dependencies. Selection is saved in the visitor's browser (localStorage).

## Files

| File | Purpose |
|---|---|
| `index.html` | The page (HTML, CSS, JS) |
| `data.js` | Session data and star ratings. Edit this file to add or re-rate sessions. |

Times in `data.js` are UTC (ISO 8601). `start: null` marks an on-demand session.

## Notes

Unofficial community tool, not affiliated with Microsoft. Session data from the public
[Ignite session catalog](https://ignite.microsoft.com/en-US/sessions) as of 2026-11-12.
Times and rooms can change, so check the official catalog before you go.
