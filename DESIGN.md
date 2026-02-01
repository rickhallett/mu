# mu — Design Document

## Overview

mu is a mindfulness pause reminder that pings every 90 minutes, inviting a 5-minute pause for non-doing. It tracks responses (heeded/skip/afk) in SQLite for pattern analysis.

## CLI Interface

```bash
mu              # Record as heeded, start 5-min timer
mu skip         # Record as skipped
mu afk          # Record as away from keyboard
mu status       # Show today's pings
mu report       # Weekly digest with patterns
mu ping         # Manual trigger (for testing/systemd)
```

## Database Schema

```sql
CREATE TABLE pings (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    response TEXT CHECK(response IN ('heeded', 'skip', 'afk', 'pending')),
    responded_at DATETIME,
    duration_seconds INTEGER
);

CREATE INDEX idx_pings_timestamp ON pings(timestamp);
CREATE INDEX idx_pings_response ON pings(response);
```

## Notification Flow

1. Timer fires → `mu ping` runs
2. Insert row with `response='pending'`
3. Send desktop notification via `notify-send`
4. User responds with `mu` / `mu skip` / `mu afk`
5. Update row with response and timestamp
6. If no response after 10 minutes → auto-mark `afk`

## Systemd Integration

**mu.timer:**
```ini
[Unit]
Description=mu pause reminder timer

[Timer]
OnCalendar=*-*-* 08:00,09:30,11:00,12:30,14:00,15:30,17:00,18:30,20:00,21:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

**mu.service:**
```ini
[Unit]
Description=mu pause reminder ping

[Service]
Type=oneshot
ExecStart=/home/mrkai/.local/bin/mu ping
```

## Directory Structure

```
mu/
├── cmd/mu/
│   └── main.go           # CLI entry point (cobra)
├── internal/
│   ├── db/
│   │   └── db.go         # SQLite operations
│   └── notify/
│       └── notify.go     # Desktop notifications
├── systemd/
│   ├── mu.service
│   └── mu.timer
├── CLAUDE.md
├── DESIGN.md
├── go.mod
└── README.md
```

## Implementation Phases

1. **Phase 1: Core** (mu-ezg.2)
   - SQLite database layer
   - InitDB, RecordPing, UpdateResponse

2. **Phase 2: CLI** (mu-ezg.4)
   - Cobra CLI with all commands
   - Status and report formatting

3. **Phase 3: Notifications** (mu-ezg.3)
   - notify-send integration
   - Auto-AFK timeout

4. **Phase 4: Timer** (mu-ezg.5)
   - systemd service and timer
   - Installation script

5. **Phase 5: HAL Integration** (mu-ezg.6)
   - Weekly digest to HAL
   - Pattern recommendations

## Tech Stack

- Go 1.21+
- modernc.org/sqlite (pure Go SQLite)
- spf13/cobra (CLI)
- notify-send (Linux desktop notifications)
