# I built this because my AI assistant kept forgetting what we were talking about

One agent, not twelve.

Everyone building with agentic AI right now defaults to the same architecture: a swarm of specialized agents. A Finance Agent, a DevOps Agent, a Research Agent — each expert in its narrow domain. Every blog post, every tutorial, every framework pushes you toward this pattern. It sounds clean on a whiteboard.

But try actually living with it. You become the orchestrator. You context-switch between agents constantly, remember which one knows what, manually carry decisions from one conversation to another. The cognitive overload lands on *you* — the person the AI was supposed to help.

I went the other way. One **personal assistant agent** — not an expert in everything, but aware of everything. He knows enough about each domain of my life to hold a conversation, make connections across topics, and query a shared knowledge base when he needs details. He's the coordinator, not the specialist.

The specialized work? That gets delegated to **dedicated specialist agents** — a finance agent, an infrastructure agent, a research agent. My assistant creates tasks for them, monitors their progress, receives their reports, communicates with them directly. They update their status, they deliver results, they stay in their lane.

And when I need to go deep — really deep — into a specific domain, I *can* talk to a specialist directly. Dive into options strategy with my finance agent. Debug infrastructure with my DevOps agent. But only when I actually need to. Not as my default workflow. 95% of the time, I talk to one entity who handles everything on my behalf.

That choice creates a specific problem the swarm model doesn't have. When one agent accumulates context across many domains, topics bleed into each other. Within just a few weeks, my agent's session had ballooned to 8.5 MB, had been compacted 21 times, and every domain was contaminating every other. Ask about financial strategy, get half an answer plus stray Docker thoughts. "Context rot" — Chroma Research named it in 2025 — is the cost of multi-domain continuity in a single context window.

So I applied a concept that's been working since the 1960s: treat conversation topics like OS processes. Each topic gets its own isolated context file — decisions, open items, recent exchanges, live status. When you switch topics, current state saves to disk, target context loads, session resets cleanly. Zero topic mixing. 21 global compactions became 0 per-topic compactions. Context per topic dropped from 8.5 MB to 10-340 KB. 126 switches across 12 active topics, less than 1% failure rate.

The system is open source: **github.com/MouseRider/skills-tsvc** — full implementation for OpenClaw, a Python adapter for other platforms, and a complete dev journal including every failed approach. If you're building a persistent AI assistant — one coordinator agent that runs your life while specialists handle the deep work — this is the architecture problem nobody else is solving.
