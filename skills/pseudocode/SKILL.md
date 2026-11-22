---
name: pseudocode
description: >-
  Two-stage comment-first coding workflow: Stage 1 writes ~10% minimal scaffolding
  and ~90% single-line logic comments (all branching as comments) across as many
  folders and files as the task needs; the user reviews and edits that pseudocode
  in place; Stage 2 implements it. Use for /pseudocode, "pseudocode mode", or staged
  comment-first implementation.
---

# Pseudocode (Two-Stage Comment-First Coding)

The pseudocode is a shared draft of the user's mental model. The first pass is expected to be wrong in places; the user corrects it by editing the comments directly, and the cycle repeats until the logic matches. Coding is iterative: write Stage 1 for **easy reading and editing** by a human reviewer.

Scope is language-agnostic and scales to whatever the prompt asks for: a single loop, one function, or a whole tree of new folders and files.

## Stage 1: Scaffold + Logic Comments

Write the skeleton straight into the target files, creating every folder and file the task needs. The split is loose, roughly 10% code and 90% comments.

### Real code (~10% scaffold)
- Folder and file creation, module/package headers, and imports.
- Type, struct, enum, class, and interface declarations, with their fields.
- Function, method, or procedure signatures.
- Outer loop headers (`for`, `while`, `foreach`, range loops) at function-body level or inside other real loops.

### Comments (~90% logic)
- **Single-line comment token on every line.** Use the target programming language's single-line comment syntax (`//`, `#`, `--`, `;`, etc.), or default to generic `//` when no language is specified or no single-line token exists. Repeat the single-line token on every line; span comments (`/* */`, `""" """`, `<!-- -->`) stay out.
- **Plain-language logic.** Describe *what happens and why* in clear prose rather than writing code inside comments: write `// if loading successful`, not `// if (check_loading() == true)`.
- **No length limit.** A comment can be as long or detailed as the logic requires. Split into separate lines only where there are separate logical steps or branches.
- **All branching lives in comments**: `if`/`else`, `switch`/`match`, ternaries, guard clauses, early returns, and error/exception handling. Give every condition and outcome its own line, including a "do nothing" branch, so the user can see and edit every decision.
- **Everything else in a body** is a comment too: calculations, state mutations, validations, calls, and return values.
- **Loops inside a commented branch become comments.** Because a real loop header cannot sit inside a commented-out `if`, write nested loops inside branches as comments (`// for each connected user`) and indent their body beneath them.
- **Function names are encouraged** when referencing helpers or functions scaffolded elsewhere in the file or across other files/modules (`// call stream.init_service with drone position`). Treat referenced names as reading aids for review rather than enforcing strict signature or name checks during Stage 1.

### Indentation shows scope
Indent comments by tab stops (or the project's standard indent unit) **before** the comment token to represent nested scope just as brackets or blocks would. Each nested branch body, loop body, or case arm indents one level deeper:

```text
function boot() {
	// load config from disk
	// if loading successful
		// initialize system
		// for each subsystem in config
			// start subsystem
			// if subsystem fails to start
				// log subsystem name and continue
	// if loading failed
		// gracefully shutdown with error
}
```

### Examples

Request: *"a for loop that takes an int array and increases every element by 10"*

```text
for i in len(array):
	// read array[i] and increase value by 10
```

Branching written as indented comments:

```text
function update_drone(drone, users) {
	// if drone is in range of 100 meters
		// initialize streaming service and emit to users
	// else
		// do nothing
}
```

Loop header kept as code, with branching inside it written as comments:

```text
function process_scores(scores, bonus) {
	// if scores is empty
		// return 0

	for i in 0..len(scores) {
		// if scores[i] is negative
			// clamp scores[i] to 0
		// add bonus to scores[i] in place
		// add scores[i] to running total
	}

	// return running total divided by number of scores
}
```

Multi-file scope: create the directory tree, scaffold each file in the target language, and reference functions across files by name:

```text
drone_stream/
├── main.<ext>          entry point: load config, start drone loop
├── drone/drone.<ext>   Drone type, in_range, update_telemetry
└── stream/stream.<ext> init_service, emit_to_users
```

```text
// main.<ext>
function main() {
	// load config from drone_stream.ini
	for (;;) {
		// call drone.update_telemetry for active drone
		// if drone.in_range reports within 100 meters
			// call stream.init_service if not already running
			// call stream.emit_to_users with latest frame
	}
}
```

### Stage 1 completion criterion (review gate)
Stage 1 is complete when every file the task needs exists and every piece of logic and branching is written as indented single-line comments. Stage 1 files are incomplete drafts and need not compile or pass syntax/lint checks (e.g., indentation-sensitive languages or unused imports/variables), so skip build, typecheck, and linter runs during Stage 1. Then:

1. List the created and modified files (as a directory tree when spanning multiple folders).
2. Highlight assumptions or logic choices most likely to differ from the user's mental model.
3. Invite the user to edit the pseudocode comments directly in the files or reply with adjustments.
4. Wait. Stage 2 starts only on the user's explicit go-ahead.

### Review iterations
When the user replies with adjustments or indicates they edited the files, re-read the affected files first—the user's edited comments, wording, indentation, and any code they wrote directly are the source of truth. If more Stage 1 changes were requested in chat, update the comments accordingly and pause at the review gate again until the user approves for implementation.

---

## Stage 2: Implementation (post-approval)

1. **Re-read** all Stage 1 files so any direct user edits to the comments or code are captured.
2. **Snapshot to `.pseudocode/<NNN>_<name>/`**: Before replacing any comments, copy only the Stage 1 files touched in this task into `<project-root>/.pseudocode/<NNN>_<name>/`, preserving each file's relative folder path from the project root (e.g., `cp --parents <touched-files...> .pseudocode/000_webhook/`). `<NNN>` is a 3-digit counter starting at `000` and incrementing by 1 (`001`, `002`, ...) from existing folders in `.pseudocode/`, and `<name>` is a short slug you pick for the task. Skip only if the user explicitly asked for no backup.
3. **Replace comments in place** with working code in the target language, matching the logic and indentation scope of the reviewed comments. Preserve any real code the user added or edited during review as-is, adapting the generated code around it to match their variables and calls. Resolve any cross-file function names to their actual signatures. Keep a comment only where it still adds lasting value.
4. **Verify** once all files are implemented by running the project's standard build, typecheck, linter, or test command.
5. **Report divergence**: if any comment (or interaction with user-added code) proved infeasible or ambiguous when translated to real code, explain what changed and why rather than silently altering the agreed logic.

If the user later asks for new features or behavior changes, scaffold the new parts as Stage 1 pseudocode first and run the same review gate.
