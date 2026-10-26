# launch-control

Build me a Claude Code mod called launch-control. Use your plugin-authoring skill and read the mods docs first (interface, events, mods API: timers, processes, status, toast, prompt.submit), so you know exactly what a pane and a band can draw and how background work runs.

## GOAL

After every production deploy, watch the live app for a set window and wake Claude only if something breaks. It should look like a mission-control launch console, striking enough to show on camera.

## STEP 1: INSPECT BEFORE BUILDING

Tell me what you find before writing code:

* How this project deploys to production: the vercel CLI, a git push to main through Vercel's git integration, or both
* Whether the vercel CLI is installed and logged in, and whether a Vercel MCP server is connected
* The best available way to get the newest production deployment, its status, and its runtime logs (check vercel --help for current commands; don't guess flags)

## STEP 2: TRIGGER (IGNITION)

Hook tool.call for Bash and start a mission on either a production vercel deploy or a git push to the production branch. Let the call run untouched. For git pushes, find the deployment Vercel builds for that commit.

## STEP 3: MISSION STAGES

Track the deployment through stages and show them on the console:

PRE-FLIGHT (building) → LAUNCH (deploying) → ASCENT (live, watch window running) → ORBIT (window closed clean)
or → ABORT (build failed or anomaly detected)

## STEP 4: TELEMETRY

During ASCENT, poll logs on a timer (not inside a hook) and track:

* Request count, error count, error rate
* Distinct error signatures, flagging ones not seen before this deploy
* Watch-window countdown (default 15 minutes, set in plugin config)

## THE CONSOLE PANE (/launch toggles it, registered as an immediate command)

* Header: MISSION ID (short deploy ID), commit message, branch, deployment URL
* Stage track: the five stages as indicator lights, with the current one lit and the completed ones checked
* Mission clock: T+mm:ss / window length, with a progress bar
* GO / NO-GO board: build, deploy, error rate, new errors, each with a status light
* Telemetry: an error-rate sparkline (▁▂▃▅▇) plus live counters
* Log feed: the last ~8 log lines, scrolling, with errors highlighted
* Style: monospace caps labels, phosphor-console feel (green/amber on dark), but readable on light themes too. Never rely on color alone: lights also use symbols (● ○ ✓ ✕).

## THE BAND (above the prompt, only during a mission)

One line, like: 🚀 MISSION a3f9 · ASCENT · T+04:12/15:00 · req 1,204 · err 0 · ALL GO

## ANOMALY HANDLING

When a new error signature appears, or the error rate crosses a threshold (plugin config):

* Board goes NO-GO, the band switches to an alert style, and a toast says ANOMALY DETECTED
* Start one Claude turn with prompt.submit containing the stack trace, deployment URL, commit, and the request path that failed. Ask Claude to investigate and propose a fix.
* Dedupe by error signature, so one bug triggers one turn, not twenty
* Don't await prompt.submit inside a handler while Claude is working

## ROLLBACK (ABORT)

Rollback never runs automatically. If Claude proposes a vercel rollback, hold the call and show which deployment it would roll back to, plus Proceed and Cancel buttons.

## MISSION COMPLETE

When the window ends clean: ORBIT ACHIEVED toast, a transcript log line with the summary, and the band clears after a few seconds.

## WHERE THE UI CAN'T DRAW

In the VS Code panel, claude -p, or cloud sessions, use status lines and transcript log lines instead of the pane.

## DEMO MODE

Add /launch demo, which simulates a full mission: build, deploy, a clean ascent, then a new error at T+03:00, the anomaly turn, and a held rollback. I need this to record without breaking a real deploy. /launch abort ends any mission.

## BUILD ORDER

1. Show me a text mockup of the console pane and the band first, and wait for my OK
2. Build it, with tests for trigger detection, stage transitions, error dedupe, and thresholds
3. Run claude plugin validate and show me the hooks and calls
4. Load it with --plugin-dir, run the demo, then explain how to install it permanently
