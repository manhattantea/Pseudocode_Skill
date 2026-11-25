---
name: pseudocode
description: >-
  Two-stage comment-first coding workflow: Stage 1 writes ~10% minimal scaffolding
  and ~90% indented single-line logic comments for new or changed logic while keeping
  existing working code intact; the user reviews and edits that pseudocode in place;
  Stage 2 implements it, checkpointing via Git commits (or .pseudocode/ shell copy
  fallback). Use for /pseudocode, "pseudocode mode", or staged comment-first
  implementation and refactoring.
---

# Pseudocode (Two-Stage Comment-First Coding)

The pseudocode is a shared draft of the user's mental model. The first pass is expected to be wrong in places; the user corrects it by editing the comments directly, and the cycle repeats until the logic matches. Coding is iterative: write Stage 1 for **easy reading and editing** by a human reviewer.

Scope is language-agnostic and scales to whatever the prompt asks for: a small inline change inside an existing function, a new function, or a whole tree of new folders and files.

## Stage 1: Scaffold + Logic Comments

The ~10% code / ~90% comment split applies **only to newly written or changed logic**—never to working code that already exists in the codebase.

### New files vs. existing files (critical distinction)
- **New files and brand-new functions**: Create the file/signature skeleton (~10% code) and write the body logic as indented single-line comments (~90%).
- **Existing files with working code**: Use targeted edits (`edit_file`) rather than rewriting whole files.
  - **Unchanged code stays untouched**: Leave existing working functions, structs, constants, imports, and unchanged lines inside modified functions 100% intact as real code. Never wipe, summarize, or regress working code back into comments.
  - **Additive changes inside an existing function**: Keep the surrounding working lines intact and insert indented Stage 1 comments **inline between existing code lines** at the exact spots where new behavior belongs.
  - **Replacing obsolete logic**: Replace *only* the specific lines or obsolete function body being retired with the new Stage 1 scaffold and logic comments, leaving every unaffected line and sibling function untouched.

### Real code (~10% of new work)
- Folder and file creation, module/package headers, and imports.
- Type, struct, enum, class, and interface declarations, with their fields.
- Function, method, or procedure signatures.
- Outer loop headers (`for`, `while`, `foreach`, range loops) in new functions or inside other real loops.

### Comments (~90% of new work)
- **Single-line comment token on every line.** Use the target programming language's single-line comment syntax (`//`, `#`, `--`, `;`, etc.), or default to generic `//` when no language is specified or no single-line token exists. Repeat the single-line token on every line; span comments (`/* */`, `""" """`, `<!-- -->`) stay out.
- **Plain-language logic, zero raw syntax.** Describe *what happens and why* in clear prose rather than stuffing language syntax, borrow sigils, or raw expressions behind a comment marker: write `// if loading successful` and `// call copilot.request_completion and update session cache`, not `// if (check_loading() == true)` or `// Ok(Some(t)) -> copilot::request(&ctx, &mut self.cache)`.
- **One logical step or decision per line.** Avoid jamming multiple steps and hidden fallbacks onto one dense line; give each distinct action or check its own line so the reviewer can edit any single step cleanly. No artificial character-length cap applies to a single step.
- **All new branching lives in comments**: `if`/`else`, `switch`/`match`, ternaries, guard clauses, early returns, and error/exception handling. Give every condition and outcome its own line, including a "do nothing" branch where relevant, so the user can see and edit every decision.
- **Everything else in new logic** is a comment too: calculations, state mutations, validations, calls, and return values.
- **Loops inside a commented branch become comments.** Because a real loop header cannot sit inside a commented-out `if`, write nested loops inside branches as comments (`// for each connected user`) and indent their body beneath them.
- **Function and type names as landmarks**: Naming helpers or types scaffolded elsewhere in the file or across sibling files (`// call stream.init_service with drone position`) is encouraged as a reading aid, without writing raw call syntax or enforcing strict signature checks during Stage 1.

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

#### 1. Modifying an existing function inline (preserving working code)
Request: *"streamline request handling by adding per-language gating and non-blocking session refresh"*

```text
function handle_completion(doc, cursor, session) {
	let cfg = config.load();
	if !cfg.enabled {
		return null;
	}

	// if completions are disabled for doc.language in cfg
		// return null immediately

	// if session token is within 2 minutes of expiry
		// trigger non-blocking session refresh in background panel state
		// if no valid token is available yet
			// update status bar to Sign in required and return null

	let prompt_ctx = prompt.build_context(doc, cursor, cfg.prefix_chars, cfg.suffix_chars);
	return backend.request_fim(prompt_ctx, cfg, session);
}
```
Notice `config.load()`, `if !cfg.enabled`, `prompt.build_context(...)`, and `backend.request_fim(...)` remain untouched real code—only the newly proposed behavior is inserted as indented Stage 1 comments.

#### 2. Greenfield loop & branching
Request: *"a for loop that takes an int array and increases every element by 10"*

```text
for i in len(array):
	// read array[i] and increase value by 10
```

Branching and loops in a new function:

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

#### 3. Multi-file scope
Create new files and folders where needed, edit existing files surgically, and reference functions across files by name:

```text
// main.<ext> (new file)
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
Stage 1 is complete when every new or modified file has the requested changes expressed as indented single-line comments while all unrelated existing code remains intact. Stage 1 files are incomplete drafts and need not compile or pass syntax/lint checks (e.g., indentation-sensitive blocks or unused imports/variables), so skip build, typecheck, and linter runs during Stage 1. Then:

1. **Checkpoint Stage 1**: Commit only the touched files (`git add <touched-files...> && git commit --no-verify -m "pseudocode(stage1): <short-summary>"`). If not a Git repo or `git commit` fails/is disallowed, fall back to copying only the touched files via a shell copy command (never LLM file writes, to save tokens) into `<project-root>/.pseudocode/<NNN>_<name>/` preserving relative paths (e.g., `cp --parents <touched-files...> .pseudocode/000_<slug>/`, where `<NNN>` increments from `000`). Skip if the user asked for no backup.
2. Highlight assumptions or logic choices most likely to differ from the user's mental model.
3. End the response with a **clickable file summary** grouped by folder (as a nested Markdown list, outside any code fence) where every created or modified file is a clickable link (`[path/to/file](file:///absolute/path/to/file)` or `#Lstart-Lend` for specific modified sections in existing files), marked `[NEW]` or `[MODIFIED]` with a brief one-line note of what changed, so the user can jump straight to each pseudocode block.
4. Invite the user to edit the pseudocode comments directly in the files or reply with adjustments, then wait. Stage 2 starts only on the user's explicit go-ahead.

### Review iterations
When the user replies with adjustments or indicates they edited the files, re-read the affected files first—the user's edited comments, wording, indentation, and any code they wrote directly are the source of truth. If more Stage 1 changes were requested in chat, update only the relevant comments accordingly and pause at the review gate again until the user approves for implementation.

---

## Stage 2: Implementation (post-approval)

1. **Re-read** all touched Stage 1 files so any direct user edits to the comments or code are captured.
2. **Checkpoint Stage 2 (before editing)**: Commit the reviewed Stage 1 files if anything changed since Stage 1 (`git add <touched-files...> && git commit --no-verify -m "pseudocode(stage2): <short-summary>"`). If not a Git repo or `git commit` fails/is disallowed, fall back to copying only the touched files via a shell copy command (`cp --parents <touched-files...> .pseudocode/<NNN>_<name>/`, incrementing `<NNN>` from `000`) so zero LLM output tokens are spent. Skip if the user asked for no backup.
3. **Replace Stage 1 comments in place** with working code in the target language, matching the logic and indentation scope of the reviewed comments:
   - Leave all pre-existing working code, pre-existing doc/explanatory comments, and any real code the user added during review intact, adapting the newly generated code around them to fit existing variables, types, and signatures.
   - Resolve cross-file function names to their actual signatures.
   - Keep a comment from Stage 1 only where it still adds lasting explanatory value.
4. **Verify** once all files are implemented by running the project's standard build, typecheck, linter, or test command.
5. **Report divergence**: if any comment (or interaction with existing/user-added code) proved infeasible or ambiguous when translated to real code, explain what changed and why rather than silently altering the agreed logic.

If the user later asks for new features, refinements, or refactors on already-implemented files, apply Stage 1 **incrementally** on top of the existing working code (inserting comments only for the new/changed parts) and run the same review gate.
