# hookwatch: Claude Code Hook Observability

## Problem Decomposition

### 1. Reframe

**What the user wants:**
Claude Code hooks are opaque. They fire silently, return data Claude may or may not use, and potentially flood context with noise. The user wants **visibility** into hook behavior — not just "is it working?" (debugging) but "is it useful?" (auditing).

**Goal (not method):**
Answer questions like:
- "What did my hooks do this session?"
- "How many tokens did they add to Claude's context?"
- "Is this hook adding signal or noise?"
- "Which hook fired 50 times with identical output?"

**Assumptions to challenge:**

| Assumption | Challenge |
|------------|-----------|
| Users can't see hook behavior | True — `--debug` exists but is developer-focused, not dashboard-like |
| Hooks add noise | Maybe — need data to know |
| A single hook can observe others | Unclear — need to investigate |
| Token counting is possible | Rough estimates only (chars → tokens heuristic) |

**Constraints:**
- Must work within Claude Code's hook system (stdin JSON, stdout, exit codes)
- Must compose with other hooks (psst, user hooks) — not replace them
- Should follow FP/Unix philosophy (one thing well, composable)

**Success:**
```bash
hookwatch audit
# Session: 2h 15m
# Hooks fired: 142 times across 4 hooks
# Context added: ~3.2k tokens
#
# Top talkers:
#   psst pre      — 47 fires, 0 tokens (all exit 0, no output)
#   lint-check    — 38 fires, 2.1k tokens
#   format-on-save— 38 fires, 0 tokens
#   notify        — 19 fires, 1.1k tokens
#
# ⚠ lint-check returned identical output 35/38 times
```

---

### 2. Decompose

| Layer / Aspect | Question to Answer |
|----------------|-------------------|
| **Observation mechanism** | How do we capture what hooks receive and return? |
| **Data model** | What do we store per hook event? |
| **Storage** | Where? SQLite? JSONL? In-memory? |
| **Aggregation** | What metrics derive insight? (counts, deduplication, token estimates) |
| **CLI interface** | What commands? `audit`, `tail`, `blame`? |
| **Lifecycle** | Always-on? Opt-in per session? |
| **Composability** | How does this work with existing hooks? |

**Dependencies:**
1. Observation mechanism (must solve first — everything depends on it)
2. Data model (what we capture)
3. Storage (where we put it)
4. Aggregation + CLI (can develop in parallel once 1-3 settled)

---

### 3. Unknowns

**Technical (require investigation):**
1. Can a hook see what other hooks returned? Or only its own input?
2. Does Claude Code provide session IDs to correlate events?
3. What environment variables are available to hooks?

**Design (require decision):**
4. Language: OCaml (like psst)? Shell (simple)? Python (prototyping)?
5. Always-on or explicit start/stop?
6. Separate tool or extension to psst?

---

## Solution Enumeration

### Path A: Observer Hook (Self-Logging)

**Insight:** A hook configured on all event types that logs what IT sees.

**Steps:**
1. Configure hookwatch as a hook on PreToolUse, PostToolUse, etc.
2. Hook reads stdin (JSON event), logs to ~/.hookwatch/events.db
3. Hook exits 0 with no output (doesn't interfere)
4. CLI queries the database for aggregates

**Limitation:** Only sees events it's registered for. Cannot see what OTHER hooks return.

---

### Path B: Wrapper/Proxy

**Insight:** Instead of configuring `my-hook.sh`, configure `hookwatch wrap my-hook.sh`. The wrapper logs I/O then delegates.

**Steps:**
1. User changes hook config: `"command": "hookwatch wrap ~/.claude/hooks/lint.sh"`
2. hookwatch reads stdin, logs it
3. hookwatch pipes stdin to actual hook, captures stdout/stderr/exit
4. hookwatch logs output, returns it to Claude Code
5. CLI queries aggregated data

**Advantage:** Sees exactly what each wrapped hook receives AND returns.
**Cost:** Requires user to modify hook configs.

---

### Path C: Debug Output Parser

**Insight:** `claude --debug` already outputs hook info. Parse it.

**Steps:**
1. User runs `claude --debug 2>&1 | hookwatch ingest`
2. hookwatch parses debug lines for hook events
3. Stores structured data

**Limitation:** Depends on debug output format (fragile). Requires special invocation.

---

### Path D: Voluntary Logging (Convention)

**Insight:** Establish a convention — hooks that want observability write to a shared location.

**Steps:**
1. Define format: hooks append to `~/.hookwatch/stream.jsonl`
2. Provide a library/helper: `hookwatch log --event=PreToolUse --input="$INPUT"`
3. CLI reads the stream

**Limitation:** Requires cooperation from each hook. Won't capture unmodified hooks.

---

### Path E: Hybrid (Observer + Wrapper)

**Insight:** Use observer hook for "what events fired", wrapper for "what did my hooks return".

**Steps:**
1. hookwatch registers as observer on all events (captures event metadata)
2. For detailed hook I/O, user wraps specific hooks with `hookwatch wrap`
3. Two levels of detail: basic (observer) and detailed (wrapper)

---

## Tradeoff Matrix

| Dimension | A: Observer | B: Wrapper | C: Debug Parser | D: Voluntary | E: Hybrid |
|-----------|-------------|------------|-----------------|--------------|-----------|
| Sees hook output | ❌ No | ✅ Yes | ⚠️ Partial | ⚠️ Opt-in | ✅ Yes |
| Setup friction | Low | Medium | Low | High | Medium |
| Fragility | Low | Low | High | Low | Low |
| Composability | ✅ Great | ⚠️ Config changes | ❌ Special run | ⚠️ Convention | ✅ Good |
| Accuracy | Partial | Full | Partial | Depends | Full where wrapped |

---

## Recommendation

**Path E: Hybrid** — but start with **Path A (Observer)** as MVP.

**Rationale:**
1. Observer hook is zero-friction: just add hookwatch as a hook, done
2. Gives immediate value: "what events fired, how often"
3. Wrapper can be added later for detailed I/O on specific hooks
4. Follows Unix philosophy: start simple, compose upward

**Tradeoff:** Observer alone can't see what other hooks RETURN. But it can see what events fired, which is 80% of the value.

**Reconsider if:** User specifically needs to audit hook OUTPUT (not just events). Then prioritize wrapper.

---

## Resolved Questions

1. **Language:** Scala 3 + Scala Native + Typelevel Toolkit (FP, native binary, batteries included)
2. **Scope:** Separate tool — different concerns, composable
3. **Storage:** JSONL — simplest thing that works, no database
4. **Token estimation:** Skip for MVP — add later if needed

---

## Investigation Results

**Environment variables available:**
- `CLAUDE_PROJECT_DIR` — project root
- `CLAUDE_CODE_REMOTE` — "true" if remote, unset locally
- `CLAUDE_ENV_FILE` — (SessionStart only) file to persist env vars

**stdin JSON structure (all events):**
```json
{
  "session_id": "abc123",           // ← perfect for correlation
  "transcript_path": "/path/...",   // ← could read for context
  "cwd": "/current/dir",
  "permission_mode": "default|plan|...",
  "hook_event_name": "PreToolUse",  // ← tells us event type
  // + event-specific fields (tool_name, tool_input, etc.)
}
```

**Multiple hooks:** Yes, all run in parallel. Observer won't interfere with other hooks.

**Conclusion:** Path A (Observer Hook) is fully viable. We get session_id for correlation, hook_event_name for categorization, and parallel execution means no interference.

---

## Implementation Plan (MVP: Observer Hook)

### Stack

**Scala 3 + Scala Native + Typelevel Toolkit**

```scala
//> using scala 3.3
//> using platform native
//> using toolkit typelevel:default
```

Why this stack:
- **Typelevel Toolkit** — batteries included (cats-effect, fs2, circe, decline)
- **Scala Native** — single binary, no JVM runtime
- **scala-cli** — no sbt, just run/build
- **JSONL** — simple append-only log, no database

### Architecture

```
┌─────────────────────────────────────────┐
│           IMPERATIVE SHELL              │
│  CLI: hookwatch audit | tail | init     │
│                                         │
│    ┌───────────────────────────────┐    │
│    │      FUNCTIONAL CORE          │    │
│    │                               │    │
│    │  Event → parse → append JSONL │    │
│    │  Events → aggregate → Report  │    │
│    │                               │    │
│    └───────────────────────────────┘    │
│                                         │
│  Hook: reads stdin, appends, exit 0     │
└─────────────────────────────────────────┘
```

### Domain Types

```scala
import io.circe.{Decoder, Encoder}
import io.circe.generic.semiauto.*

case class HookEvent(
  sessionId: String,
  timestamp: String,
  eventType: String,        // PreToolUse, PostToolUse, etc.
  toolName: Option[String], // for tool events
  cwd: String,
  rawJson: String           // preserve full input for debugging
) derives Encoder.AsObject, Decoder

case class SessionSummary(
  sessionId: String,
  startTime: String,
  eventCount: Int,
  byEventType: Map[String, Int],
  byTool: Map[String, Int]
)
```

### Files

| File | Purpose |
|------|---------|
| `src/domain.scala` | Domain types + circe codecs |
| `src/store.scala` | JSONL append/read (fs2) |
| `src/report.scala` | Pure aggregation functions |
| `src/cli.scala` | CLI with decline |
| `src/main.scala` | Entry point, wiring |

### CLI Commands

```bash
hookwatch init      # Add observer hooks to settings.json
hookwatch log       # (internal) append stdin to JSONL
hookwatch audit     # Summary of current/recent session
hookwatch tail      # Live stream of events
hookwatch sessions  # List recent sessions
hookwatch doctor    # Verify setup
hookwatch uninstall # Remove hooks
```

### Hook Registration

Observer hooks for all relevant events:
- PreToolUse (matcher: `*`)
- PostToolUse (matcher: `*`)
- UserPromptSubmit
- SessionStart
- SessionEnd
- Stop

Each configured as: `{ "command": "hookwatch log", "type": "command" }`

### Storage

JSONL at `~/.hookwatch/events.jsonl`:

```jsonl
{"sessionId":"abc123","timestamp":"2024-01-01T12:00:00Z","eventType":"PreToolUse","toolName":"Bash","cwd":"/home/user","rawJson":"..."}
{"sessionId":"abc123","timestamp":"2024-01-01T12:00:01Z","eventType":"PostToolUse","toolName":"Bash","cwd":"/home/user","rawJson":"..."}
```

- Append-only (no corruption risk)
- Human-readable (`cat`, `jq`, `grep`)
- Easy to rotate/archive
- fs2 streams for efficient reading

### Build & Verify

```bash
# Build native binary
scala-cli --power package --native src/ -o hookwatch -f

# Install to PATH
cp hookwatch ~/.local/bin/

# Configure hooks
hookwatch init
hookwatch doctor

# Use Claude Code normally, then:
hookwatch audit
```

---

## Beads (Issues)

| ID | Title | Depends On |
|----|-------|------------|
| hw-001 | Domain types + circe codecs | — |
| hw-002 | JSONL store (fs2 append/read) | hw-001 |
| hw-003 | Observer hook (log command) | hw-001, hw-002 |
| hw-004 | Report/aggregation functions | hw-001 |
| hw-005 | CLI commands (decline) | hw-002, hw-003, hw-004 |
| hw-006 | Homebrew formula | hw-005 |

---

## Decisions

- **Name:** `hookwatch`
- **Repo:** m-gris/hookwatch
- **Language:** Scala 3 + Scala Native
- **Toolkit:** Typelevel (cats-effect, fs2, circe, decline)
- **Build:** scala-cli (no sbt)
- **Storage:** JSONL (no SQLite)
- **Token estimation:** Skip for MVP, add later

