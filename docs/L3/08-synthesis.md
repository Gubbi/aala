# L3 — Synthesis Components

For the container framing, see [`L2/08-synthesis.md`](../L2/08-synthesis.md). Synthesis composes multiple capabilities into a single response with provenance — Q&A, ADR drafts, sequence diagrams, walkthroughs, comparisons.

## Component diagram

![L3 — Synthesis](../diagrams/synthesisInternals.png)

## Component reference

| Component | Responsibility | Internal state | Emits / consumes |
|---|---|---|---|
| **Intent Parser** | Classifies the incoming intent into a generator kind. May be rule-based, LLM-driven, or hybrid. | None. | Calls LLM Gateway with `aala.synthesis` for LLM-driven classification. |
| **Generator Registry** | Pluggable generators: `qa`, `adr`, `sequence_diagram`, `walkthrough`, `comparison`, plus any deployment-specific. Each declares its capability floor and graceful enhancements. | Registry of installed generators. | Read by Capability Composer. |
| **Capability Composer** | For a chosen generator, decides which capabilities to invoke and in what order based on what's wired up. Drives the per-call composition. | None. | Reads Capability Registry from [Orchestration](./05-orchestration.md). |
| **Audience Adapter** | Adjusts framing for PM / Designer / Engineer / Exec audience. | None (configuration only). | Drives prompt shaping into LLM calls. |
| **Conflict Surfacer** | Detects unresolved conflicts in retrieved atoms; expresses them in the response ("current view: X / conflicting evidence: Y") rather than silently picking a side. | None. | In: retrieved atoms. Out: framed response fragments. |
| **Confidence Temperer** | Adjusts language certainty based on retrieved-atom confidence scores. | None. | In: atoms + draft response. Out: hedged response. |
| **Provenance Attacher** | Attaches atom IDs, navigation paths, sources as citations. Every response includes provenance. | None. | Out: response with citations. |
| **Faithfulness Checker** | Verifies every claim in the response is grounded in retrieved atoms. Configurable: soft warning vs. hard block. | None. | Calls LLM Gateway with `aala.faithfulness_judge`. Out: verdict. |
| **Diagram Generator** | For structural output (sequence diagrams, C4-style diagrams), composes from atoms' references and emits a rendered form (mermaid, likec4, etc.). | None. | Calls LLM Gateway with `aala.synthesis`. Out: diagram source. |

## Internal flow — query

```mermaid
sequenceDiagram
    participant Caller as Caller (Orchestration)
    participant Intent as Intent Parser
    participant Reg as Generator Registry
    participant Comp as Capability Composer
    participant Aud as Audience Adapter
    participant Atoms
    participant Nav as Hierarchical Nav (opt)
    participant Blast as Blast Radius (opt)
    participant Proj as Projection
    participant LLM as LLM Gateway
    participant Surf as Conflict Surfacer
    participant Temp as Confidence Temperer
    participant Prov as Provenance Attacher
    participant Faith as Faithfulness Checker
    participant Diag as Diagram Generator

    Caller->>Intent: query(intent, hints)
    Intent->>LLM: aala.synthesis (classify)
    LLM-->>Intent: generator kind
    Intent->>Reg: pick generator
    Reg->>Comp: generator + floor + optional uses
    Comp->>Aud: audience hint
    opt Hierarchical Nav present
        Comp->>Nav: navigate(axis, path)
        Nav-->>Comp: candidate atoms
    end
    opt Blast Radius present
        Comp->>Blast: get_report / unresolved
        Blast-->>Comp: report excerpt
    end
    Comp->>Atoms: read referenced atoms
    Comp->>Proj: read aligned prose (optional, for anchoring)
    Comp->>LLM: aala.synthesis (compose)
    LLM-->>Comp: draft response
    Comp->>Surf: surface conflicts
    Surf->>Temp: temper confidence
    Temp->>Prov: attach provenance
    opt structural output (ADR, diagram)
        Prov->>Diag: structural compose
        Diag->>LLM: aala.synthesis (diagram)
        LLM-->>Diag: diagram source
    end
    Prov->>Faith: verify
    Faith->>LLM: aala.faithfulness_judge
    LLM-->>Faith: verdict
    Faith-->>Caller: response + provenance + capability list consulted
```

## Variation points

| Variation | Examples |
|---|---|
| Generator set | Q&A only (smallest); +ADR; +sequence diagram; +walkthrough; +comparison; +deployment-custom. Each independently pluggable. |
| Intent classification | Rule-based (cheap, brittle); LLM-driven (more accurate, more expensive); hybrid. |
| Faithfulness mode | Off (no check); warn (flag confabulation but return); block (fail request on unsupported claim). |
| Audience adaptation | Single voice; per-role tuned (PM / Designer / Engineer / Exec). |
| External-source set | None; web search; vendor MCPs; internal company sources. |
| Response cache | None; in-process; persistent. |
