# TSVC: Treating AI Conversation Topics as Virtual Processes

*github.com/MouseRider/skills-tsvc*

---

Persistent AI agents have a memory problem that isn't about memory size. It's about isolation.

Run a single-session agent long enough across multiple domains — coding, finance, personal logistics — and you'll hit a wall. Compaction events blend everything into a lossy average. The agent starts reaching across domains, contaminating answers with irrelevant context. Chroma Research named this "context rot" in 2025. The problem is well-documented. The solutions are incomplete.

This post describes TSVC (Topic-Scoped Virtual Context): a system that treats AI conversation topics the way an OS treats processes. Separate address spaces. Controlled isolation. Topic-scoped compaction. We built it for a persistent Claude agent running on OpenClaw, measured 126 production topic switches across 12 topics with 3,735 exchanges tracked, and eliminated per-topic compactions entirely.

---

## Design Philosophy: One Agent, Many Topics

TSVC is designed specifically for the **personal assistant model** — one main agent as a single point of contact for everything.

The alternative is a swarm of specialized agents: a Finance Agent, a DevOps Agent, a Research Agent, each expert in its domain. This pattern has real appeal. But it has a fundamental UX problem: the user becomes the orchestrator. You switch between agents, you carry context across them, you remember which agent knows which decisions. The cognitive overhead doesn't disappear — it moves from the AI to you.

The personal assistant model is better. One persistent agent handles all domains — finances, infrastructure, projects, family logistics — and orchestrates sub-agents *behind the scenes* when parallelism helps. From the user's perspective: one conversation thread, full continuity, everything remembered. The agent acts as a genuine executive assistant, not a tool you have to drive.

But this design creates a specific technical problem that the swarm model doesn't have: **context mixing**. When one agent accumulates conversation across many domains, topics bleed into each other. Financial discussions surface infrastructure details. Family decisions contaminate DevOps answers. Over time, the context window becomes a lossy average of everything. Chroma Research named this "context rot" in 2025.

This is the problem TSVC solves. Not for swarm architectures (they distribute context externally) — for the personal assistant model, where one agent holds it all. The OS process metaphor provides the solution: give each topic its own isolated address space, manage switching explicitly, and share only what should be shared.

---

## The OS Metaphor

MemGPT (Packer et al., 2023) used the OS memory model as a foundation for AI agent memory — virtual memory paging, with a limited "physical" context window managed by an LLM-driven memory controller. It was a useful framework. But virtual memory is only half the OS story.

The other half is the process model.

Starting with Atlas (1962) and maturing through Unix (1970s), each process gets its own virtual address space. Context switches between processes save and restore complete state. One process cannot corrupt another's memory. The scheduler manages which process has access to the CPU at any moment.

Apply this to AI agent conversations:

- A **process** maps to a **topic** (a coherent conversation domain)
- A **virtual address space** maps to **topic context** (isolated conversation + decisions + state)
- A **scheduler** maps to a **topic detector** (determines which topic owns the current message)
- A **context switch** maps to a **topic switch** (save current state → load target state → fresh session)
- **IPC** (inter-process communication) maps to a **shared facts layer** (SSoT files accessible to all topics)

This isn't just a metaphor — it's a design specification. Each architectural decision in TSVC follows directly from this mapping.

---

## Architecture

The system has three persistent layers and one ephemeral one.

**The Kernel Layer (always in context, ~15-20k tokens):**
System prompt, identity, tool definitions, agent configuration. Fixed overhead. Does not change between topic switches.

**The Topic Awareness Layer (~3k tokens):**
A lightweight topic index loaded at every session start. Contains topic IDs, titles, last-active timestamps, and one-line summaries. The agent can see what topics exist and roughly what's in them without loading any context. This keeps awareness overhead minimal while enabling intelligent topic detection.

**Per-Topic Context Files (10-85KB each, on disk):**
Each topic maintains a context file with:
- Active decisions and their dependency chains
- Open items and action tracking
- Working files and artifacts
- Compressed recent exchanges (not raw transcripts)
- One-line topic summary for the index

Only the active topic's context file loads into the session. All others stay on disk.

**The Shared Facts Layer:**
SSoT (Single Source of Truth) files that all topics can read. User profile, long-term memory, facts that span domains. This is the "IPC" that lets topics communicate without contaminating each other's isolated contexts.

Here's the conceptual layout:

```
┌─────────────────────────────────────────────────────┐
│               KERNEL (Always in Context)             │
│  System prompt, identity, tools, shared facts        │
│  ~15-20k tokens                                      │
└─────────────────────────────────────────────────────┘

┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Topic A  │  │ Topic B  │  │ Topic C  │  │ Topic D  │
│ [ACTIVE] │  │ [PAGED]  │  │ [PAGED]  │  │ [PAGED]  │
│ ~20KB    │  │ on disk  │  │ on disk  │  │ on disk  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘

┌─────────────────────────────────────────────────────┐
│            Topic Awareness Layer (~3k tokens)        │
│  ID, title, last_active, summary per topic           │
└─────────────────────────────────────────────────────┘
```

---

## How a Topic Switch Works

Topic switches require a clean session reset to guarantee context isolation. The system handles this transparently:

1. A message arrives → the gateway plugin runs `detect-topic-switch.js` before the LLM sees it
2. Topic match detected → `tsvc-switch.sh` saves current topic context to disk
3. A session reset is triggered in the background
4. The user's next message creates a fresh session → `tsvc-boot.sh` loads the pending topic context
5. The agent responds with full target topic context, zero cross-topic contamination

The topic detection script uses fuzzy title matching first, then semantic classification fallback. This keeps topic detection deterministic and free — no LLM tokens required to parse "switch to finance topic."

The core topic manager (`tsvc-manager.js`) handles the state transitions:

```javascript
// Save current topic state before switch
async function saveTopicContext(topicId, recentExchanges, decisions) {
  const contextPath = `tsvc/contexts/${topicId}.md`;
  const content = renderContext({
    decisions: buildDecisionChains(decisions),
    openItems: getOpenItems(topicId),
    recentExchanges: compressExchanges(recentExchanges),
    workingFiles: getWorkingFiles(topicId)
  });
  await fs.writeFile(contextPath, content);
}

// Load target topic on boot
async function loadPendingContext() {
  const pending = readJSON('tsvc/pending-reset.json');
  if (!pending) return null;
  
  const contextPath = `tsvc/contexts/${pending.targetTopic}.md`;
  const context = await fs.readFile(contextPath, 'utf8');
  
  await fs.unlink('tsvc/pending-reset.json');
  return { topic: pending.targetTopic, context };
}
```

The boot sequence runs at every session start:

```bash
#!/bin/bash
# tsvc-boot.sh — runs on every session start

PENDING="tsvc/pending-reset.json"

if [ -f "$PENDING" ]; then
  # Post-switch boot: load pending topic context
  TARGET=$(jq -r '.targetTopic' "$PENDING")
  CONTEXT_FILE="tsvc/contexts/${TARGET}.md"
  
  if [ -f "$CONTEXT_FILE" ]; then
    echo "PENDING_TOPIC_SWITCH"
    cat "$CONTEXT_FILE"
  fi
else
  # Normal boot: load active topic
  ACTIVE=$(node -e "const i=require('./tsvc/topic_files/index.json'); console.log(i.activeTopic)")
  echo "NORMAL_BOOT"
  cat "tsvc/contexts/${ACTIVE}.md"
fi
```

---

## Topic Spawn: Mid-Conversation Splits

Sometimes you're deep in a topic and realize the last twenty exchanges have been their own thing. They deserve their own isolated context going forward. TSVC supports this with **Topic Spawn**.

```
User: "This is a new topic — create 'GPU Troubleshooting'"
         │
         ▼
┌─────────────────────┐
│ LLM: scan exchanges │──▶ "Line 47 is where GPU talk started"
│ for semantic boundary│
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ tsvc-spawn.sh       │──▶ Create topic, move lines 47+,
│ "GPU Troubleshoot"  │    update counts, refresh contexts
│ 47                  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Self-reset           │──▶ Next message → fresh session
│ (delete + wait)      │    with new topic loaded
└─────────────────────┘
```

**Protocol:**
1. **Trigger**: Explicit user request only. No auto-detection (too many false positives; assessed and backlogged)
2. **Semantic boundary**: LLM scans `conversation.jsonl` backwards to find where the new discussion semantically started. Judgment call — no heuristic reliably detects topic drift inflection points
3. **Execute**: `tsvc-spawn.sh "<title>" <from_line>` — creates the new topic and **moves** exchanges (not copies). Exchange ownership is exclusive; one topic per exchange
4. **Switch**: Normal self-reset fires → next message lands in the new topic

Topic Spawn makes four operation types total: create, switch, close, **spawn**.

---

## Per-Topic State Management

Each topic maintains a living `where-are-we.md` file with structured sections managed by `tsvc-state.sh`:

```markdown
## Key Facts
- Repo: github.com/MouseRider/skills-tsvc
- Whisper prompt: "tsvc, tsvc-manager, tsvc-boot, tsvc-spawn, OpenClaw, whisper_prompt"

## In Progress
- [ ] Write Show HN post

## Pending Notifications
- [sub-agent:1ab2c3] tsvc-spawn.sh implementation complete

## Recently Completed
- [x] tsvc-vocab.sh v1 merged

## Next Actions
- Review spawn edge cases
```

`tsvc-state.sh` operations: `show`, `append`, `complete`, `finalize`, `clear-notifications`. The first thing loaded when a topic resumes is this file — it answers "where were we?" without requiring the agent to re-read conversation history.

---

## Topic-Scoped Transcription Vocabulary

Voice input introduces a domain-specific transcription problem. Whisper's accuracy degrades when it encounters specialized terms it hasn't seen in training. A static global vocabulary causes the opposite problem: terms from unrelated topics interfere with each other.

TSVC solves this with per-topic transcription vocabulary:

```
┌────────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Topic Switch /  │────▶│  tsvc-vocab.sh    │────▶│ active-      │
│ Boot            │     │  sync             │     │ whisper-     │
└────────────────┘     └──────────────────┘     │ prompt.txt   │
                                                 └──────┬───────┘
                                                        │
                       ┌──────────────────┐             │
  Audio message ──────▶│ tsvc-transcribe   │◀────────────┘
                       │ .sh (CLI wrapper) │
                       │ → OpenAI Whisper  │
                       │ + topic vocabulary│
                       └──────────────────┘
```

Each topic's `where-are-we.md` Key Facts section includes a `Whisper prompt:` field. On topic boot or switch, `tsvc-vocab.sh sync` updates `active-whisper-prompt.txt`. `tsvc-transcribe.sh` passes this to the Whisper API as the `prompt` parameter.

Result: when you're in the Trading topic, Whisper knows "iron condor" and "theta decay." When you're in DevOps, it knows your infrastructure codenames. Neither vocabulary pollutes the other.

This mirrors the problem Anthropic addressed in their 2026 voice validation work for Claude Code — domain-specific vocabulary as a first-class concern for voice-to-agent pipelines. TSVC arrives at the same conclusion independently, with the added dimension of per-topic dynamism.

---

## Async Result Routing

Sub-agents complete work asynchronously, often while the user is in a different topic. Without routing, results arrive in the wrong context.

TSVC routes sub-agent results via board task `topic:` tags:

1. Sub-agent completes → result arrives in main session
2. Plugin Phase 0 detects it's a sub-agent message (not user input)
3. Strategy C routing: extract `[task:task_ID]` → board tag lookup, or keyword match against topic titles
4. If topic is **paged** → file as notification in that topic's `where-are-we.md`
5. If topic is **active** → pass through for normal processing

When the user returns to the paged topic, `where-are-we.md` shows the pending notification. Nothing is lost because the active topic was somewhere else.

---

## Unified Operations Logging

All TSVC scripts log to `tsvc/logs/tsvc-ops.log` via a shared `tsvc-log.sh` helper:

```bash
# Every script sources this
source tsvc/scripts/tsvc-log.sh

tsvc_log INFO "tsvc-boot" "Normal boot, active topic: ${ACTIVE_TOPIC}"
tsvc_log INFO "tsvc-switch" "Switch initiated: ${FROM} → ${TO}"
tsvc_log WARN "tsvc-spawn" "Boundary detection returned line 0, defaulting to 10"
```

Format: `[2026-03-07 14:23:01 PST] [INFO] [tsvc-boot] Normal boot, active topic: trading`

When something breaks, the full event trace is one command: `tail -100 tsvc/logs/tsvc-ops.log`.

---

## Decision Dependency Tracking

Standard context systems store *what* was decided. TSVC also stores *why*.

The core insight from the development process: compaction doesn't just lose conversations, it loses causality chains. An agent that remembers "we chose approach X" without remembering "because Y ruled out approaches A and B" will eventually contradict past decisions without realizing it.

Each decision in TSVC can carry dependency links:

```javascript
// Log a decision with its dependency chain
tsvc-manager.js decision <topic_id> \
  "Use Delete+Wait for session reset" \
  --depends-on "sessions_send deadlocks within active turn"

// Query the full chain
tsvc-manager.js chain <topic_id> <decision_id>
// Output: dec_47eb → "sessions_send deadlocks"
//              └→ dec_b1d9 → "delete+wait confirmed working"
//                       └→ dec_f0f2 → "final self-reset approach"
```

On context reload after a switch, the dependency chain renders with the root decision's full reasoning path. The agent doesn't need to reconstruct "why did we do this" from fragments — it's preserved explicitly.

This is the highest-ROI component in the system. It directly prevents the failure mode where an agent "improves" something that was deliberately built a specific way for reasons that are no longer visible.

---

## Sentiment Hygiene

Memory that survives context switches should be operationally useful, not emotionally colored. The memory protocol filters emotional valence before facts hit persistent storage:

- "Alex seemed frustrated with the Docker setup" → **dropped or rewritten**
- "Docker setup is blocked on X" → **kept**
- "This was a rough debugging session" → **dropped**
- "Root cause: race condition in tsvc-boot.sh line 47" → **kept**

The mood doesn't follow you into the next session; the operational fact does. Over time this meaningfully reduces context pollution — persistent memory that describes feelings instead of facts is noise.

---

## Comparison to Prior Work

**MemGPT** addresses the context window size problem through virtual memory paging — offloading older content to external storage and retrieving it on demand. It doesn't address the isolation problem. All topics still share the same paged pool.

**Mem0** is excellent for persistent fact extraction across sessions. It handles "remember that I prefer X" and "what did I decide about Y last month" well. It doesn't handle conversational thread isolation — you can't give financial discussions and infrastructure debugging separate, clean windows.

**ACON** (Zhang et al., 2025) uses observation masking to reduce token usage. It operates within a single context window and doesn't provide cross-topic isolation.

TSVC doesn't replace any of these — it operates at a different layer. Mem0 facts live in the shared facts layer (accessible to all topics). ACON-style filtering applies within each topic's exchange logger. MemGPT's paging could theoretically sit beneath the per-topic context files.

The key gap TSVC fills: no existing system gives each conversation domain its own isolated context that survives independently across sessions.

**Related work (2026):**
- **Anthropic Voice for Claude Code** — validates domain-specific vocabulary as a first-class concern for voice-to-agent pipelines. TSVC's per-topic transcription vocabulary independently addresses the same problem with the added dimension of per-topic dynamism.
- **Claude Memory Import** (Anthropic, 2026) — cross-platform memory portability for switching between AI assistants. Solves horizontal context migration (user switching tools). TSVC solves vertical context isolation (one agent, many topics). The intersection — portable topic export/import — is a natural future extension.

---

## Results

Production operation on a Telegram-based persistent assistant (running Claude via OpenClaw on an Intel NUC):

Before TSVC: 8.5 MB session file, 3,140 lines, 21 global compactions, zero topic isolation.

After TSVC:
- Per-topic session files: 10-340 KB (versus 8.5 MB global)
- Compactions per topic: **0** — completely eliminated
- Topic switches: **126 recorded**
- Active topics: **12**
- Exchanges tracked: **3,735**
- Switch failure rate: **<1%** (one anomalous 16-minute switch, under investigation)
- Median switch time: **31 seconds** (V3/current — down from 140s in V2, 77% improvement)
- Context load on switch: 10-85 KB (versus 8.5 MB)
- Load-to-first-reply: consistently 11-19 seconds

The zero compaction number is the key result. Compaction was caused by context rot from topic mixing. TSVC eliminates the cause, so the symptom disappears.

---

## Portability

TSVC is a pattern. The implementation is OpenClaw-specific in one place: the session reset mechanism. Everything else is portable.

**Fully portable (zero changes):**
- Topic context files (markdown on disk)
- Topic index (`index.json`)
- Topic detection script (standalone Node.js)
- Decision dependency tracking (stored in context files)
- Context save/load logic

**Platform-specific (adapt per harness):**
- Session reset: how you clear conversation history and start fresh
- Event hooks: where you intercept messages before the LLM
- Context injection: how topic context enters the system prompt
- Boot hook: how you run initialization code on session start

The `template/` directory in the repo contains `tsvc_adapter.py` — a Python base class with three abstract methods:

```python
class TSVCAdapter(ABC):
    @abstractmethod
    def detect_topic(self, message: str, topics: List[Topic]) -> Optional[Topic]:
        """Return matched topic or None if no switch needed."""
        pass
    
    @abstractmethod  
    def reset_session(self, pending_context: dict) -> None:
        """Clear conversation history and prepare for fresh session."""
        pass
    
    @abstractmethod
    def inject_context(self, topic_context: str) -> None:
        """Make topic context available at session start."""
        pass
```

Any agent harness with file system access, a session reset mechanism, and a startup hook can run TSVC.

Known-compatible platforms: OpenClaw (production), Claude Code / Codex (AGENTS.md + file system), Cursor / Windsurf (rules files + session management), LangChain / LangGraph (checkpointing as natural reset points).

---

## What's Missing

**Semantic thread detection.** Current context loading grabs the last N exchanges. A smarter approach would detect where the current *sub-thread* started using time gaps and keyword shift heuristics, then load the semantically relevant exchanges rather than just the most recent ones.

**The 16-minute anomaly.** One switch took 954 seconds. This is likely session file locking during concurrent cron activity, but it's unconfirmed. For a production system serving multiple users, this tail latency matters.

**Lobster integration.** The switch pipeline has a Lobster-orchestrated version sitting alongside the bash implementation. It hasn't proven cleaner in practice yet.

---

## Getting Started

```bash
git clone github.com/MouseRider/skills-tsvc
# See docs/integration.md for setup
# See template/ for adapter to your platform
# See docs/architecture.md for full design
```

The `docs/dev-journal.md` contains the complete development history — every design decision, dead end, and lesson learned. If you're building something similar, start there.

---

*TSVC is production software running in a real deployment. The numbers are from real use. It works. It has rough edges. The repo documents both.*

*Built Feb 25 – Mar 8, 2026. Pull requests welcome.*
