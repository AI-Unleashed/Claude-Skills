# edit-heatmap

Build me a Claude Code mod called edit-heatmap. Use your plugin-authoring skill and read the mods interface docs first, so you know exactly which elements, colors, and layout options a pane and a band support in the terminal and in the Desktop app.

## GOAL

A live, visually striking view of where Claude is editing this session, so thrashing on one file is obvious at a glance. It should look good enough to show on camera.

## DATA TO TRACK

Hook tool.call for Edit, Write, MultiEdit, and NotebookEdit. Let each call run untouched, but record per file:

* Edit count and timestamps
* Lines added and removed
* Reverts: an edit whose new text matches text a previous edit removed from the same file. That's real thrashing, not just repeat edits.

Keep all state in memory for this session only.

## THE PANE (/heatmap toggles it, registered as an immediate command so it works mid-task)

* Header: files touched, total edits, lines changed, and a churn score
* One row per file, hottest first, grouped under collapsible folder headings
* Each row shows: a heat gradient from cool to hot based on edit count; a sparkline of edits over time using block characters (▁▂▃▅▇) so bursts are visible; a small +/− lines bar; and a ↺ marker with a count if the file has reverts
* The row for the file just edited flashes, then fades back to its normal color over a few seconds (use timers and redraws)
* Never rely on color alone: heat also shows as a symbol or bar length, so it reads on any theme and for colorblind viewers
* Check that it looks right in both dark and light terminal themes

## THE BAND (always on, above the prompt)

One compact line with the three hottest files as mini heat chips, like: 🔥 auth.ts ×8 ↺2 · api.ts ×4 · db.ts ×2. Hide it until the first edit.

## THRASH ALERT

When a file hits 5 edits within 10 minutes, or 2 reverts, its row turns to an alert style and a toast names the file. Make the thresholds plugin config.

## WHERE THE UI CAN'T DRAW

In the VS Code panel, claude -p, or cloud sessions, /heatmap prints a plain-text table instead.

## DEMO MODE

Add /heatmap demo, which feeds in a realistic fake session (a dozen files, one obviously thrashing with reverts) so I can check visuals and record without waiting on real work. /heatmap reset clears it.

## BUILD ORDER

1. Before writing code, show me a text mockup of the pane and the band, and wait for my OK
2. Build it, with tests for counting, revert detection, and thresholds
3. Run claude plugin validate and show me the hooks and calls
4. Load it with --plugin-dir, run the demo, and tell me how to install it permanently
