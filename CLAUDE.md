# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 1. Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan immediately — don't keep pushing
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

## 2. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution

## 3. Self-Improvement Loop
- After ANY correction from the user: update tasks/lessons. md with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until mistake rate drops
- Review lessons at session start for relevant project

## 4. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

## 5. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes — don't over-engineer
- Challenge your own work before presenting it

## 6. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests — then resolve them
- Zero context switching required from the user
- Go fix failing CI tests without being told how

## Task Management

1. *Plan First*: Write plan to tasks/todo.md with checkable items  
2. *Verify Plan*: Check in before starting implementation  
3. *Track Progress*: Mark items complete as you go  
4. *Explain Changes*: High-level summary at each step  
5. *Document Results*: Add review section to tasks/todo. md  
6. *Capture Lessons*: Update tasks/lessons. md after corrections  

## Core Principles

- *Simplicity First*: Make every change as simple as possible. Impact minimal code.
- *No Laziness*: Find root causes. No temporary fixes. Senior developer standards.


## What this project is

**KermitKernel** is a Kubernetes operator (written in Go) for deploying and managing fleets of AI agents with deep security isolation. Each agent runs in a hardened sandbox ("FrogFort") using Kata Containers (micro-VMs), with an Envoy/Wasm sidecar for egress/ingress filtering.

The project is in early architecture/design phase. The target planned structure from README.md is:

```
kermit-kernel/
├── api/v1alpha1/       # CRD Go structs (Agent, AgentFleet, Gateway)
├── charts/             # Helm charts (kermit-operator, frogfort-runtime, kermit-dash)
├── cmd/                # Binary entrypoints (operator, gateway, sidecar)
├── config/             # Kustomize manifests + CRD samples
├── internal/           # Private logic (controller, graph, session, inference)
├── images/             # Dockerfiles for hardened agent images
├── pkg/                # Reusable SDK for agents
└── ui/                 # Next.js dashboard (Kermit Dash)
```

## Custom Resource Definitions

Two CRDs are defined. See `config/samples/` for full examples.

**Agent** (`kermit.io/v1alpha1`): A single agent pod with:
- `spec.agentType`: `unit | sequential | loop | parallel | cron`
- `spec.sandbox`: runtime class (kata-clh/kata-qemu), resources, egress allowlist, DLP toggle
- `spec.intelligence`: primary/fallback model, temperature, priority (`performance | cost | privacy`)
- `spec.mesh`: communication protocols (MCP, A2A), tool connectors, collaboration flags

**AgentFleet** (`kermit.io/v1alpha1`): Deployment-like wrapper for scaling homogeneous agent pools, with KEDA autoscaling based on Prometheus queue depth metrics.

## Architecture: key design decisions

- **Isolation layer**: Kata Containers (kata-clh or kata-qemu runtime class) — each agent gets its own micro-VM Linux kernel.
- **Sidecar security**: Envoy proxy with Wasm filters for real-time content filtering (DLP) on every agent pod.
- **Inference routing**: LiteLLM proxy unifies local (vLLM/Ollama) and cloud (OpenAI, Anthropic) under one interface. Routing based on privacy/cost/performance priority.
- **Agent graph**: ArcadeDB or FalkorDB stores fleet metadata and immutable audit log.
- **Session persistence**: PostgreSQL + pgvector; JuiceFS or MinIO for shared workspaces between agents.
- **Service mesh**: Istio (Ambient Mesh) + Dapr for A2A and MCP agent communication.
- **Policy enforcement**: Kyverno for Kubernetes-native policy (preferred over OPA for YAML-native maintenance).
- **Frontend**: Next.js + shadcn/ui + React Flow for live fleet graph visualization.

## Operator (Go) conventions

The operator follows standard controller-runtime patterns:
- CRD types in `api/v1alpha1/` with `+kubebuilder:` markers
- Reconciliation logic in `internal/controller/`
- Binary entrypoint in `cmd/operator/`
- Use `controller-runtime` and `client-go` (not raw API calls)

## Makefile targets (to be implemented)

The Makefile skeleton targets `BINARY_NAME=kermit`. Standard kubebuilder targets to add:
- `make generate` — run controller-gen for deepcopy/CRD manifests
- `make manifests` — generate CRD YAML from Go markers
- `make test` — run unit tests
- `make docker-build` — build operator image
- `make deploy` — apply to cluster via Kustomize

## Data flow summary

User submits workflow → Operator queries graph for agent plan → Creates FrogFort pods (Kata) → Session Manager mounts JuiceFS volume → Agent calls Kermit Gateway → Gateway routes inference (local vs cloud based on sensitivity) → Envoy sidecar scans LLM response → Result returned, audit logged in ArcadeDB.
