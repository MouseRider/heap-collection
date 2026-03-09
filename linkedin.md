# I built this because my AI assistant kept forgetting what we were talking about

One agent, not twelve.

Everyone building with agentic AI right now defaults to the same architecture: a swarm of specialized agents. A Finance Agent, a DevOps Agent, a Research Agent — each expert in its narrow domain. Every blog post, every tutorial, every framework pushes you toward this pattern. It sounds clean on a whiteboard.

But try actually living with it. You become the orchestrator. You context-switch between agents constantly, remember which one knows what, manually carry decisions from one conversation to another. The cognitive overload lands on *you* — the person the AI was supposed to help.

I went the other way. One persistent agent who knows everything: my finances, my infrastructure, my family situation, my side projects. One conversation thread. Full continuity. Not a fleet of tools I have to drive — a genuine executive assistant. When parallelism is needed, *he* delegates to specialized sub-agents behind the scenes. From my end: one entity, everything remembered, everything handled.

That choice creates a specific problem the swarm model doesn't have. When one agent accumulates context across many domains, topics bleed into each other. Within just a few weeks, my agent's session had ballooned to 8.5 MB, had been compacted 21 times, and every domain was contaminating every other. Ask about financial strategy, get half an answer plus stray Docker thoughts. "Context rot" — Chroma Research named it in 2025 — is the cost of multi-domain continuity in a single context window.

So I applied a concept that's been working since the 1960s: treat conversation topics like OS processes. Each topic gets its own isolated context file — decisions, open items, recent exchanges, live status. When you switch topics, current state saves to disk, target context loads, session resets cleanly. Zero topic mixing. 21 global compactions became 0 per-topic compactions. Context per topic dropped from 8.5 MB to 10-340 KB. 126 switches across 12 active topics, less than 1% failure rate.

The system is open source: **github.com/MouseRider/skills-tsvc** — full implementation for OpenClaw, a Python adapter for other platforms, and a complete dev journal including every failed approach. If you're building a persistent AI assistant — one agent that actually runs your life instead of a swarm you have to manage — this is the architecture problem nobody else is solving.
