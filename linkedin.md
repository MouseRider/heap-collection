# I built this because my AI assistant kept forgetting what we were talking about

One agent, not twelve.

That's the design choice I made when I built my personal AI assistant. Not a swarm of specialized agents — a Finance Agent, a DevOps Agent, a Research Agent — but one persistent agent who knows everything: my finances, my infrastructure, my family situation, my side projects. One conversation thread. Full continuity. The AI as a genuine executive assistant, not a fleet of tools you have to drive yourself. When you need parallelism, the agent handles that in the background. From your end: one entity, everything remembered.

That choice creates a specific problem the swarm model doesn't have. When one agent accumulates context across many domains, topics bleed into each other. Six months in, my agent's session was 8.5 MB, had been compacted 21 times, and every domain was contaminating every other. Ask about financial strategy, get half an answer plus stray Docker thoughts. "Context rot" — Chroma Research named it in 2025 — is the cost of multi-domain continuity in a single context window.

So I applied a concept that's been working since the 1960s: treat conversation topics like OS processes. Each topic gets its own isolated context file — decisions, open items, recent exchanges, live status. When you switch topics, current state saves to disk, target context loads, session resets cleanly. Zero topic mixing. 21 global compactions became 0 per-topic compactions. Context per topic dropped from 8.5 MB to 10-340 KB. 126 switches across 12 active topics, less than 1% failure rate.

The system is open source: **github.com/MouseRider/skills-tsvc** — full implementation for OpenClaw, a Python adapter for other platforms, and a complete dev journal including every failed approach. If you're building persistent AI assistants and care about the personal assistant model, this is the architecture problem nobody else is solving.
