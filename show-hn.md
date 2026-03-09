# Show HN: My AI Agent Kept Forgetting What We Were Talking About, So I Built Topic-Scoped Virtual Context

I've been running a persistent AI assistant — I call him my agent — for a few weeks. He lives on an Intel NUC in my apartment, talks to me via Telegram, does research, tracks my finance studies, helps with infrastructure. One agent. Everything.

The problem started within days.

---

## One Agent for Everything (And Why That's the Right Call)

Before I explain context rot, let me explain why I have this problem in the first place.

If you follow the agentic AI space right now, everyone is pushing swarms. Every blog post, every framework, every tutorial: build a Finance Agent, an Infrastructure Agent, a Projects Agent, a Research Agent. Specialized agents for everything. It sounds clean on a whiteboard.

The problem is that *you* become the orchestrator. You switch between agents. You remember which agent knows what. You manually carry context from one conversation to the next when your finance discussion reveals something about your infrastructure setup. The cognitive overload doesn't disappear — it moves from the AI to you. The person the AI was supposed to help is now managing a team of AIs.

I chose the personal assistant model instead. One agent. Everything. He knows my finances, my infrastructure, my family situation, my side projects. He remembers decisions I made last week and why I made them. When tasks need parallelism, he spins up specialized sub-agents behind the scenes and monitors them — but from my perspective, I'm talking to one entity who knows all of it and handles the delegation himself. That's a qualitatively different experience. An AI system that actually runs your life, not one that requires you to manage the AIs.

This is the model TSVC is designed for. And it's exactly why context rot becomes a problem.

---

## The Problem Has a Name: Context Rot

The agent is built on Claude via OpenClaw, a self-hosted agent framework. Persistent agent. Same session. And that session was... accumulating.

By late February — just weeks into daily use — the session file was **8.5 MB**. Three thousand one hundred forty lines of conversation. The agent had experienced **21 global compactions** — the point where the context window fills and the entire conversation history gets summarized into a lossy blob. Twenty-one times. In weeks, not months.

You know what 21 compactions does to a conversation history that spans financial strategies, family planning, infrastructure debugging, and AI development work? It turns it into soup. A degraded average of everything. "Context rot" — Chroma Research put a name to it in 2025, and the description is perfect: the agent's memory isn't lost, it's blended into something that's no longer useful for any specific domain. And this happened in *days*, not months — that's how fast context rots when you're using your agent for everything, all day.

Talking to the post-compaction agent about our online education course was like talking to someone with amnesia who remembered fragments of every conversation we'd ever had, mixed together. He'd bring up infrastructure details in the middle of a finance discussion. Reference family decisions when I asked about DevOps. Not wrong, exactly. Just... soup.

I wanted to fix this. I'd read MemGPT (the 2023 paper that modeled AI agents on OS virtual memory). I'd looked at Mem0 (good for persistent facts, bad for conversational threads). I'd looked at ACON (observation masking — saves tokens, doesn't solve the mixing).

Everyone treats context as one big pool. Nobody segments by topic.

---

## The OS Metaphor That Actually Helped

Here's the thing I kept coming back to: this is a solved problem in computer science. We solved it in the 1960s.

In 1956, Güntsch imagined automatic memory management. In 1962, the Atlas computer built the "one-level store" — programs don't know where their data physically lives, the OS manages it transparently. In the 1970s, Unix gave each process its own virtual address space.

Each process gets isolation. When the kernel context-switches between processes, it saves the current process state, loads the new one, and the new process has its full address space to itself — not a fraction of a pool degraded by everyone else's activity.

Why don't AI agents do this for conversation topics?

MemGPT used the OS metaphor for memory (virtual memory paging). But memory is only half the story. The full mapping is:

- Process → Topic (conversation domain)
- Address space → Topic context (isolated conversation history)
- Scheduler → Topic detector
- Context switch → Topic switch (save current → load target)
- IPC (inter-process communication) → Shared facts (SSoT files that all topics can read)

We built that. We called it TSVC: Topic-Scoped Virtual Context.

---

## What We Actually Built

The core is simple. Five files. A few shell scripts. No new runtime.

`tsvc-manager.js` handles topic lifecycle: create, switch, save, load, close. Each topic gets a context file on disk — recent exchanges, active decisions, open items, working files. When you switch topics, the current context saves to disk, the target context loads, and the agent starts fresh with only that topic's history in its window.

The topic awareness layer is always loaded — a lightweight index (~3k tokens) with topic IDs, titles, last-active timestamps, and one-line summaries. Enough for the agent to know what topics exist without loading any of them.

The topic detector runs as a gateway plugin — before the LLM ever sees your message. Fuzzy string matching on topic titles, then semantic classification fallback. Zero LLM tokens spent on "is this a topic switch?" decisions. A bash script can parse "switch to finance topic" in 2ms for free.

But here's where it gets weird. Because topic switching requires a session reset. And session reset turned out to be... complicated.

---

## The Self-Reset Deadlock (A Story in Four Failed Attempts)

The first instinct for "reset the session" was obvious: `sessions_send("/reset")` to my own session key. Tell the agent to send a reset command to itself.

Deadlock.

The gateway locks the session during an active turn. The reset message can't be delivered until the turn completes. The turn is blocked waiting for the reset. Classic dining philosophers situation, except with a Claude instance and a Node.js gateway.

Okay, fire and forget. `sessions_send("/reset", { timeoutSeconds: 0 })`. Don't wait for acknowledgment, just send it and move on.

This... almost worked. The error was cosmetic — the old session dying before it could respond. Noisy but functional. I shipped it. Then compaction hit and I forgot the script was tested and working. Post-compaction, the agent saw the code, thought "this could be improved," changed the `openclaw message send` to `openclaw agent --message /reset` because it looked cleaner.

`openclaw agent` is synchronous. It hangs forever.

I reverted. Compaction happened again. I "improved" it again. This cycle repeated four times in one afternoon. Four rewrites between the same two approaches, each time losing the context that message send was the working one. I added a rule to AGENTS.md: "NEVER modify a working script." The rule needs to survive compaction. Script tombstones.

Eventually we discovered the actual deadlock-proof solution: **Delete + Wait**.

Instead of trying to reset from within the session, you just delete the session files. Background script, 2-second delay, then the files are gone. The current turn completes normally. The user's next message arrives. There's no session to attach to. A fresh one spawns. The boot sequence picks up a `pending-reset.json` file that was written before the delete, loads the target topic context, and the agent wakes up in the new topic as if he'd been there all along.

"My AI solved its own deadlock by scheduling its own death on a 2-second timer."

It's extremely cursed and it works perfectly.

---

## The Exchange Logger Incident

While debugging the reset mechanism, we needed to track which exchanges belonged to which topics — especially since pre-TSVC, all conversations had been jumbled together in one mega-session.

We built `tsvc-exchange-logger.py`: reads the session transcript, detects topic-switch events, appends exchanges to the correct topic's `conversation.jsonl`. Tracks the last-processed line so each run is incremental.

On the first run, the exchange logger looked at the session history, dutifully processed everything it found, and dumped **993 exchanges** into the TSVC Development topic.

Every conversation we'd ever had. Finance. Family. Infrastructure. Education courses. All of it, attributed to TSVC Development, because that's what was active when the logger first ran.

This is permanent. You can't re-attribute historical exchanges. The TSVC Development topic now has 1,820 exchanges, and roughly 993 of them are about something else entirely.

We accepted the corrupted history and moved on. The lesson: run the exchange logger on day one, before the session has history. We learned it on day forty-something.

---

## And Then We Needed to Split Topics Mid-Conversation

After a few weeks in production, a new problem surfaced. Sometimes you're knee-deep in one topic and you realize: this sub-thread has been its own thing for the last twenty exchanges. It deserves its own context. The old workflow was to just leave it mixed in.

So we built **Topic Spawn**.

It works like this: you say "make this a new topic called GPU Troubleshooting." The agent scans the `conversation.jsonl` backwards to find where the GPU discussion semantically started — that's a judgment call, so the LLM does it. Then `tsvc-spawn.sh` creates the new topic and *moves* (not copies) those exchanges there. Exchange ownership is exclusive. Then the normal self-reset fires and your next message lands in the freshly spawned topic.

No auto-detection. It's explicit-only. We debated whether to add auto-suggest (Tier 2 plugin detection) and decided no — false positives on topic spawn would be more disruptive than useful. The detection problem for "this conversation has drifted into a new domain" is harder than "this message is about topic X," and we'd learned our lesson about over-automating.

Topic Spawn makes four operation types total: create, switch, close, spawn.

---

## The Features That Came After the Core

Once the basic infrastructure worked, we kept building. A few things surprised us with how useful they turned out to be.

**Per-topic transcription vocabulary.** I talk to my agent via voice a lot. The problem: Whisper's transcription is worse when it doesn't know your domain. "Drumknott" (a sub-agent name) doesn't appear in any training corpus. Options trading terminology is ambiguous. When I'm in the Trading topic, Whisper should know about "iron condor" and "theta decay." When I'm in DevOps, it should know my infrastructure codenames.

So each topic maintains a `whisper_prompt` in its Key Facts. On topic switch or boot, `tsvc-vocab.sh` syncs the active vocabulary to a shared file, and `tsvc-transcribe.sh` passes it to the Whisper API. Your trading topic doesn't pollute DevOps transcription with financial terms that sound like other things.

**Async result routing.** Sub-agents complete their work asynchronously. The problem: I might be in the Trading topic when a sub-agent finishes work that belongs to the TSVC Development topic. Without routing, that result arrives in the wrong context and either gets confused with trading discussion or gets ignored.

The solution: board tasks carry `topic:` tags. When a sub-agent result arrives, the plugin checks the tag and routes it to the right topic — filing it as a pending notification in that topic's `where-are-we.md` if the topic is paged. When I switch back to TSVC Development, the notification is there waiting.

**Per-topic state management.** Each topic now has a living `where-are-we.md` — Key Facts, In Progress, Pending Notifications, Recently Completed, Next Actions. Managed by `tsvc-state.sh`. It's the thing the agent reads first when a topic loads. "Where were we?" has an actual answer now.

**Sentiment hygiene.** Memory protocol got an upgrade: emotional valence gets filtered out before facts hit persistent storage. "Alex seemed frustrated with the Docker setup" becomes "Docker setup is blocked on X." The mood doesn't follow you across sessions; the operational fact does. This sounds minor but it meaningfully reduces context pollution over time.

**Unified ops logging.** All TSVC scripts log to `tsvc/logs/tsvc-ops.log` via `tsvc-log.sh`. Every switch, boot, state change, routing event — timestamped, tagged, one place. When something breaks, `cat tsvc-ops.log` shows the full event trace. Pre-logging, debugging was reconstructed from Telegram history. Post-logging, it's just grep.

---

## Does It Actually Work?

Here's the honest number dump.

Before TSVC:
- Session file: 8.5 MB
- Session lines: 3,140
- Global compactions: 21
- Topic isolation: zero

After production use:
- Per-topic session files: 10-340 KB
- Compactions per topic: **0**
- Topic switches recorded: **126**
- Switch failure rate: **<1%** (one anomalous 16-minute switch, cause unknown, everything else clean)
- Median switch time: **31 seconds** (V3, current — was 140s in V2)
- Context loaded on switch: 10-85 KB (versus 8.5 MB)
- Active topics: **12**
- Exchanges tracked: **3,735**

The system went through three versions. V1 (manual self-reset) deadlocked. V2 (agent-driven detection + background reset script) worked but had a median of 140 seconds. V3 (gateway plugin, deterministic detection) cut that to 31 seconds — a 77% improvement. Moving detection from the LLM to a script eliminated per-switch token costs entirely.

Zero compactions per topic is the number I care about most. Compaction doesn't happen because context rot doesn't happen. Each topic session stays small because it only contains that topic's conversation. There's nothing to compact.

The 31-second switch time is fine for my workflow. I talk to my agent via Telegram. I send a message, switch topics, he dies and reboots in the background, my next message gets a fresh context. I don't see the gap. "Seamless" means no visible interruption, not zero latency.

The anomalous 16-minute switch matters and is unresolved. It happened once out of 126 switches. My guess is session file locking during concurrent cron activity. We need to confirm it. If you're building something like this for users, the anomaly is what breaks trust, not the median.

---

## What We Learned That Surprised Me

**Decision memory outlives conversation memory.** The worst part of compaction wasn't losing conversations — it was losing the *why* behind decisions. We made a choice about infrastructure, the reasoning got compacted, the agent in future sessions saw the decision without context, made a different choice, and I had to re-litigate everything.

We added a decision dependency chain: `dec_A depends_on dec_B depends_on dec_C`. When reloading context after a switch, you don't just get isolated facts — you get the reasoning chain. "Why did we decide X?" walks the chain, doesn't require guessing. This was the highest-ROI addition to the whole system.

**Plugin beats prompt for deterministic tasks.** Moving topic detection from "tell the LLM to watch for switches" to "run a deterministic script before the LLM sees the message" improved accuracy and eliminated tokens. The agent once switched to "Finance" when I said "switch back to tsvc topic" — because the previous work had been on the TSVC switching mechanism *inside* the Finance context. Semantic confusion. The fix was fuzzy title matching: "tsvc topic" → "TSVC Development" (exact substring match) wins before semantic analysis runs.

**Telemetry changes what you build.** Without numbers, switch time "felt slow." With t0/t1/t2 timestamps: reset+load takes 35-80 seconds (I/O, optimizable), load-to-reply takes 11-19 seconds (LLM overhead, fixed). Any optimization without the breakdown would have been guesswork.

---

## The Part That Still Feels Weird

Topics are global — workspace-level, not per-channel. If we're talking about online education course on Telegram and I hop over to webchat, the agent is still on Education. Same brain, different phone.

That's the right default. I'm one person, he's one agent, topics don't change because I switched apps.

There are still rough edges. The exchange logger will lie about history once (we lived through that). The 16-minute anomaly needs investigation. Semantic thread detection — loading context based on what we were *actually* talking about, not just the last N exchanges — would be smarter than the current time-gap heuristics. The Lobster-orchestrated switch pipeline is an experiment that hasn't proven itself over the bash implementation yet.

But zero compactions per topic, 126 clean switches, 12 active topics, and context files that are 10-340 KB instead of 8.5 MB — that's real. It works in production on real conversations about real things.

---

## Architecture in One Paragraph

Every topic gets a context file on disk: compressed decisions, open items, working files, recent exchanges. A lightweight topic index (~3k tokens) stays always-loaded so the agent knows what exists. A gateway plugin intercepts every message before the LLM sees it, runs deterministic fuzzy matching, fires a switch if needed. Switch sequence: log exchanges → write pending-reset.json → delete session files → die. On the user's next message: fresh session spawns → finds pending-reset.json → loads target topic → replies with full topic context. The "IPC layer" — facts shared across all topics — lives in SSoT files that every topic can read. Four operation types: create, switch, close, spawn.

File-based. No custom runtime. Adapts to any agent harness with file system access, some form of session reset, and a boot hook.

---

## How to Port This

TSVC is a pattern, not a framework. The portable parts: topic context files (markdown on disk), topic index (JSON), topic detection script (Node.js, runs standalone), context save/load logic. The platform-specific parts: the session reset mechanism (our Delete+Wait is OpenClaw-specific) and the context injection hook (how you get topic context into the system prompt on boot).

For Claude Code / Codex: AGENTS.md boot + file system access + session restart covers all three requirements. For LangChain / LangGraph: state management + checkpointing provides natural reset points. For Cursor: rules files + workspace context + session management.

The `template/` directory in the repo has a Python adapter (`tsvc_adapter.py`) with three abstract methods: `detect_topic()`, `reset_session()`, `inject_context()`. Subclass it and you're most of the way there.

---

## Try It

github.com/MouseRider/skills-tsvc

README has the full architecture. `docs/dev-journal.md` has the complete war stories including the four-cycle deadlock debugging and the exchange logger massacre. `docs/telemetry-results.md` has all the numbers. `template/` has the Python adapter for non-OpenClaw deployments.

Built Feb 25 – Mar 8, 2026 on an Intel NUC running OpenClaw.

The magnificence is real. The soup is gone.
