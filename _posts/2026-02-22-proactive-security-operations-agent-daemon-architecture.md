---
layout: post
title: "Building a Proactive Security Operations Agent: Daemon Architecture, Scheduled Monitoring, and Rate-Limited Alerting"
date: 2026-02-22
categories: [homelab, security-automation, ai-security]
tags: [python, daemon, apscheduler, telegram, sqlite, automation, monitoring, windows]
excerpt: "APScheduler-based Python daemon with 10 scheduled monitoring modules, 6 rate-limited heartbeat checks, Telegram push notification dispatch, and platform-specific Windows process management."
---

APScheduler-based Python daemon with 10 scheduled monitoring modules, 6 rate-limited heartbeat checks, Telegram push notification dispatch, and platform-specific Windows process management.

<!--more-->

## Overview

The daemon manages its own lifecycle (PID tracking, orphan detection, graceful shutdown), persists state across 12 SQLite tables, and exposes 22 API endpoints for external integration. The implementation spans 5,274 lines across 23 files with 104 new tests against an existing 501-test suite (605 total, zero regressions).

## Daemon Framework

The daemon runs on APScheduler 3.x with a BlockingScheduler. Modules register as callables with cron or interval triggers. A health heartbeat writes to SQLite every 60 seconds.

```python
IntervalTrigger(seconds=60)            # Heartbeat
CronTrigger(hour=2, minute=0)          # Nightly archive
CronTrigger(day_of_week="sun", hour=3) # Weekly sync
```

### Lifecycle Management

On startup, the daemon reads its PID file, checks for orphaned processes, and aborts if an instance is already running. Cleanup handlers are registered through both `atexit` and (on Windows) `SetConsoleCtrlHandler` via ctypes to ensure the PID file and database state are cleaned up on normal exit, SIGTERM, and console close events.

## Heartbeat Notification System

Checks run on independent schedules. Each evaluates a condition against live data and dispatches a Telegram notification when the threshold is exceeded:

| Check | Condition | Cooldown |
|-------|-----------|----------|
| Stale applications | Job applications in "applied" status for 7+ days | 24 hours |
| Session crash detector | Active session with no activity for 30+ minutes | 1 hour |
| Goal staleness | Goals with no progress update in 14+ days | 7 days |

### Rate-Limited Dispatch

Before dispatching, each check queries the `heartbeat_log` table for the most recent notification of the same type. If the elapsed time is within the cooldown window, dispatch is suppressed.

```
Check: stale_applications
  Last notified: 2026-02-22 10:15:00
  Cooldown: 24 hours
  Current time: 2026-02-22 14:30:00
  Result: SUPPRESSED (4h 15m into 24h cooldown)
```

Notification delivery uses the Telegram Bot API with per-check rate limiting stored in SQLite. The cooldown values are configurable per check type.

## Windows Process Management: App Container PID Detection Failure

`os.kill(pid, 0)` silently misreports process status on Windows Store Python. The Microsoft Store distribution runs inside an app container that restricts cross-process signal operations, causing `PermissionError` to be raised for alive processes. The error was caught in a generic `except OSError` block, making alive processes appear dead.

### Impact

The PID check returned "not running" for processes that were actively writing heartbeats to the database. The launcher spawned additional instances on each invocation, resulting in 8 orphaned daemon processes.

### Resolution

PID detection was replaced with a `tasklist`-based check that operates outside the app container's restrictions:

```python
def _is_pid_alive(pid: int) -> bool:
    if sys.platform == "win32":
        result = subprocess.run(
            ["tasklist", "/FI", f"PID eq {pid}", "/NH", "/FO", "CSV"],
            capture_output=True, text=True, timeout=5,
        )
        return str(pid) in result.stdout
    else:
        os.kill(pid, 0)
        return True
```

Background process launch uses Windows Task Scheduler as the primary mechanism. Processes created with `DETACHED_PROCESS` or `CREATE_NEW_PROCESS_GROUP` creation flags do not reliably survive console window closure; Task Scheduler-launched processes persist independently of any console session.

## Database Schema

Twelve tables were added across five additive migrations (versions 7 through 11). All use `CREATE TABLE IF NOT EXISTS` with no destructive operations. WAL mode handles concurrent access between the daemon and the main application.

| Migration | Tables |
|-----------|--------|
| 7 | daemon_state |
| 8 | conversation_archive_runs, conversation_archive_sessions |
| 9 | life_domains, life_goals, life_sub_goals, life_goal_dependencies, life_goal_metrics |
| 10 | heartbeat_log |
| 11 | discovered_events, onboarding_state, personality_config |

## Conversation Archive

A nightly job discovers JSONL conversation files, parses them into structured records (user prompts, assistant responses, tool calls), generates summaries, and archives to persistent storage. The pipeline is idempotent: archived session IDs are tracked in `conversation_archive_sessions` and skipped on subsequent runs. Validation against 22 conversation files confirmed successful extraction of 110 turns and 191 tool calls from a single session.

## Results

| Metric | Value |
|--------|-------|
| New code | 5,274 lines / 23 files |
| Database tables | 12 new (migrations 7-11) |
| API endpoints | 22 new |
| Tests | 104 new / 605 total / 0 failures |
| Daemon modules | 10 scheduled |
| Heartbeat checks | 6 with rate limiting |
| Notification delivery | Telegram push, confirmed end-to-end |
