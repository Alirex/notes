---
summary: "Measuring a desktop app's memory and start time on Linux without touching the user's desktop: PSS over the process tree, idle and first frame, load gate, 5+ repeats with spread, budgets only next to a measured number"
group: quality
tags: [performance, gui, desktop, tooling]
applies-to: [desktop, rust, all]
reviewed: 2026-11-15
status: draft
---

# Measuring a desktop app's memory and start time

How to get a memory and start-time figure for a GUI app that a decision (stack choice, migration, budget) can rest on,
on a machine where a person is working. Companion to [gui-stack-choice.md](https://github.com/evocaslab/hq/blob/main/practices/gui-stack-choice.md) (what to compare) and
[profiling.md](profiling.md) (where time goes).

## Memory: PSS over the whole process tree

- Read `/proc/<pid>/smaps_rollup` for every process of the app and sum it. The file has the fields of `smaps`, each
  summed over all mappings, and is cheaper to read.
- Report **PSS** as the main figure. PSS divides each shared page by the number of processes that map it; RSS counts a
  shared page fully in every process, so a multi-process app is double-counted.
- Count the whole tree. A webview shell (Tauri, Electron) is the app plus a network process plus a web process; measuring
  only the parent hides most of the memory. Find children by parent pid (and the cgroup or session when helpers detach).
- Show RSS next to PSS, labelled as the comparison figure. Example: a webview shell read 592 MB RSS against 390 MB PSS.
- Say when and how the sample was taken: a fixed time after the first frame, idle, fresh config/data folders for each
  start (no cached state from an earlier run).

## Start time: window mapped and first frame

- "Window mapped" and "first painted frame" differ. A window can exist long before it shows content.
- Poll the window image (X11 `xwd`/`import`, or a portal screenshot of your own window only) until its pixel variance
  passes a threshold; record that time from process start. State that the poll step adds noise, so the number is
  indicative. Confirm once by eye or with a screenshot that the frame is the real window content, not a splash.

## Do not touch the user's desktop

- Never inject synthetic input (XTEST, `xdotool`, a virtual keyboard) into the person's real session. Under GNOME it
  raises a Remote Desktop permission prompt and moves their focus.
- Measure what needs no input: idle state and first frame. Whatever needs a click (memory during a run, a dialog's cost)
  goes in a **Not measured** list with the reason, and any estimate is marked as inference. Options that need setup
  (a nested X server such as Xvfb, a WebDriver, a start flag or env hook that triggers the action) are backlog items,
  not reasons to take over the desktop.

## Repeats, load and spread

- Gate each repeat on machine load (the 1-minute load average below a stated limit, for example the CPU count divided
  by 2); record the load of every run. If the machine never gets quiet, relax the limit once, say so, and keep the values.
- At least 5 valid runs per variant, alternated (A, B, A, B, ...) so drift hits both. Report median and min-max, never one run.
- Memory is steadier than time under load, so a memory ranking can survive a noisy gate while times carry more noise.
  Say which is which.
- Look for **bimodal** results: one binary showed 45 MB or 90 MB PSS depending on GPU-driver mappings. Report both modes
  and what the ratio would be against each, instead of hiding the spread in a median.

## Budgets need the measured number

- A rule of thumb ("a desktop tool stays under 250 MB PSS") is a decision input only with the measured figure next to
  it, the machine class, and the date. Quote "390 MB measured against a 250 MB budget", not "too heavy".
- Separate measured facts from judgment in the result table (a Status column: measured, fact, judgment). Testability or
  "feels light" are judgment even when the memory is measured.

## Result page shape

1. Method: build, machine, folders, repeats, sampling point.
2. Table: median (min-max) per quantity, processes, binary size, ratio.
3. Caveats: bimodality, load, poll accuracy.
4. Not measured, with reasons.
5. Recommendation tied to the measured numbers.

Example: a Rust game-model tool compared its egui GUI with a Tauri shell this way (idle PSS 93 MB against 390 MB, first
frame 0.13 s against 0.94 s) and listed the in-run memory as unmeasured.

## Sources

- `/proc/<pid>/smaps_rollup`, Rss, Pss ("each page divided by the number of processes sharing it"; 1000 own plus 1000
  shared with one other process is a PSS of 1500), Private and Shared:
  <https://www.kernel.org/doc/html/latest/filesystems/proc.html>
- Per-page mapping data behind these figures (it does not define PSS): <https://www.kernel.org/doc/html/latest/admin-guide/mm/pagemap.html>
- Observed in a measurement of a Tauri + Vue app against an egui app (process-tree PSS, pixel-variance first-frame poll,
  synthetic input refused on the real desktop, load gate, bimodal PSS); the load gate, repeat count and result shape have
  no external source.
