# mu 無

Non-doing reminder. Technology supporting stillness.

## What

A micro app that pings every 90 minutes inviting a 5-minute pause for being/non-doing. Regular invitations to stop, regardless of flow state.

## Why

Contemplative practice integrated into the workday. Not productivity — presence.

## Usage

```bash
mu heeded    # Acknowledged the pause
mu skip      # Not now
mu afk       # Away from keyboard
mu status    # Current streak/stats
mu report    # Weekly digest
```

## Architecture

- **Daemon:** systemd timer (08:00-22:00, every 90 min)
- **Notification:** Desktop notification via notify-send
- **Storage:** SQLite tracking pings and responses
- **Digest:** Weekly summary to HAL

## Status

Scaffolded. Implementation in progress.

## License

MIT
