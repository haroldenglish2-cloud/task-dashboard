# Strategy scanner — run instructions

A Routine fires a fresh Claude session on a schedule and points it here. Each firing
follows this file exactly. The whole point is that most runs end **silently**.

## Schedule

Nine runs per weekday, every 30 minutes from 10:00 to 14:00 New York time.

| Routine | Cron (UTC) | New York slots |
| --- | --- | --- |
| top-of-hour | `2 14-18 * * 1-5` | 10:02, 11:02, 12:02, 13:02, 14:02 |
| half-past | `32 14-17 * * 1-5` | 10:32, 11:32, 12:32, 13:32 |

Two Routines, not one, because the scheduler's minimum interval is hourly. Each is
hourly on its own; interleaved they give the 30-minute cadence.

Cron is evaluated in UTC. The offsets above assume EDT (UTC-4). **When New York
returns to EST (UTC-5) in November, both hour ranges must shift one hour later**
(`2 15-19` and `32 15-18`) or the scanner will run 09:00–13:00 local instead.

## Each run

1. Work out today's New York date.
2. Read `scanner/state.json` from this branch. `FIRST_RUN` is true when
   `state.date` is not today's New York date.
3. Run the scan: call the Co-Invest `market_picks` tool. Every candidate it
   returns is a hit. Key each hit as `SYMBOL:DIRECTION`, uppercased —
   `NVDA:LONG`. The key is what makes a hit "already reported", so it must not
   embed prices, timestamps, or scores.
4. Notify, per these rules and no others:
   - **`FIRST_RUN`** — send one `PushNotification` (status `proactive`)
     summarizing the scan, *including when it found nothing*. This is the daily
     "scanner is alive" ping.
   - **Otherwise** — let `NEW` be the hits whose keys are absent from
     `state.seen`. If `NEW` is non-empty, send one `PushNotification` naming
     only those. If `NEW` is empty, send nothing at all: no notification, no
     summary, no closing remark. Update state and end the turn.
5. Write `scanner/state.json` back and push it to this branch:
   - `date` — today's New York date.
   - `seen` — on `FIRST_RUN`, exactly this run's keys (the previous day's are
     discarded). Otherwise the existing keys plus the `NEW` ones.
   - `runs` — increment, or reset to 1 on `FIRST_RUN`.

Notifications go to a phone: under 200 characters, one line, no markdown, symbols
first. Never ask a question — nobody is watching the session. Never start
unrelated work, and never touch files outside `scanner/`.

## State lives in git on purpose

Each firing gets a brand-new container, so a scratchpad file would not survive
between runs — every run would look like the first run of the day and ping. The
state file is committed to this branch instead, which is what keeps runs 2-9
quiet. A run that cannot read the state file should treat that as `FIRST_RUN`
rather than guessing.

If two runs ever race on the push, rebase onto the remote and re-apply the
`seen` union. Losing a `seen` entry costs a duplicate ping, nothing worse.

## Changing the scan

Step 3 is the only part that defines *what* is scanned. To screen a fixed
watchlist instead, replace it with the symbols and the condition — the schedule
and the quiet rules are independent of it and need no changes.
