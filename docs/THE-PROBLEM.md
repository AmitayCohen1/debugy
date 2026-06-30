# The problem Debugy solves

> A reminder for us. Whenever we write copy, design a screen, or build a feature,
> it should serve *this* pain. If it doesn't, cut it.

## The pain

When you run AI agents in real workflows — pipelines, ETLs, agentic runs — your
code does **not** run in one place. A single run is smeared across environments:

- It hits **Modal** to run some GPU work (inference, embeddings).
- Then **Render** to run CPU work on a VM.
- Then **Inngest** to orchestrate a step / queue / job.
- Then **Vercel**, where the app/API/frontend is hosted.
- …and many others.

Every one of those has **its own logs, its own dashboard, its own MCP**. So when
something breaks, you (or your agent) are hopping between 5 consoles, 5 logins,
5 MCPs — each showing only one slice of the run. **Nobody has a single view of
the whole pipeline.** The context is fragmented across vendors.

## What we're building

A **centralized log sink**. Every environment in the pipeline POSTs its logs to
one place with one key. Then the agent hits **one** endpoint and gets **full
cross-environment visibility** of the entire run — Modal + Render + Inngest +
Vercel, stitched together, no per-vendor MCP juggling.

Instead of:

> agent → Modal MCP, agent → Render MCP, agent → Vercel MCP, agent → Inngest MCP
> (and manually correlating)

it becomes:

> every service → **Debugy** → agent reads the whole story in one query

## What makes it valuable (not just a log dump)

The magic isn't "one bucket" — it's **correlation**:

- `service` → the **per-environment** view (one tab per environment).
- shared `run_id` / `trace_id` in `metadata` → the **cross-environment** view
  (the full pipeline for one execution, in order, as one connected trace).

## The non-negotiable

It must stay **dead simple**: one key, no SDK, no infra. The pain must be
**crystal clear** on the landing page, in the UI, and in setup. Same trivial
step for every environment. If we ever start looking like Datadog, we've lost.
