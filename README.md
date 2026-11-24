# `/pseudocode` — Two-Stage Comment-First Coding Skill

It's a langauge agnostic skill to help you write complex state logic and refine/review complex logical states and such by going through less expensive refinement steps, and maintain a history of changes in .pseudocode folder for future reference. Suppose you asked for a transaction system but the model forgot a step or misunderstood something, you can refine it from the get go and be in the loop while also being familiar with your codebase. This can also span across many files and so on.


1. **Stage 1 (Scaffold & Single-Line Logic Comments)**: Writes ~10% structural scaffolding (folders, files, types, function signatures, outer loop headers) and ~90% plain-language single-line comments representing every branch, check, and operation. Execution pauses here so you can review and tweak the comments (or add code yourself) to match your mental model.
2. **Stage 2 (Post-Review Snapshot & Implementation)**: After you approve, the agent re-reads your edits, snapshots the touched Stage 1 files (preserving their relative folder path) into `.pseudocode/<NNN>_<name>/` (`000_...`, `001_...`, etc.), and replaces the comments in place with working code while keeping any code you wrote during review intact.

---

## Installation

Install globally with the [Skills CLI](https://skills.sh/):

```bash
npx skills add manhattantea/Pseudocode_Skill -g
```

Once installed, invoke it with `/pseudocode` or by asking for `"pseudocode mode"`.

---

## Key Rules

- **~10% Code / ~90% Comments (for new work)**: On new files and functions, types, function signatures, and outer loop headers (`for`, `while`) are written as code while all new logic and branching (`if`/`else`, `switch`/`match`, error checks, early returns) are written as single-line comments.
- **Non-Destructive on Existing Code**: Running `/pseudocode` on existing files let's you refine and improve without deleting existing code.
- **Tab Indentation Shows Scope**: Nested branches and blocks indent one level deeper before the comment token (`//`, `#`, `--`), acting like braces so scope is clear at a glance.
- **Plain-Language Logic**: Comments describe *what happens* in readable prose (`// if response status is 200 OK`), not raw syntax (`// if (res.status == 200)`).
- **Multi-File Aware + Clickable Review Summary**: Creates full folder/file trees when needed, names scaffolded functions across files (`// call auth.verify_token on request header`), and ends Stage 1 with clickable file/line links to every `[NEW]` and `[MODIFIED]` section.

---

## Stage 1 Example: HTTP Webhook Receiver & Retry Worker

AI generated example for now, but the logic is good i'd say.

Prompt:
> `/pseudocode` Create an HTTP service that accepts incoming webhook payloads, verifies their auth signature, queues valid events, and runs a background worker that forwards queued events to an upstream endpoint with retries.

### Stage 1 Output (Awaiting User Review)

```text
webhook_service/
├── server.go
├── auth/verify.go
└── worker/forwarder.go
```

```go
// auth/verify.go
package auth

import "net/http"

func VerifySignature(r *http.Request, secret string) bool {
	// read X-Webhook-Signature header from request
	// if signature header is missing
		// return false

	// compute HMAC-SHA256 of request body using secret
	// if computed signature matches header value in constant time
		// return true
	// else
		// return false
}
```

```go
// server.go
package main

import "net/http"

type WebhookEvent struct {
	ID      string
	Payload []byte
	Retries int
}

func HandleWebhook(w http.ResponseWriter, r *http.Request) {
	// if request method is not POST
		// respond with 405 Method Not Allowed and return

	// call auth.VerifySignature with request and webhook secret
	// if signature verification failed
		// respond with 401 Unauthorized and return

	// read and parse JSON body into WebhookEvent
	// if body exceeds 1MB or JSON is malformed
		// respond with 400 Bad Request and return

	// push event onto in-memory delivery queue
	// if queue is full
		// respond with 503 Service Unavailable and return
	// else
		// respond with 202 Accepted
}
```

```go
// worker/forwarder.go
package worker

func ProcessQueue(queue <-chan WebhookEvent, upstreamURL string) {
	for event := range queue {
		// build HTTP POST request to upstreamURL with event.Payload
		// send request with a 5 second timeout

		// if request succeeded with 2xx status code
			// log delivery success for event.ID
		// if request failed or returned 5xx status code
			// if event.Retries is less than 3
				// increment event.Retries by 1
				// wait exponential backoff based on retry count
				// re-queue event for another attempt
			// else
				// move event to dead-letter log and alert on-call
		// if upstream returned 4xx client error
			// do not retry; log permanent rejection for event.ID
	}
}
```

At this point **Stage 1 stops**. You can directly edit any comment line like for example.

```
// if upstream returned 4xx client error
	// do not retry; log permanent rejection for event.ID

Changed to:

// if upstream returned 4xx client error
	// do not retry; log permanent rejection for event.ID and shutdown entire process immediately
```

and tell the agent to proceed to **Stage 2** to implement it.
