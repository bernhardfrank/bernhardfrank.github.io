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
| `data.js` | All Ignite sessions plus the curated "Hybrid & Infra" ratings |
| `update-data.js` | Refreshes `data.js` from the official catalog (Node 18+) |

List filters: **Hybrid & Infra** (all rated picks, default), **★★★**, **★★**, **★**, **All Ignite** (every session), plus a session-type filter.

### Re-rating sessions

Edit the `curated` block at the top of `data.js`: `stars` (1–3) and an optional short `abstract`.
Do not edit the `sessions` array by hand.

### Refreshing the catalog (times and rooms change before the event)

```bash
node update-data.js
```

This downloads the current catalog, rebuilds `sessions` and keeps your `curated` block.
Times are UTC (ISO 8601). `start: null` marks an on-demand session. Sessions that run more than
once get one entry per time slot (`LAB724-1`, `LAB724-2`, …).

## Publish on GitHub Pages

1. Create a repository, e.g. `ignite-2026-planner`, and push `index.html`, `data.js` and `README.md` to the `main` branch.
2. In the repository, open **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, pick `main` and `/ (root)`, then **Save**.
4. After a minute the page is live at `https://<your-user>.github.io/ignite-2026-planner/`.

```bash
git init
git add index.html data.js README.md
git commit -m "Ignite 2026 planner"
git branch -M main
git remote add origin https://github.com/<your-user>/ignite-2026-planner.git
git push -u origin main
```

## Notes

Unofficial community tool, not affiliated with Microsoft. Session data from the public
[Ignite session catalog](https://ignite.microsoft.com/en-US/sessions) as of 2026-11-12.
Times and rooms can change, so check the official catalog before you go.
