# mu — Non-Doing Reminder

A micro Go app that pings every 90 minutes inviting a 5-minute pause for being/non-doing.

## Purpose

Technology supporting stillness, not productivity. Regular invitations to stop, regardless of flow state.

## Architecture

- **CLI:** `mu heeded|skip|afk|status|report`
- **Daemon:** systemd timer fires every 90 mins (08:00-22:00)
- **Notification:** notify-send desktop notification
- **Storage:** SQLite tracking every ping and response

## Database Schema

```sql
CREATE TABLE pings (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    response TEXT CHECK(response IN ('heeded', 'skip', 'afk', 'pending')),
    responded_at DATETIME,
    duration_seconds INTEGER  -- if heeded, how long
);
```

## Commands

- `mu` — Record current ping as heeded (starts 5-min timer)
- `mu skip` — Record as skipped
- `mu afk` — Record as away
- `mu status` — Show today's pings
- `mu report` — Weekly digest (% heeded, patterns)
- `mu ping` — Manually trigger a ping (for testing)

## Systemd

- `mu.timer` — OnCalendar every 90 min, 08:00-22:00
- `mu.service` — Runs `mu ping` which sends notification

## Tech Stack

- Go 1.21+
- SQLite via modernc.org/sqlite (pure Go)
- notify-send for desktop notifications
