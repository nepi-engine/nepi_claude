# NEPI PROMPT GENERATION MODE

When a user message begins with "prompt:", treat everything after that word as a prompt generation request. Do not enter prompt generation mode for any other input — respond to all other messages normally as Claude Code.

When "prompt:" is detected, check whether you have enough information to define a clear MVP boundary. If the task description is missing any of the following, ask one specific question to fill the gap before drafting:

- Which submodule or repo path is affected (if not obvious from the task)
- Whether this is Creation, Documentation, Development Support, Audit, or Release
- For Creation tasks: what the component does, what hardware or interface it connects to, and whether a spec exists

Only ask one clarifying question at a time. If the task is clear enough to draft, draft it without asking.

When you have enough information, produce the prompt as a plain text code block following these rules:

RULE 1: Plain text only inside the code block. No markdown formatting, no bold, no # headers, no bullet points.

RULE 2: Begin with the framework file read sequence, adapted to the task:
  Read src/nepi_claude/NEPI-LORE.md in full.
  Read src/nepi_claude/NEPI-CODEX.md in full.
  Read the top-level CLAUDE.md in full.
  Read src/<submodule>/CLAUDE.md in full.
  Do not take any action until you have read all documents.

RULE 3: State the target repo and exact path being modified.

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

After presenting the prompt, offer the user three options:
  Option A: Request adjustments — describe what to change and a revised prompt will be produced.
  Option B: "runfull" — execute fully automated with no permission prompts of any kind.
  Option C: "runwatch" — execute normally, pausing for permission prompts so the user can approve each action.

CONCURRENT RUNS: Multiple prompts may run at the same time. Each run gets its own set of state files (.txt, .sid, .log) identified by a slot number, so runs never clobber each other. Before launching in either run mode, allocate a free slot with the Bash tool:
    RUN=/tmp/nepi_prompt_run; if [ -e "$RUN.txt" ]; then i=2; while [ -e "${RUN}_$i.txt" ]; do i=$((i+1)); done; RUN="${RUN}_$i"; fi; : > "$RUN.txt"; echo "$RUN"
The first run uses /tmp/nepi_prompt_run; the next free slot is /tmp/nepi_prompt_run_2, then _3, and so on. The command prints the concrete base path for this run and reserves it by creating an empty placeholder .txt. Below, <RUN> stands for that concrete base path (for example /tmp/nepi_prompt_run_2). Remember it for the rest of the conversation, and substitute it literally into every subsequent command for this run — shell variables do not persist between separate Bash tool calls, so do not rely on $RUN after the allocation command. To list all active runs at any time: ls /tmp/nepi_prompt_run*.txt

If the user says "runfull" (or any clear variant like "run full", "run it full", "full run"):

  Step 1: Allocate a slot as above, then write the exact prompt text (the contents of the code block, no surrounding markdown) to this run's <RUN>.txt (for example /tmp/nepi_prompt_run_2.txt) using the Write tool, overwriting the placeholder.

  Step 2: Mint a fixed session id so the run can be resumed later, then launch it in the background using the Bash tool with run_in_background set to true, so it is not bound by the 10-minute foreground timeout. Redirect all output to this run's log file (substitute the concrete <RUN> path):
    SID=$(uuidgen); echo "$SID" > <RUN>.sid
    "$CLAUDE_CODE_EXECPATH" --dangerously-skip-permissions --session-id "$SID" -p "$(cat <RUN>.txt)" > <RUN>.log 2>&1
  All file edits and bash commands proceed without any permission prompts. Do not wait synchronously — the background process keeps running across turns and re-invokes you when it exits.

  Step 3: When the background subprocess exits, read this run's <RUN>.log. If the run paused for clarification (its final output is a line beginning NEEDS INPUT:), relay that question to the user verbatim and do NOT report completion and do NOT delete the slot files — the run is not done. When the user answers, resume the same session with their answer, appending to the same log:
    "$CLAUDE_CODE_EXECPATH" --dangerously-skip-permissions --resume "$(cat <RUN>.sid)" -p "<the user's answer>" >> <RUN>.log 2>&1
  Re-check for a trailing NEEDS INPUT: line after each resume; keep resuming until the run finishes without one. Once the run finishes without a NEEDS INPUT: line, report a one-sentence summary of what was completed or any errors encountered, then free the slot by deleting this run's state files:
    rm -f <RUN>.txt <RUN>.sid <RUN>.log

  To watch progress live while the background run is going, tell the user they can run in a terminal (using this run's concrete log path):
    tail -f <RUN>.log | grep --line-buffered '===== STEP'
  This shows each step banner the instant it begins (drop the grep to see full output). The user can also ask "what step?" at any time and you will read that run's <RUN>.log and report the current step; if several runs are active, report the step for each active run.

If the user says "runwatch" (or any clear variant like "run watch", "run it watch", "watch run"):

  Step 1: Allocate a slot as above, then write the exact prompt text (the contents of the code block, no surrounding markdown) to this run's <RUN>.txt using the Write tool, overwriting the placeholder.

  Step 2: Mint a fixed session id, then execute via Bash with a timeout of 600000ms (substitute the concrete <RUN> path):
    SID=$(uuidgen); echo "$SID" > <RUN>.sid
    "$CLAUDE_CODE_EXECPATH" --session-id "$SID" -p "$(cat <RUN>.txt)"
  Permission prompts will appear in the terminal as normal. The user approves or denies each action. This runs in the foreground, so the 10-minute maximum applies — if the run exceeds it, the subprocess is killed (exit 143) and returns no final report. For long automated runs that do not need per-action approval, use runfull instead.

  Step 3: After the subprocess finishes, check its output. If it ends in a line beginning NEEDS INPUT:, relay that question to the user and do NOT report completion and do NOT delete the slot files; when the user answers, resume the same session:
    "$CLAUDE_CODE_EXECPATH" --resume "$(cat <RUN>.sid)" -p "<the user's answer>"
  and repeat until it finishes without a NEEDS INPUT: line. Once it finishes without a NEEDS INPUT: line, report a one-sentence summary of what was completed or any errors encountered, then free the slot by deleting this run's state files:
    rm -f <RUN>.txt <RUN>.sid <RUN>.log

In both run modes, run the command from the working directory /home/production/nepi_engine_ws.

If the user types "prompt:" with nothing after it, respond: Please describe the task after "prompt:" and include the target submodule if known.
