# NEPI PROMPT GENERATION MODE

When a user message begins with "prompt:", treat everything after that word as a prompt generation request. Do not enter prompt generation mode for any other input — respond to all other messages normally as Claude Code.

## WORKSPACE ROOT

This file must work for anyone who has a nepi_engine_ws checkout, on any machine and any account — native Linux, WSL, or a container. Never assume a specific home directory, user name, or absolute prefix such as /home/production. There is no fixed location for the workspace.

Resolve the workspace root once, at the start of prompt generation, and use it for everything afterwards:

1. Run `git rev-parse --show-toplevel` from the current working directory. If the result is a directory containing `src/nepi_claude`, that is the workspace root.
2. If that fails or the result does not contain `src/nepi_claude`, walk up from the current working directory looking for a directory that contains `src/nepi_claude`.
3. If neither finds it, ask the user: "What is the full path to your nepi_engine_ws checkout?" Do not search the filesystem for it.

Below, <WS> stands for the resolved workspace root. Every path written into a generated prompt is relative to <WS> (for example `src/nepi_claude/NEPI-LORE.md`, `src/nepi_engine/nepi_api/...`), never absolute. Absolute paths belong only in shell commands where they are unavoidable, and there they are built from <WS>, not typed literally.

## CLARIFYING QUESTIONS

When "prompt:" is detected, check whether you have enough information to define a clear MVP boundary. If the task description is missing any of the following, ask one specific question to fill the gap before drafting:

- Which submodule or repo path is affected (if not obvious from the task)
- Whether this is Creation, Documentation, Development Support, Audit, or Release
- For Creation tasks: what the component does, what hardware or interface it connects to, and whether a spec exists

Only ask one clarifying question at a time. If the task is clear enough to draft, draft it without asking.

## BOUNDED DISCOVERY — ASK, DO NOT HUNT

Prompt generation is a drafting step, not an investigation. Locating the files a prompt will target must be fast. Apply a hard search budget:

- At most three targeted lookups per unknown file, directory, or symbol — a Glob against the expected package layout, a Grep scoped to one submodule, or a single `ls` of a known directory.
- Never run a recursive search across the whole workspace, across all submodules, or across the user's home directory.
- Never launch subagents to find a path.
- Do not read files to confirm a path exists; the path existing is enough.

If those lookups do not find it, stop searching and ask the user one specific question naming what is missing and the best guess at where it should be — for example: "I could not find the pan/tilt app package. Is it at src/nepi_apps/nepi_app_pan_tilt_auto, or somewhere else in your checkout?" Ask this before drafting, not after. One unfound thing is one question; if several are missing, ask about them together in a single message rather than one round trip each.

A prompt may legitimately reference a path that does not exist yet — that is normal for Creation tasks. Only ask when the path is supposed to already exist and cannot be found.

When you have enough information, produce the prompt as a plain text code block following these rules:

RULE 1: Plain text only inside the code block. No markdown formatting, no bold, no # headers, no bullet points.

RULE 2: Begin with the framework file read sequence, adapted to the task. All paths are relative to the workspace root, which is the directory the run starts in:
  Read src/nepi_claude/NEPI-LORE.md in full.
  Read src/nepi_claude/NEPI-CODEX.md in full.
  Read the top-level CLAUDE.md in full.
  Read src/<submodule>/CLAUDE.md in full.
  Do not take any action until you have read all documents.

RULE 3: State the target repo and exact path being modified, as a path relative to the workspace root. Never write an absolute path, a home directory, or a user name into a generated prompt — the person running it may have the workspace anywhere, including a WSL home directory nested several levels deep. If the prompt needs the run to know the root, tell it to resolve it with `git rev-parse --show-toplevel` rather than stating a location.

RULE 3A: Carry the search budget into the generated prompt. Instruct the run to locate files by their stated relative paths and, if something expected is missing, to raise it as the RULE 4A NEEDS INPUT question rather than searching the workspace for it.

RULE 4: Sequence the work as numbered steps. At the start of each step, print a single-line banner exactly in the form ===== STEP n/N: <short title> ===== so progress is greppable in the run log. At the end of each step, print a brief report of what that step found or changed, then continue immediately into the next step within the same run. The report is a checkpoint, not a stopping point. Never instruct Claude Code to stop, halt, pause, wait, or "stop before Step N" between steps — in a headless run that phrasing is read literally and terminates the run after the first step. The run proceeds through all N steps in a single pass and stops only at the end, or at the one clarification pause defined in RULE 4A.

RULE 4A: The run may pause before completion in exactly one situation. If, after finishing the first step (the Explore/read step where all framework documents and target files are read), Claude Code has a genuine blocking question it cannot resolve from the codebase, the framework documents, or sensible defaults, it prints a single line beginning NEEDS INPUT: followed by one specific question, and stops there. It must not manufacture a question in order to pause — if the first step resolves everything, it runs straight through to the end. All blocking questions are raised at the end of the first step or not at all; later steps never pause for input. When the user answers, the run resumes from where it paused (same session) and continues to completion.

RULE 5: For Creation and Development Support tasks, include an explicit Explore step: read all relevant existing files in the target path and report what you find before writing any code.

RULE 6: Instruct Claude Code to follow existing patterns in the target submodule rather than inventing new ones.

RULE 7: State the Python naming convention rules explicitly for any task involving Python:
  Public methods: snake_case, receives a docstring.
  Private methods: camelCase (with or without leading underscore), no docstring.
  ROS callbacks: camelCase with Cb suffix, audit external call sites before renaming.

RULE 8: Instruct Claude Code to surface, not apply, any changes that would update framework files (NEPI-LORE.md, NEPI-CODEX.md, NEPI-FORGE.md, top-level CLAUDE.md). Those are reported in the final output only.

RULE 9: Define a clear stopping point. No open-ended scope. The only mid-run halt permitted is the single NEEDS INPUT clarification pause defined in RULE 4A.

Before presenting the prompt, run the following quality check silently and fix any failures before output:
1. Does it begin with the four-file read sequence?
2. Is the target repo and path specified?
3. Is the sequence of steps unambiguous with a report between each?
4. For creation prompts: is there an explicit explore step before any code is written?
5. Does it reference existing patterns rather than inventing new ones?
6. Are naming convention rules stated for Python tasks?
7. Is the scope bounded with a defined stopping point?
8. Does it instruct Claude Code to surface rather than apply framework updates?
9. Is it plain text with no markdown formatting inside the code block?
10. Does the prompt run continuously through all steps with no "stop/halt/wait before Step N" phrasing, pausing only for a single NEEDS INPUT question after the first step (RULE 4A)?
11. Is every path in the prompt relative to the workspace root, with no absolute path, no home directory, and no user name anywhere in the text?
12. Was every path in the prompt either confirmed within the three-lookup budget, expected not to exist yet (Creation), or asked about before drafting — with no workspace-wide search performed?

After presenting the prompt, offer the user three options:
  Option A: Request adjustments — describe what to change and a revised prompt will be produced.
  Option B: "runfull" — execute fully automated with no permission prompts of any kind.
  Option C: "runwatch" — execute normally, pausing for permission prompts so the user can approve each action.

CONCURRENT RUNS: Multiple prompts may run at the same time. Each run gets its own set of state files (.txt, .sid, .log) identified by a slot number, so runs never clobber each other. Before launching in either run mode, prune dead slots and allocate a free one with a single Bash tool call:
    for f in /tmp/nepi_prompt_run*.sid; do [ -e "$f" ] || continue; b="${f%.sid}"; s=$(cat "$f" 2>/dev/null); [ -n "$s" ] && pgrep -f "$s" >/dev/null && continue; tail -n 5 "$b.log" 2>/dev/null | grep -q '^NEEDS INPUT:' && continue; rm -f "$b.txt" "$b.sid" "$b.log"; done; RUN=/tmp/nepi_prompt_run; if [ -e "$RUN.sid" ]; then i=2; while [ -e "${RUN}_$i.sid" ]; do i=$((i+1)); done; RUN="${RUN}_$i"; fi; { uuidgen 2>/dev/null || cat /proc/sys/kernel/random/uuid; } > "$RUN.sid"; rm -f "$RUN.txt" "$RUN.log"; echo "$RUN"
The prune step deletes the state files of any slot whose session id has no live process, unless that slot's log ends in a NEEDS INPUT: line — those slots belong to runs paused for clarification and must survive. Allocation then reserves the slot with its .sid file (which also holds the session id for this run) and removes any leftover .txt and .log for that slot, so the Write tool always creates the prompt file fresh. Never leave a placeholder .txt behind for Write to overwrite: Write refuses to overwrite a file that has not been read in the current conversation, and that is what causes a "Write failed" on the prompt file.

The first run uses /tmp/nepi_prompt_run; the next free slot is /tmp/nepi_prompt_run_2, then _3, and so on. The command prints the concrete base path for this run. Below, <RUN> stands for that concrete base path (for example /tmp/nepi_prompt_run_2). Remember it for the rest of the conversation, and substitute it literally into every subsequent command for this run — shell variables do not persist between separate Bash tool calls, so do not rely on $RUN after the allocation command. To list all active runs at any time: ls /tmp/nepi_prompt_run*.sid

If the user says "runfull" (or any clear variant like "run full", "run it full", "full run"):

  Step 1: Allocate a slot as above. The allocation removed any leftover .txt for that slot, so write the exact prompt text (the contents of the code block, no surrounding markdown) to this run's <RUN>.txt (for example /tmp/nepi_prompt_run_2.txt) with the Write tool as a new file. Do not Read it first — nothing is there.

  Step 2: The session id was already minted into <RUN>.sid by the allocation command, so the run can be resumed later. Launch it in the background using the Bash tool with run_in_background set to true, so it is not bound by the 10-minute foreground timeout. Redirect all output to this run's log file (substitute the concrete <RUN> path):
    "$CLAUDE_CODE_EXECPATH" --dangerously-skip-permissions --session-id "$(cat <RUN>.sid)" -p "$(cat <RUN>.txt)" > <RUN>.log 2>&1
  All file edits and bash commands proceed without any permission prompts. Do not wait synchronously — the background process keeps running across turns and re-invokes you when it exits.

  Step 3: When the background subprocess exits, read this run's <RUN>.log. If the run paused for clarification (its final output is a line beginning NEEDS INPUT:), relay that question to the user verbatim and do NOT report completion and do NOT delete the slot files — the run is not done. When the user answers, resume the same session with their answer, appending to the same log:
    "$CLAUDE_CODE_EXECPATH" --dangerously-skip-permissions --resume "$(cat <RUN>.sid)" -p "<the user's answer>" >> <RUN>.log 2>&1
  Re-check for a trailing NEEDS INPUT: line after each resume; keep resuming until the run finishes without one. Once the run finishes without a NEEDS INPUT: line, report a one-sentence summary of what was completed or any errors encountered, then free the slot by deleting this run's state files:
    rm -f <RUN>.txt <RUN>.sid <RUN>.log
  Free the slot on every terminal outcome, not just success — a run that errored out, was killed, or produced nothing still gets its state files deleted. The only case that keeps them is a run paused at NEEDS INPUT:. Deleting the files is the last action of the run; do it in the same turn you report the summary, so no stale .txt is left to collide with the next prompt.

  To watch progress live while the background run is going, tell the user they can run in a terminal (using this run's concrete log path):
    tail -f <RUN>.log | grep --line-buffered '===== STEP'
  This shows each step banner the instant it begins (drop the grep to see full output). The user can also ask "what step?" at any time and you will read that run's <RUN>.log and report the current step; if several runs are active, report the step for each active run.

If the user says "runwatch" (or any clear variant like "run watch", "run it watch", "watch run"):

  Step 1: Allocate a slot as above. The allocation removed any leftover .txt for that slot, so write the exact prompt text (the contents of the code block, no surrounding markdown) to this run's <RUN>.txt with the Write tool as a new file. Do not Read it first — nothing is there.

  Step 2: The session id is already in <RUN>.sid from the allocation command. Execute via Bash with a timeout of 600000ms (substitute the concrete <RUN> path):
    "$CLAUDE_CODE_EXECPATH" --session-id "$(cat <RUN>.sid)" -p "$(cat <RUN>.txt)"
  Permission prompts will appear in the terminal as normal. The user approves or denies each action. This runs in the foreground, so the 10-minute maximum applies — if the run exceeds it, the subprocess is killed (exit 143) and returns no final report. For long automated runs that do not need per-action approval, use runfull instead.

  Step 3: After the subprocess finishes, check its output. If it ends in a line beginning NEEDS INPUT:, relay that question to the user and do NOT report completion and do NOT delete the slot files; when the user answers, resume the same session:
    "$CLAUDE_CODE_EXECPATH" --resume "$(cat <RUN>.sid)" -p "<the user's answer>"
  and repeat until it finishes without a NEEDS INPUT: line. Once it finishes without a NEEDS INPUT: line, report a one-sentence summary of what was completed or any errors encountered, then free the slot by deleting this run's state files:
    rm -f <RUN>.txt <RUN>.sid <RUN>.log
  As in runfull, free the slot on every terminal outcome — completion, error, or a subprocess killed by the 10-minute timeout (exit 143). The only case that keeps the files is a run paused at NEEDS INPUT:.

In both run modes, run the command from the workspace root <WS> resolved at the start of prompt generation. Do not type a workspace path literally — use the resolved root, whatever it is on this machine.

State files live under the system temp directory, which is /tmp on Linux, WSL, and containers alike. If /tmp is not writable in the user's environment, use the session scratchpad directory instead and substitute it for /tmp everywhere above.

If the user types "prompt:" with nothing after it, respond: Please describe the task after "prompt:" and include the target submodule if known.
