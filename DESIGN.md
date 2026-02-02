# mu — Comprehensive Architecture Document

> Non-doing practice reminders every 90 minutes, 08:00-22:00

## Overview

mu is a mindfulness pause system that interrupts your work day with gentle reminders to practice non-doing. It tracks your responses (heeded/skip/afk), builds statistics over time, and generates weekly digests for HAL integration.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SYSTEMD LAYER                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────────┐ │
│  │  mu.timer    │────▶│  mu.service  │────▶│  /usr/local/bin/mu ping      │ │
│  │  (90min)     │     │  (oneshot)   │     │  (within 08:00-22:00)        │ │
│  └──────────────┘     └──────────────┘     └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               CLI LAYER                                      │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  mu                    │ mu skip     │ mu afk      │ mu status      │    │
│  │  (heeded + timer)      │ (skip ping) │ (was away)  │ (today's view) │    │
│  ├─────────────────────────────────────────────────────────────────────┤    │
│  │  mu history            │ mu stats    │ mu pause    │ mu config      │    │
│  │  (browse pings)        │ (metrics)   │ (temp off)  │ (settings)     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌──────────────────────┐  ┌──────────────────┐  ┌──────────────────────────────┐
│   NOTIFICATION SVC   │  │   DATABASE SVC   │  │       CONFIG LAYER           │
│                      │  │                  │  │                              │
│  • notify-send       │  │  • SQLite3       │  │  ~/.config/mu/config.toml    │
│  • Sound (paplay)    │  │  • Pure Go       │  │  • pause_until               │
│  • Action buttons    │  │  • WAL mode      │  │  • sound_enabled             │
│                      │  │                  │  │  • notification_timeout      │
└──────────────────────┘  └────────┬─────────┘  └──────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │     ~/.local/share/mu/       │
                    │           mu.db              │
                    │                              │
                    │  ┌────────────────────────┐  │
                    │  │ pings                  │  │
                    │  │ config                 │  │
                    │  │ daily_stats (view)     │  │
                    │  │ weekly_stats (view)    │  │
                    │  └────────────────────────┘  │
                    └──────────────────────────────┘
```

---

## 1. SQLite Schema

### Location
```
~/.local/share/mu/mu.db
```

### CREATE Statements

```sql
-- Enable WAL mode for concurrent reads
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;

-- =============================================================================
-- CORE TABLE: pings
-- =============================================================================
-- Every reminder event, whether responded to or not
CREATE TABLE IF NOT EXISTS pings (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    
    -- When the ping was triggered
    triggered_at    DATETIME NOT NULL DEFAULT (datetime('now', 'localtime')),
    
    -- Response type: pending → heeded|skip|afk|expired
    response        TEXT NOT NULL DEFAULT 'pending'
                    CHECK(response IN ('pending', 'heeded', 'skip', 'afk', 'expired')),
    
    -- When user responded (NULL if pending/expired)
    responded_at    DATETIME,
    
    -- Response latency in seconds (computed on response)
    latency_seconds INTEGER,
    
    -- Duration of pause in seconds (only for heeded, NULL otherwise)
    pause_duration  INTEGER,
    
    -- Optional note from user
    note            TEXT,
    
    -- Day of week (1=Mon, 7=Sun) for easy aggregation
    day_of_week     INTEGER GENERATED ALWAYS AS (
                        CAST(strftime('%w', triggered_at) AS INTEGER)
                    ) STORED,
    
    -- Hour of day (0-23) for time-of-day analysis
    hour_of_day     INTEGER GENERATED ALWAYS AS (
                        CAST(strftime('%H', triggered_at) AS INTEGER)
                    ) STORED
);

-- Indexes for common queries
CREATE INDEX IF NOT EXISTS idx_pings_triggered_at ON pings(triggered_at);
CREATE INDEX IF NOT EXISTS idx_pings_response ON pings(response);
CREATE INDEX IF NOT EXISTS idx_pings_day_of_week ON pings(day_of_week);
CREATE INDEX IF NOT EXISTS idx_pings_hour_of_day ON pings(hour_of_day);

-- =============================================================================
-- TABLE: config
-- =============================================================================
-- Key-value configuration stored in DB (supplements TOML file)
CREATE TABLE IF NOT EXISTS config (
    key         TEXT PRIMARY KEY,
    value       TEXT NOT NULL,
    updated_at  DATETIME DEFAULT (datetime('now', 'localtime'))
);

-- Initial config values
INSERT OR IGNORE INTO config (key, value) VALUES 
    ('pause_until', ''),           -- ISO8601 datetime when to resume
    ('default_pause_duration', '300'),  -- 5 minutes in seconds
    ('afk_timeout', '600'),        -- 10 minutes to auto-expire
    ('sound_enabled', 'true'),
    ('notification_timeout', '0'); -- 0 = persist until dismissed

-- =============================================================================
-- VIEW: daily_stats
-- =============================================================================
-- Aggregated statistics per day
CREATE VIEW IF NOT EXISTS daily_stats AS
SELECT 
    date(triggered_at) as date,
    COUNT(*) as total_pings,
    SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) as heeded,
    SUM(CASE WHEN response = 'skip' THEN 1 ELSE 0 END) as skipped,
    SUM(CASE WHEN response = 'afk' THEN 1 ELSE 0 END) as afk,
    SUM(CASE WHEN response = 'expired' THEN 1 ELSE 0 END) as expired,
    SUM(CASE WHEN response = 'pending' THEN 1 ELSE 0 END) as pending,
    ROUND(100.0 * SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) / 
          NULLIF(COUNT(*) - SUM(CASE WHEN response = 'pending' THEN 1 ELSE 0 END), 0), 1) 
          as heed_rate,
    ROUND(AVG(CASE WHEN response = 'heeded' THEN latency_seconds END), 0) 
          as avg_latency,
    SUM(CASE WHEN response = 'heeded' THEN pause_duration ELSE 0 END) 
          as total_pause_time
FROM pings
GROUP BY date(triggered_at);

-- =============================================================================
-- VIEW: weekly_stats
-- =============================================================================
-- Aggregated statistics per ISO week
CREATE VIEW IF NOT EXISTS weekly_stats AS
SELECT 
    strftime('%Y-W%W', triggered_at) as week,
    COUNT(*) as total_pings,
    SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) as heeded,
    SUM(CASE WHEN response = 'skip' THEN 1 ELSE 0 END) as skipped,
    SUM(CASE WHEN response = 'afk' THEN 1 ELSE 0 END) as afk,
    SUM(CASE WHEN response = 'expired' THEN 1 ELSE 0 END) as expired,
    ROUND(100.0 * SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) / 
          NULLIF(COUNT(*) - SUM(CASE WHEN response = 'pending' THEN 1 ELSE 0 END), 0), 1) 
          as heed_rate,
    ROUND(AVG(CASE WHEN response = 'heeded' THEN latency_seconds END), 0) 
          as avg_latency,
    SUM(CASE WHEN response = 'heeded' THEN pause_duration ELSE 0 END) / 60 
          as total_pause_minutes
FROM pings
GROUP BY strftime('%Y-W%W', triggered_at);

-- =============================================================================
-- VIEW: hourly_pattern
-- =============================================================================
-- Which hours have best/worst heed rates
CREATE VIEW IF NOT EXISTS hourly_pattern AS
SELECT 
    hour_of_day,
    COUNT(*) as total_pings,
    SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) as heeded,
    ROUND(100.0 * SUM(CASE WHEN response = 'heeded' THEN 1 ELSE 0 END) / 
          NULLIF(COUNT(*), 0), 1) as heed_rate
FROM pings
WHERE response != 'pending'
GROUP BY hour_of_day
ORDER BY hour_of_day;
```

### Response Type Definitions

| Response | Meaning | Trigger |
|----------|---------|---------|
| `pending` | Awaiting response | Initial state when ping fires |
| `heeded` | User took the pause | `mu` or `mu heed` command |
| `skip` | User consciously skipped | `mu skip` command |
| `afk` | User was away | `mu afk` command (manual) |
| `expired` | No response within timeout | Auto-set after `afk_timeout` |

---

## 2. Notification System

### Interface Contract

```go
// internal/notify/notify.go

package notify

import "time"

// NotifyConfig holds notification preferences
type NotifyConfig struct {
    SoundEnabled bool          // Play sound with notification
    SoundFile    string        // Path to sound file (or "" for default)
    Timeout      time.Duration // 0 = persist until dismissed
    Urgency      string        // low, normal, critical
}

// Notifier sends desktop notifications
type Notifier interface {
    // Send displays a notification and returns immediately
    Send(title, body string, cfg NotifyConfig) error
    
    // SendWithActions displays notification with action buttons
    // Returns the action ID clicked (or "" if dismissed/timeout)
    SendWithActions(title, body string, actions []Action, cfg NotifyConfig) (string, error)
}

// Action represents a notification action button
type Action struct {
    ID    string // Internal identifier (e.g., "heed", "skip")
    Label string // Display text (e.g., "Take Pause", "Skip")
}

// DefaultNotifier returns platform-appropriate notifier
func DefaultNotifier() Notifier
```

### Implementation Details

**Linux (notify-send + paplay):**
```bash
# Basic notification
notify-send \
    --app-name="mu" \
    --urgency=normal \
    --icon=appointment-soon \
    --expire-time=0 \
    "μ — Pause" \
    "Time for 5 minutes of non-doing"

# With sound
paplay /usr/share/sounds/freedesktop/stereo/bell.oga &

# With actions (requires notify-send from libnotify 0.7.9+)
notify-send \
    --app-name="mu" \
    --action="heed=Take Pause" \
    --action="skip=Skip" \
    "μ — Pause" \
    "Time for 5 minutes of non-doing"
```

### Notification Content

**Title:** `μ — Pause`

**Body variants (rotate for variety):**
```
• Time for 5 minutes of non-doing
• A moment of stillness awaits
• Return to presence
• Let go, just for now
• The pause between thoughts
• Nothing to do, nowhere to go
• Rest in awareness
```

**Sound options:**
```
Default:  /usr/share/sounds/freedesktop/stereo/bell.oga
Gentle:   /usr/share/sounds/freedesktop/stereo/message.oga
Custom:   ~/.local/share/mu/sounds/ping.oga
```

---

## 3. CLI Interface

### Command Structure

```
mu                         # Record heeded, start 5-min timer
mu heed [--duration=5m]    # Alias for bare mu, explicit heed
mu skip [--reason="..."]   # Record as skipped
mu afk                     # Record as away from keyboard

mu status                  # Today's pings, current state
mu history [--days=7]      # Browse past pings
mu stats [--period=week]   # Aggregate statistics

mu pause [duration]        # Pause reminders (default: 2h)
mu resume                  # Resume reminders early

mu config                  # Show current configuration
mu config set KEY VALUE    # Set configuration value

mu ping                    # Trigger ping (for systemd/testing)
mu digest [--format=json]  # Generate weekly digest

mu version                 # Show version info
mu help [command]          # Show help
```

### Command Specifications

#### `mu` / `mu heed`
```
USAGE:
    mu [heed] [OPTIONS]

OPTIONS:
    --duration, -d    Pause duration (default: 5m)
    --note, -n        Add a note to this ping

BEHAVIOR:
    1. Find most recent pending ping (within last 15 min)
    2. If found: update to heeded, record latency
    3. If not found: create new ping as heeded (manual trigger)
    4. Start countdown timer (visual in terminal)
    5. Play gentle chime when timer ends (optional)

OUTPUT:
    μ heeded — 5:00 remaining
    ████████████████████░░░░░░░░░░ 3:42
    
EXIT CODES:
    0 - Success
    1 - No pending ping found (with --strict)
```

#### `mu skip`
```
USAGE:
    mu skip [OPTIONS]

OPTIONS:
    --reason, -r    Why skipping (stored as note)

BEHAVIOR:
    1. Find most recent pending ping
    2. Update to skipped with timestamp
    3. Record optional reason

OUTPUT:
    μ skipped — next ping in ~90 min

EXIT CODES:
    0 - Success
    1 - No pending ping to skip
```

#### `mu afk`
```
USAGE:
    mu afk

BEHAVIOR:
    1. Find pending pings from today
    2. Mark all as afk
    3. Useful for returning after being away

OUTPUT:
    μ marked 3 pings as afk
```

#### `mu status`
```
USAGE:
    mu status [OPTIONS]

OPTIONS:
    --json    Output as JSON

BEHAVIOR:
    Shows today's pings and current state

OUTPUT:
    μ status — Mon 2025-02-03
    
    08:00  ● heeded   (4m 32s latency, 5m pause)
    09:30  ● heeded   (1m 12s latency, 5m pause)  
    11:00  ○ skipped  (deep in flow)
    12:30  ◌ pending  (triggered 2m ago)
    
    Today: 2/3 heeded (67%) | 1 skip | 0 afk
    Streak: 3 days with ≥50% heed rate
    
    Next ping: ~14:00 (in 88 min)
```

#### `mu history`
```
USAGE:
    mu history [OPTIONS]

OPTIONS:
    --days, -d     Days to show (default: 7)
    --response     Filter by response type
    --json         Output as JSON

OUTPUT:
    μ history — last 7 days
    
    2025-02-03 Mon  ●●○◌    3/4 (75%)
    2025-02-02 Sun  ●●●●●   5/5 (100%)  ★
    2025-02-01 Sat  ●●○○●   3/5 (60%)
    2025-01-31 Fri  ●◐◐●●   3/5 (60%)
    2025-01-30 Thu  ●●●○●   4/5 (80%)
    2025-01-29 Wed  ○○●●○   2/5 (40%)
    2025-01-28 Tue  ●●●●●   5/5 (100%)  ★
    
    Legend: ● heeded  ○ skipped  ◐ afk  ◌ pending
```

#### `mu stats`
```
USAGE:
    mu stats [OPTIONS]

OPTIONS:
    --period, -p   week|month|all (default: week)
    --json         Output as JSON

OUTPUT:
    μ stats — this week
    
    Total pings:     32
    Heeded:          21 (66%)
    Skipped:         8  (25%)
    AFK:             2  (6%)
    Expired:         1  (3%)
    
    Avg latency:     2m 18s
    Total pause:     1h 45m
    
    Best hour:       09:00 (82% heed rate)
    Worst hour:      15:00 (43% heed rate)
    
    Best day:        Tuesday (78% heed rate)
    Worst day:       Wednesday (52% heed rate)
```

#### `mu pause`
```
USAGE:
    mu pause [DURATION]

ARGUMENTS:
    DURATION    How long to pause (default: 2h)
                Formats: 30m, 2h, 1d, "until 14:00"

BEHAVIOR:
    Sets pause_until in config, systemd still triggers but
    mu ping exits early when paused

OUTPUT:
    μ paused until 16:30 (2h from now)
```

#### `mu config`
```
USAGE:
    mu config [set KEY VALUE]

KEYS:
    default_pause_duration   Pause duration in seconds (default: 300)
    afk_timeout             Auto-expire timeout in seconds (default: 600)
    sound_enabled           Play sounds (default: true)
    sound_file              Custom sound path (default: system bell)
    notification_timeout    Notification timeout ms, 0=persist (default: 0)

OUTPUT (no args):
    μ config
    
    default_pause_duration = 300    (5m)
    afk_timeout            = 600    (10m)
    sound_enabled          = true
    sound_file             = 
    notification_timeout   = 0      (persist)
    pause_until            =        (not paused)
```

#### `mu ping`
```
USAGE:
    mu ping [OPTIONS]

OPTIONS:
    --force    Ignore time window and pause status

BEHAVIOR:
    1. Check if within 08:00-22:00 (exit if not, unless --force)
    2. Check if paused (exit if yes, unless --force)
    3. Expire any pending pings older than afk_timeout
    4. Create new ping with status=pending
    5. Send desktop notification
    6. Exit (don't wait for response)

EXIT CODES:
    0 - Ping sent
    2 - Outside time window
    3 - Currently paused
```

#### `mu digest`
```
USAGE:
    mu digest [OPTIONS]

OPTIONS:
    --format    text|json|markdown (default: text)
    --period    week|month (default: week)

OUTPUT (text):
    μ Weekly Digest — 2025-W05 (Jan 27 - Feb 2)
    
    OVERVIEW
    ────────
    Pings: 35 │ Heeded: 24 (69%) │ Paused: 2h 0m
    
    TREND
    ─────
    vs last week: +8% heed rate ▲
    
    PATTERNS
    ────────
    • Best performance: mornings (09:00-11:00)
    • Afternoon slump: 14:00-16:00 needs attention
    • Tuesday is your strongest day
    
    STREAKS
    ───────
    Current: 4 days ≥50% heed rate
    Best ever: 12 days (Dec 2024)

OUTPUT (json):
    {
      "period": "2025-W05",
      "total_pings": 35,
      "heeded": 24,
      "skipped": 8,
      "afk": 2,
      "expired": 1,
      "heed_rate": 68.6,
      "total_pause_minutes": 120,
      "avg_latency_seconds": 138,
      "trend_vs_last": 8.2,
      "best_hour": 9,
      "worst_hour": 15,
      "best_day": 2,
      "current_streak": 4,
      "best_streak": 12
    }
```

---

## 4. systemd Integration

### mu.timer

```ini
# ~/.config/systemd/user/mu.timer

[Unit]
Description=mu pause reminder timer (every 90 minutes)
Documentation=man:mu(1)

[Timer]
# Fire every 90 minutes, aligned to clock
# This gives us: 00:00, 01:30, 03:00... 07:30, 09:00, 10:30...
OnCalendar=*-*-* 00/3:00,30:00
# Alternative: fixed schedule within 08:00-22:00
# OnCalendar=*-*-* 08,09:30,11,12:30,14,15:30,17,18:30,20,21:30:00

# Persist across reboots — catch up if machine was off
Persistent=true

# Add 0-5 min randomized delay to feel less robotic
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

### mu.service

```ini
# ~/.config/systemd/user/mu.service

[Unit]
Description=mu pause reminder ping
Documentation=man:mu(1)

# Only run if graphical session is available
After=graphical-session.target
Wants=graphical-session.target

[Service]
Type=oneshot

# The mu ping command handles time window checking internally
ExecStart=/usr/local/bin/mu ping

# Required for desktop notifications
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/%U/bus

# Don't fail the service if ping exits with "outside time window"
SuccessExitStatus=2 3
```

### Time Window Handling

The 08:00-22:00 constraint is handled in `mu ping`, not systemd:

```go
// internal/schedule/window.go

package schedule

import "time"

// DefaultWindow defines operating hours
var DefaultWindow = Window{
    StartHour: 8,  // 08:00
    EndHour:   22, // 22:00 (last ping at 21:30)
}

type Window struct {
    StartHour int
    EndHour   int
}

// IsActive returns true if current time is within the window
func (w Window) IsActive() bool {
    now := time.Now()
    hour := now.Hour()
    return hour >= w.StartHour && hour < w.EndHour
}

// NextPingTime returns when the next ping would occur
func (w Window) NextPingTime() time.Time {
    // Implementation: find next 90-min boundary within window
}
```

### Installation Script

```bash
#!/bin/bash
# bin/install-systemd

set -euo pipefail

SYSTEMD_USER_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/systemd/user"
INSTALL_BIN="${INSTALL_BIN:-/usr/local/bin}"

echo "Installing mu systemd units..."

# Create directory if needed
mkdir -p "$SYSTEMD_USER_DIR"

# Copy unit files
cp systemd/mu.timer "$SYSTEMD_USER_DIR/"
cp systemd/mu.service "$SYSTEMD_USER_DIR/"

# Update binary path in service file
sed -i "s|/usr/local/bin/mu|${INSTALL_BIN}/mu|g" "$SYSTEMD_USER_DIR/mu.service"

# Reload systemd
systemctl --user daemon-reload

# Enable and start timer
systemctl --user enable mu.timer
systemctl --user start mu.timer

echo "✓ mu.timer enabled and started"
echo ""
echo "Status: systemctl --user status mu.timer"
echo "Logs:   journalctl --user -u mu.service -f"
echo "Test:   mu ping --force"
```

### Uninstall Script

```bash
#!/bin/bash
# bin/uninstall-systemd

set -euo pipefail

SYSTEMD_USER_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/systemd/user"

echo "Uninstalling mu systemd units..."

# Stop and disable
systemctl --user stop mu.timer 2>/dev/null || true
systemctl --user disable mu.timer 2>/dev/null || true

# Remove files
rm -f "$SYSTEMD_USER_DIR/mu.timer"
rm -f "$SYSTEMD_USER_DIR/mu.service"

# Reload
systemctl --user daemon-reload

echo "✓ mu systemd units removed"
```

---

## 5. Weekly Digest (HAL Integration)

### Digest Format for HAL

The weekly digest is designed to be consumed by HAL for pattern recognition and gentle coaching.

```markdown
## μ Weekly Digest — 2025-W05

### Summary
- **Period:** Jan 27 – Feb 2, 2025
- **Pings:** 35 total
- **Heeded:** 24 (69%)
- **Total pause time:** 2h 0m

### Response Breakdown
| Response | Count | Rate |
|----------|-------|------|
| Heeded   | 24    | 69%  |
| Skipped  | 8     | 23%  |
| AFK      | 2     | 6%   |
| Expired  | 1     | 3%   |

### Trends
- vs previous week: **+8%** heed rate ▲
- 3-week average: 64%
- Best week ever: 82% (2025-W02)

### Time Patterns
**Best hours:** 09:00-11:00 (82% heed rate)
**Worst hours:** 14:00-16:00 (48% heed rate)
**Best day:** Tuesday
**Worst day:** Wednesday

### Observations
- Morning practice is strong and consistent
- Post-lunch slump (14:00-16:00) shows resistance
- Tuesdays work well — possibly fewer meetings?
- 3 consecutive days with 70%+ this week

### Streak
- **Current:** 4 days with ≥50% heed rate
- **Best ever:** 12 days (Dec 2024)

---
*Generated by mu v1.0.0*
```

### JSON Digest for Programmatic Use

```json
{
  "version": "1.0",
  "generated_at": "2025-02-03T08:00:00Z",
  "period": {
    "type": "week",
    "identifier": "2025-W05",
    "start": "2025-01-27",
    "end": "2025-02-02"
  },
  "summary": {
    "total_pings": 35,
    "responses": {
      "heeded": 24,
      "skipped": 8,
      "afk": 2,
      "expired": 1
    },
    "heed_rate": 68.6,
    "total_pause_minutes": 120,
    "avg_latency_seconds": 138
  },
  "trends": {
    "vs_previous_week": 8.2,
    "three_week_average": 64.1,
    "best_week_rate": 82.3,
    "best_week": "2025-W02"
  },
  "patterns": {
    "by_hour": [
      {"hour": 8, "pings": 5, "heeded": 4, "rate": 80.0},
      {"hour": 9, "pings": 5, "heeded": 4, "rate": 80.0},
      {"hour": 11, "pings": 5, "heeded": 4, "rate": 80.0},
      {"hour": 12, "pings": 5, "heeded": 3, "rate": 60.0},
      {"hour": 14, "pings": 5, "heeded": 2, "rate": 40.0},
      {"hour": 15, "pings": 5, "heeded": 3, "rate": 60.0},
      {"hour": 17, "pings": 5, "heeded": 4, "rate": 80.0}
    ],
    "by_day": [
      {"day": 1, "name": "Mon", "pings": 5, "heeded": 3, "rate": 60.0},
      {"day": 2, "name": "Tue", "pings": 5, "heeded": 4, "rate": 80.0},
      {"day": 3, "name": "Wed", "pings": 5, "heeded": 2, "rate": 40.0},
      {"day": 4, "name": "Thu", "pings": 5, "heeded": 4, "rate": 80.0},
      {"day": 5, "name": "Fri", "pings": 5, "heeded": 3, "rate": 60.0},
      {"day": 6, "name": "Sat", "pings": 5, "heeded": 4, "rate": 80.0},
      {"day": 7, "name": "Sun", "pings": 5, "heeded": 4, "rate": 80.0}
    ],
    "best_hour": 9,
    "worst_hour": 14,
    "best_day": 2,
    "worst_day": 3
  },
  "streaks": {
    "current_days": 4,
    "best_ever_days": 12,
    "best_ever_period": "2024-12"
  },
  "insights": [
    "Morning practice (09:00-11:00) shows strong consistency",
    "Post-lunch period (14:00-16:00) has lowest engagement",
    "Tuesday performance is notably higher than average",
    "3 consecutive days with 70%+ heed rate this week"
  ]
}
```

### HAL Integration Hook

```bash
# Cron entry for weekly digest (Sundays at 20:00)
# 0 20 * * 0 /usr/local/bin/mu digest --format=markdown >> ~/clawd/memory/mu-digest.md

# Or via systemd timer for HAL to consume
# mu-digest.timer fires weekly, writes to a known location
```

---

## 6. Interface Contracts

### Database Interface

```go
// internal/db/db.go

package db

import (
    "context"
    "time"
)

// Ping represents a single reminder event
type Ping struct {
    ID             int64
    TriggeredAt    time.Time
    Response       string  // pending, heeded, skip, afk, expired
    RespondedAt    *time.Time
    LatencySeconds *int
    PauseDuration  *int
    Note           *string
    DayOfWeek      int
    HourOfDay      int
}

// DailyStats represents aggregated daily statistics
type DailyStats struct {
    Date           string
    TotalPings     int
    Heeded         int
    Skipped        int
    AFK            int
    Expired        int
    Pending        int
    HeedRate       float64
    AvgLatency     *int
    TotalPauseTime int
}

// Store defines the database operations
type Store interface {
    // Initialize creates tables if they don't exist
    Initialize(ctx context.Context) error
    
    // Ping operations
    CreatePing(ctx context.Context) (*Ping, error)
    GetPendingPing(ctx context.Context, maxAge time.Duration) (*Ping, error)
    UpdatePingResponse(ctx context.Context, id int64, response string, duration *int, note *string) error
    ExpireOldPings(ctx context.Context, timeout time.Duration) (int, error)
    
    // Query operations
    GetPingsForDate(ctx context.Context, date time.Time) ([]Ping, error)
    GetPingsInRange(ctx context.Context, start, end time.Time) ([]Ping, error)
    GetDailyStats(ctx context.Context, days int) ([]DailyStats, error)
    GetWeeklyStats(ctx context.Context, weeks int) ([]WeeklyStats, error)
    
    // Config operations
    GetConfig(ctx context.Context, key string) (string, error)
    SetConfig(ctx context.Context, key, value string) error
    
    // Close releases database resources
    Close() error
}

// NewSQLiteStore creates a new SQLite-backed store
func NewSQLiteStore(path string) (Store, error)
```

### Notification Interface

```go
// internal/notify/notify.go

package notify

// Notifier sends desktop notifications
type Notifier interface {
    Send(title, body string) error
    SendUrgent(title, body string) error
    PlaySound(soundFile string) error
}

// LinuxNotifier implements Notifier for Linux/freedesktop
type LinuxNotifier struct {
    SoundEnabled bool
    DefaultSound string
}

func NewLinuxNotifier() *LinuxNotifier
func (n *LinuxNotifier) Send(title, body string) error
func (n *LinuxNotifier) SendUrgent(title, body string) error  
func (n *LinuxNotifier) PlaySound(soundFile string) error
```

### CLI Command Interface

```go
// cmd/mu/main.go

package main

// Commands are registered with cobra.Command
// Each command follows this pattern:

type CommandContext struct {
    DB       db.Store
    Notifier notify.Notifier
    Config   *config.Config
    Out      io.Writer
    Err      io.Writer
}

// Command functions receive context and return error
type CommandFunc func(ctx *CommandContext, args []string) error
```

---

## 7. File Locations

```
~/.local/share/mu/
├── mu.db                    # SQLite database

~/.config/mu/
├── config.toml              # User configuration (optional)

~/.config/systemd/user/
├── mu.timer                 # systemd timer unit
├── mu.service               # systemd service unit

/usr/local/bin/
├── mu                       # Binary (or ~/.local/bin/mu)

~/.local/share/mu/sounds/    # Custom sounds (optional)
├── ping.oga
```

---

## 8. Implementation Phases

### Phase 1: Database Layer
- [ ] SQLite schema creation
- [ ] Store interface implementation
- [ ] Basic CRUD operations
- [ ] Config get/set

### Phase 2: Notification System
- [ ] notify-send wrapper
- [ ] Sound playback (paplay)
- [ ] Message rotation

### Phase 3: Core CLI
- [ ] `mu ping` (create pending)
- [ ] `mu` / `mu heed` (respond heeded)
- [ ] `mu skip` (respond skip)
- [ ] `mu afk` (respond afk)
- [ ] `mu status` (today view)

### Phase 4: Extended CLI
- [ ] `mu history`
- [ ] `mu stats`
- [ ] `mu pause` / `mu resume`
- [ ] `mu config`
- [ ] `mu digest`

### Phase 5: systemd Integration
- [ ] Timer and service units
- [ ] Install/uninstall scripts
- [ ] Time window handling

### Phase 6: Polish
- [ ] Man page
- [ ] Shell completions
- [ ] HAL digest integration

---

## 9. Design Decisions

### Why SQLite?
- Zero infrastructure
- Single file backup
- Good enough for years of data
- Pure Go driver available (no CGO)

### Why 90 minutes?
- Long enough to enter flow
- Short enough to maintain awareness
- ~10 pings per workday (08:00-22:00)

### Why systemd timer, not cron?
- Better logging (journalctl)
- Persistent timers (catch up after sleep)
- User-level units (no root needed)
- RandomizedDelaySec for natural feel

### Why handle time window in code?
- More flexible than systemd calendar syntax
- Can be overridden with `--force`
- Easier to test and modify
- Config-driven in future

---

*Document version: 2.0*
*Last updated: 2025-02-03*
