# L3 — Blast Radius Components

For the container framing, see [`L2/07-blast-radius.md`](../L2/07-blast-radius.md). Blast Radius answers "what else changes when this atom transitions?" and tracks the answer as humans resolve each impact.

## Component diagram

![L3 — Blast Radius](../diagrams/blastRadiusInternals.png)

## Component reference

| Component | Responsibility | Internal state | Emits / consumes |
|---|---|---|---|
| **Delta Consumer** | Subscribes to Atoms's `changes_since(ref)` stream. Tracks the last consumed `ref`. Filters for transition events relevant to open reports (feeds Iterative Refiner) and high-impact transitions that may trigger automatic analysis. | Per-snapshot consumer ref. | Consumes Atoms events. |
| **Trigger Handler** | When the Delta Consumer surfaces a transition with broad downstream impact (sweeping decisions, classification changes affecting many descendants), MAY initiate a fresh analysis. Configurable per deployment. | Configuration thresholds. | Drives Analyze Pipeline. |
| **Origin Resolver** | First stage of the analyze pipeline. Resolves the input `atom_id` to a canonical atom; rejects if not found. Computes the hypothetical state when supplied. | None. | Reads Atoms. |
| **Cascade Simulation** | Obtains the structural impact set from `Atoms.simulate_transition(atom_id, hypothetical, { depth_limit })` — the read-only dry-run of the one cascade engine (three channels + cross-tree `kind=equivalence` expansion + composition confidence gate, run to fixpoint). Does **not** re-implement the traversal; the structural set therefore matches the real lifecycle cascade by construction. | None. | Calls `Atoms.simulate_transition`. |
| **LLM-Implicit Pass** | For each candidate atom (bounded by subtree scoping), asks: "does this still hold under the proposed transition?" Batched aggressively. Optional and configurable; **additive and advisory** — outside the structural-equivalence guarantee. | None. | Calls LLM Gateway with `aala.blast_implicit`. |
| **Report Builder** | Produces the persisted `BlastReport` with impacts, derived_loss, iteration count, and cascade paths. | None. | Writes to Report Store. |
| **Report Store** | Durable persistence of blast reports. Implementation choice: alongside atoms in the snapshot, or container-internal. | The reports themselves. | Receives writes from Report Builder + Iterative Refiner. Pure reads by `get_report` / `list_reports`. |
| **Iterative Refiner** | Receives `record_resolution` calls. Translates the resolution into `Atoms.update_status`. Updates the report. Optionally re-runs the LLM-Implicit Pass with the new resolution as signal. | None. | Calls Atoms; may call LLM Gateway. |
| **Change Log** | Maintains the ordered, append-only event log for the container. | Event sequence + ref / checkpoint surface. | Emits `ReportOpened` / `ImpactResolved` / `ReportRefined` / `ReportClosed`. Serves `changes_since(ref)`. |

## Internal flow — analyze pipeline

```mermaid
sequenceDiagram
    participant Caller
    participant Origin as Origin Resolver
    participant Sim as Cascade Simulation
    participant Atoms as Atoms
    participant Imp as LLM-Implicit Pass
    participant LLM as LLM Gateway
    participant Build as Report Builder
    participant Store as Report Store
    participant Log as Change Log

    Caller->>Origin: analyze(input)
    Origin->>Sim: atom + hypothetical state
    Sim->>Atoms: simulate_transition(atom_id, hypothetical, depth_limit)
    Atoms-->>Sim: structural impact set (read-only dry-run of the cascade engine)
    opt LLM-implicit pass enabled
        Sim->>Imp: structural set
        Imp->>LLM: aala.blast_implicit (batched)
        LLM-->>Imp: additive non-structural impacts
        Imp-->>Build: additive impacts (advisory)
    end
    Sim->>Build: structural impact set
    Build->>Store: persist BlastReport
    Build->>Log: ReportOpened
    Log-->>Caller: BlastReport
```

## Internal flow — record_resolution

```mermaid
sequenceDiagram
    participant Caller
    participant Refiner as Iterative Refiner
    participant Store as Report Store
    participant Atoms as Atoms
    participant Imp as LLM-Implicit Pass
    participant LLM as LLM Gateway
    participant Log as Change Log

    Caller->>Refiner: record_resolution(blast_id, atom_id, status, rationale, meta?)
    Refiner->>Store: load report
    Store-->>Refiner: report
    Refiner->>Atoms: update_status(atom_id, status, rationale, meta)
    Atoms-->>Refiner: cascade summary
    Refiner->>Store: update impact resolution
    opt iterative refinement enabled
        Refiner->>Imp: re-run with new signal
        Imp->>LLM: aala.blast_implicit
        LLM-->>Imp: refined remaining set
        Imp-->>Refiner: refined impacts
        Refiner->>Store: update report
        Refiner->>Log: ReportRefined
    end
    Refiner->>Log: ImpactResolved
    opt all impacts resolved
        Refiner->>Store: close report
        Refiner->>Log: ReportClosed
    end
    Log-->>Caller: updated BlastReport
```

## Variation points

| Variation | Examples |
|---|---|
| Pipeline depth | Structural-only (from `simulate_transition`, fast and exact); + LLM-implicit pass (additive, advisory, full coverage). |
| Implicit-pass model tier | Top-tier LLM; mid-tier with self-check; small local model for privacy-constrained tenants. |
| Iteration policy | Owned by the cascade engine — `simulate_transition` runs the fixpoint; not a Blast Radius variation. |
| Subtree scoping | Conservative (large candidate set); aggressive (tighter scoping, may miss). |
| Report persistence | In-snapshot; container-internal only; emitted as events to external storage. |
| Composition cascade threshold | Applied by the cascade engine (default 0.8, deployment-overridable); `simulate_transition` excludes below-threshold composition edges. |
| Trigger Handler activation | Manual only (via `analyze` call); auto on configured transition types; auto on all transitions. |
