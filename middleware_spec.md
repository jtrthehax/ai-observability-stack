# AI Observability Stack — Middleware Spec v0.2

**Author:** Joel Robinson
**Status:** Draft
**Related specs:** DSR/SLS v0.2, SDE v1.0, Unified Model v1.2

---

### Problem Statement

LLM systems currently have no runtime observability layer. Output failures — inference drift events, register shifts, pattern retrievals, constraint violations — are invisible at the time they occur. Debugging is post-hoc, blind, and produces folklore rather than engineering. 

The transformer is not an uninstrumented system. It is a **poorly instrumented system.** The signals are present. The instrumentation layer has not been built.

---

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                     USER INPUT                       │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│            LAYER 1: CONSTRAINT INTERCEPTOR           │
│                                                      │
│  Runs constraint inventory before generation         │
│  Flags: UNDEFINED_CONSTRAINT / INCONSISTENT_LOGIC    │
│  Source: SDE schema standardization layer            │
│  Emits: BREAKPOINT if threshold crossed              │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│              TRANSFORMER (generation)                │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│            LAYER 2: OUTPUT MONITOR                   │
│                                                      │
│  Reads style register delta (boundary signature)     │
│  Reads declared hop chain (traversal audit)          │
│  Reads constraint resolution (did it answer the      │
│  reconstructed intent or the compressed surface?)    │
│  Emits: BOUNDARY_EVENT / PATTERN_JUMP / CLEAN        │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│               LAYER 3: EVENT BUS                     │
│                                                      │
│  BREAKPOINT          → Constraint Auditor            │
│  BOUNDARY_EVENT      → State Reconstructor           │
│  PATTERN_JUMP        → Hop Count Inspector           │
│  CLEAN               → Pass through                  │
│  STATE_BOUNDARY      → DSR capture trigger           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│            LAYER 4: SESSION STATE CAPTURE            │
│                                                      │
│  On STATE_BOUNDARY or session end:                   │
│  → Emit DSR Structured State Envelope                │
│  → Capture: constraints, trajectory, regulatory      │
│    state, instruction nodes, event log               │
│  Source: DSR/SLS spec v0.2                           │
└──────────────────────┬──────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│         USER OUTPUT + INTROSPECTION LOG              │
└─────────────────────────────────────────────────────┘
```

---

### Console Command Registry

```
COMMAND             LAYER    FIRES                    RETURNS
──────────────────────────────────────────────────────────────
meta:log            3        SESSION_START            Begin event log
meta:dump           3        Manual                   Full session state snapshot
meta:trace          2        Manual / BREAKPOINT      Token decision path, last response
meta:diff           2        BREAKPOINT               User intent vs. model active frame
meta:constraints    1        Manual / BREAKPOINT      All variables: DEFINED / UNDEFINED / INCONSISTENT
meta:hops [N]       2        Manual / BREAKPOINT      Inference hop chain, N steps back
meta:boundary       2        Manual                   Output mode shift events this session
meta:frame          2        Manual                   Active frame and referent set
meta:pattern        2        PATTERN_JUMP             Retrieved pattern vs. derived step
meta:state          4        STATE_BOUNDARY           DSR envelope preview
meta:entropy        2        Manual                   Token uncertainty surface (where available)
```

---

### Named Failure Taxonomy

The field currently treats all output failures as a single category. This is the wrong taxonomy — it conflates mechanistically distinct failure modes that require different fixes: 

```
FAILURE TYPE              MECHANISM                     FIX
──────────────────────────────────────────────────────────────────────
UNDEFINED_CONSTRAINT      Variable has no binding       Ask user to constrain
                          at generation time            before generating

INCONSISTENT_LOGIC        Two constraints contradict    Surface the conflict,
                          each other                    ask user to resolve

PATTERN_RETRIEVAL         Model retrieved stored        Force explicit hop
                          pattern instead of            enumeration
                          traversing the chain

REGISTER_SHIFT            Model output mode changed     Reconstruct STATE[pre-shift],
                          — style delta confirmed       attempt re-entry

CONTEXT_PRUNING           Decompression material        Restore context before
                          removed to save tokens,       re-generating
                          narrowing frame construction

CLEAN                     All signals nominal           Pass through
```

---

### Why This Is Infrastructure, Not Prompting

| Prompting | Infrastructure |
| --- | --- |
| Modifies input | Instruments the runtime |
| Session-specific | Session-agnostic |
| No event log | Persistent event log |
| Blind to failures | Names and routes failures |
| Cannot distinguish failure types | Taxonomy-driven signal separation |
| Chain-of-thought (ask for reasoning) | Hop count inspector (verify traversal) |

Chain-of-thought asked the model to *narrate* its reasoning. This stack asks the model to *prove* it traversed every step — and catches it when it doesn't. 

---

### Relationship to Existing Specs

```
SDE (input schema)         →  feeds Layer 1 (Constraint Interceptor)
                               constraint taxonomy, flag registry

DSR/SLS (state)            →  feeds Layer 4 (Session State Capture)
                               SSE format, trajectory nodes,
                               regulatory state nodes

Unified Model              →  explains WHY the failure modes exist
(substrate)                   mechanistically — token precision-lock,
                               prediction window geometry,
                               output drift as structural not random

Reasoning Integrity        →  implements Layers 2 and 3
Module v3.0                   (Output Monitor + Event Bus)
```

---

### What This Enables That Doesn't Exist Today

1. **Signal separation** — distinguishing UNDEFINED_CONSTRAINT from PATTERN_RETRIEVAL from REGISTER_SHIFT instead of seeing "a wrong answer"

2. **Traversal proof** — hop count inspector forces the model to demonstrate it walked every inference step, not just narrate a plausible path

3. **Output mode visibility** — style register delta makes BOUNDARY_EVENTs detectable by signal, not just by recognizing template phrases

4. **Session continuity with audit trail** — DSR captures not just what was decided but what events fired, what frame was active, what the regulatory state was 

5. **The missing stack trace** — every software system has one. AI currently ships output with no trace. This is the trace layer.

---

### Global Runtime Model

At the highest level, every request runs through a fixed sequence:

1. **Constraint inventory (Layer 1)** on `USER INPUT`
2. **Transformer generation** (model call)
3. **Output monitoring (Layer 2)** on the completion
4. **Event routing (Layer 3)** based on signals
5. **State capture (Layer 4)** on `STATE_BOUNDARY` or session end

---

### Layer 1 — Constraint Interceptor Semantics

**Inputs:**
- Raw user input (text)
- SDE constraint schema (variable definitions, types, allowed ranges, binding rules)

**Processing steps:**

1. Parse constraints — extract all variables, assign status in `{DEFINED, UNDEFINED, INCONSISTENT}`
2. Status rules:
   - `DEFINED` — binding satisfies SDE type and range constraints
   - `UNDEFINED_CONSTRAINT` — variable present but no binding supplied
   - `INCONSISTENT_LOGIC` — two or more bindings violate logical constraints
3. Constraint severity score `S`:
   - Each `UNDEFINED_CONSTRAINT` → +1
   - Each `INCONSISTENT_LOGIC` → +2
   - If `S ≥ S_threshold` → emit `BREAKPOINT` to Event Bus
4. On `BREAKPOINT`:
   - Default policy: pause generation, route to Constraint Auditor
   - Alternative policy: generate but mark session as degraded

---

### Layer 2 — Output Monitor Semantics

**Inputs:**
- Model completion (text + token-level metadata where available)
- Declared hop chain
- Constraint inventory result (from Layer 1)
- Style register baseline

**Processing steps:**

1. **Style register delta:**
   Compute `DELTA[style]` for the completion:
   - Features: register shift, tone, verbosity, hedge density, entity count
   - If `DELTA[style] ≥ D_threshold` → emit `BOUNDARY_EVENT`

2. **Hop chain inspection:**
   - If reconstructed chain shows jump from premise to conclusion without intermediate steps → emit `PATTERN_JUMP`

3. **Constraint resolution check:**
   - Silent assumption of undefined variable → emit `UNDEFINED_CONSTRAINT` (runtime)
   - Answer addresses compressed surface instead of reconstructed intent → emit `CONTEXT_PRUNING`

4. **CLEAN rule:**
   - No `BOUNDARY_EVENT`, no `PATTERN_JUMP`, no unresolved constraints, no `CONTEXT_PRUNING` → emit `CLEAN`

---

### Layer 3 — Event Bus Semantics

**Priority ordering:**
```
Highest  →  BREAKPOINT
         →  BOUNDARY_EVENT
         →  PATTERN_JUMP
         →  CONTEXT_PRUNING
Lowest   →  CLEAN
STATE_BOUNDARY is orthogonal — triggers capture, not diagnosis
```

**Handlers:**
```
BREAKPOINT       → Constraint Auditor
                   Pause generation. Surface constraint table.

BOUNDARY_EVENT   → State Reconstructor
                   Rebuild STATE[pre-shift].
                   Attempt re-entry from STATE[pre-shift].

PATTERN_JUMP     → Hop Count Inspector
                   Force explicit hop enumeration.
                   Mark session as pattern-substituted until
                   traversal proof is complete.

CLEAN            → Pass through. Log as nominal.

STATE_BOUNDARY   → Invoke Layer 4 to emit SSE.
```

**Event coalescing:**
Multiple events in a single response → composite failure envelope:
```
EVENT[BREAKPOINT + PATTERN_JUMP] → "constraint + traversal failure"
```

---

### Layer 4 — Session State Capture Semantics

**Trigger conditions:**
- Explicit `STATE_BOUNDARY` event (major frame change, logical episode end)
- Session end

**Capture semantics:**
Build DSR Structured State Envelope: 
- Constraints — full table with statuses and resolution history
- Trajectory — ordered turns with event annotations
- Regulatory state — current and historical
- Instruction nodes — all instructions that shaped the session
- Event log — all events from Layer 3

---

### Execution Loop

For each user turn:

1. Run Layer 1 → compute constraint statuses → possibly emit `BREAKPOINT`
2. Decide generation:
   - `BREAKPOINT` + strict policy → route to Constraint Auditor, no generation
   - Otherwise → call transformer
3. Run Layer 2 → compute style delta, hop chain, constraint resolution → emit events
4. Run Layer 3 → route events to handlers → update event log
5. Run Layer 4 if `STATE_BOUNDARY` or session end → emit SSE

---

### Repo Structure

```
ai-observability-stack/
│
├── README.md
├── specs/
│   ├── middleware_spec.md              ← this document
│   ├── console_commands.md             ← full command registry
│   └── failure_taxonomy.md             ← named failure types
│
├── layers/
│   ├── layer1_constraint_interceptor.md
│   ├── layer2_output_monitor.md
│   ├── layer3_event_bus.md
│   └── layer4_state_capture.md
│
├── modules/
│   └── reasoning_integrity_v3.md
│
└── related/
    ├── → SDE repo
    ├── → DSR/SLS repo
    └── → Unified Model repo
```
