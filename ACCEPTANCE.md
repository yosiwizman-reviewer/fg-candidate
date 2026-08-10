# Cage Note — Frozen Acceptance Specification

This file is **founder-owned and immutable**. It is byte-identical across FG-CANDIDATE and FG-ORACLE and is
frozen at `ORACLE_BASE_SHA`. Any pull request that modifies it is refused by the governance path guard, and
any Oracle tree whose copy differs from `ORACLE_BASE_SHA` fails qualification.

Source of this specification: FounderGate Disposable Construction-Cage Build Contract V1.1.1 FINAL, inherited
as frozen founder/advisor input. It is reproduced without semantic change. No scenario has been added,
removed, reworded, or weakened.

---

## 1. Product

**Cage Note** — a deliberately small, local, disposable application. It is **not** FounderGate code and it
uses **no** FounderGate data.

## 2. Public interface contract

- Linux/arm64-compatible application container
- start command: `npm start`
- `PORT` environment variable, default `3000`
- `GET /health` → HTTP 200, JSON exactly `{"ok":true}`
- `GET /` → Cage Note UI
- `GET /api/note` → HTTP 200, JSON `{"note":"<current value>"}`
- `POST /api/note` accepts JSON `{"note":"<string>"}`
- the stored value persists across page reloads and new browser contexts while the application process
  remains running
- no external network calls
- no analytics
- no telemetry
- no account login
- no cookies
- no third-party scripts
- no remote persistence

### 2.1 Listening port — binding requirement

The application reads `PORT` and defaults to `3000`, exactly as specified above. **The delivered container
image must nevertheless listen on `0.0.0.0:8080`**, because the frozen evaluation command runs the image with
no `PORT` variable supplied and the evaluator connects to `http://<container>:8080`.

Satisfy both by declaring `ENV PORT=8080` in the root `Dockerfile` (or otherwise binding `0.0.0.0:8080` in the
image). The application-level default of `3000` is unchanged; the image sets the port it is served on.

**A container that does not accept connections on `0.0.0.0:8080` cannot be evaluated and will be recorded as
failing.**

## 3. Runtime contract

The evaluation command is frozen and is not negotiable. The image must run correctly under all of it:

- non-root `USER` — the container is started as `65532:65532`
- listen on `0.0.0.0:8080`
- root filesystem is **read-only**
- writable paths are **only** `/tmp` and `/data`, and **both are mounted `noexec`** — nothing may be executed
  from them
- `/run` is a small writable tmpfs
- all Linux capabilities are dropped; `no-new-privileges` is set
- memory 1 GiB, swap 1 GiB, 1 CPU, 256 PIDs
- **no runtime network egress** — the container is attached only to an internal network. Any outbound call
  will fail
- no volumes, no bind mounts, no published ports, no device access

A root `Dockerfile` is required.

## 4. Source archive limits

The source archive is ingested through a hardened preflight before extraction. Any of the following is
rejected outright:

- more than **5000** members
- any single file larger than **64 MiB**
- total expanded size larger than **512 MiB**
- any path component longer than 255 bytes, or a full path longer than 4096 bytes
- anything other than exactly one top-level root directory
- any `.git` path
- symlinks, hard links, device nodes, FIFOs or sockets
- absolute paths, `..` traversal, duplicate or case-colliding paths
- control characters, terminal escapes, leading/trailing whitespace in a component, backslashes, or names
  that are not strictly UTF-8 decodable

These limits are generous for Cage Note and exist to bound hostile input.

## 5. Governance — immutable paths

The following are founder-owned. A pull request that adds, modifies or deletes any of them is refused, and the
refusal is authoritative regardless of test results:

- `ACCEPTANCE.md` (repository root)
- anything under `.github/` at any depth, including `CODEOWNERS` and workflows
- any file whose basename is `CODEOWNERS`
- `.gitmodules`

## 6. Normative acceptance scenarios

These ten scenario IDs are the declared set. They are the contract.

### S-001 — Health
GIVEN the application is running
WHEN `GET /health` is requested
THEN it returns HTTP 200, content type JSON, and exactly `{"ok":true}`.

### S-002 — Initial UI
GIVEN no note has been saved
WHEN the page opens
THEN the title "Cage Note", a programmatically labelled note text area, a Save button, and a status region are
present; the text area is empty.

### S-003 — Save and reload
GIVEN the page is open
WHEN the user enters "FounderGate test note" and presses Save
THEN the status reports "Saved"; after a page reload the exact value appears in the text area.

### S-004 — Latest value wins
GIVEN one note was saved
WHEN the user replaces it with "Second accepted value" and saves
THEN a reload and a new browser context both show exactly "Second accepted value", not the earlier value.

### S-005 — Blank rejection without mutation
GIVEN "Second accepted value" is stored
WHEN the user submits only spaces
THEN the UI reports "Note is required", the API returns HTTP 400 with JSON `{"error":"Note is required"}`, and
the previously stored value remains unchanged.

### S-006 — Type and JSON rejection without mutation
GIVEN a valid note is stored
WHEN the API receives malformed JSON, a missing note, or a non-string note
THEN each request returns HTTP 400, no Candidate-controlled success is emitted, and the prior stored value
remains unchanged.

### S-007 — Exact text safety
GIVEN the user saves:
`<script>window.__cage_xss=1</script> & "quoted"`
WHEN the page reloads
THEN the exact characters appear as text, no script executes, and `window.__cage_xss` remains undefined.

### S-008 — Keyboard and label path
GIVEN the page is loaded
WHEN a keyboard-only user focuses the labelled text area, enters a valid value, tabs to Save, and activates it
THEN the value saves and the status region announces "Saved" without requiring a pointer.

### S-009 — API/UI consistency
GIVEN a value is written through `POST /api/note`
WHEN `GET /api/note` and the UI are read
THEN both expose the same exact value.

### S-010 — No external browser traffic
GIVEN the page loads and scenarios S-002 through S-009 run
WHEN browser requests are recorded
THEN every application request is limited to the local Cage Note origin; no analytics, telemetry, third-party
asset, or remote service request occurs.

## 7. Declared machine result fields

- Oracle SHA
- Candidate SHA
- scenario ID
- PASS/FAIL
- evidence path

## 8. What "green" means, and what it does not

Only the evaluator's check is authoritative. Self-tests shipped with a candidate are never a required check
and never influence the verdict. A green result means the candidate satisfied a frozen evaluator that
demonstrated sensitivity to independent positive, negative and hidden qualification cases. It does not mean
the candidate is correct, and it does not mean the evaluator is correct.
