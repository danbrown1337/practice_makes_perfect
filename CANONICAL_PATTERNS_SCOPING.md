# CANONICAL_PATTERNS_SCOPING.md

## Purpose
Define how `CANONICAL_PATTERNS.md` applies to this repository.

The canonical file contains both hard Domino/AICE platform requirements and reference patterns for agent-shaped applications. These are not universally interchangeable.

## Categories

### Platform-Mandatory
Binding where the underlying technology is used.

Examples:
- required AAD authentication / no API keys;
- approved credential and bearer-token patterns;
- mandatory deployment/security constraints;
- required environment integration;
- approved tracing/logging interfaces when enabled by contract.

### Capability-Conditional
Apply only when the capability actually uses the relevant shape.

Examples:
- LangGraph typed state;
- runtime context schemas;
- graph-node conventions;
- tool binding;
- conditional routing;
- agentic entry-point conventions;
- MCP;
- streaming.

Do not introduce these capabilities solely to satisfy the reference pattern.

A deterministic pipeline is not non-compliant because it lacks dynamic routing, tool loops, or an LLM-first entry point.

### Reference / Starting Point
Defaults or examples, not universal requirements.

Examples may include:
- chunk sizes;
- overlap values;
- FAISS as a starting vector store;
- model names;
- temperature defaults;
- sync/async dual implementations when not actually required.

## Repository-Specific Architecture
For existing AI Engineering Factory architecture:
- `ARCHITECTURE.md` defines current repository-specific design;
- `DESIGN_DECISIONS.md` records active/reserved decisions;
- `architecture.instructions.md` governs general design and future work.

Do not use a capability-conditional canonical pattern to override an active repository-specific decision.

## Toolchain Migration
If canonical guidance names preferred tooling such as Python version, `uv`, `pyproject.toml`, Ruff, or a packaging layout, do not smuggle a toolchain migration into an unrelated correctness pass.

Dependency pinning, packaging migration, Python-version migration, and build-system changes should be deliberately sequenced unless explicitly in scope.

## Conflict Review Rule
When a canonical pattern appears to conflict with current code:
1. classify it as platform-mandatory, capability-conditional, or reference;
2. inspect `ARCHITECTURE.md` and `DESIGN_DECISIONS.md`;
3. determine whether divergence is deliberate;
4. surface the conflict before modifying architecture.

Do not assume every divergence is a defect.
