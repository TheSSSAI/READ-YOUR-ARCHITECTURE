# Prompt — Generate `architecture.json` for any codebase

Copy everything in the **PROMPT** block below and paste it to Claude Code (or any capable coding agent) **inside the target project's repository**. Put explorer.html` from this folder into the target repo first (e.g. in `docs/architecture/`), or paste `schema.json`'s contents when asked. The agent will analyze the code and emit a `docs/architecture/architecture.json` that the generic `explorer.html` viewer renders as an interactive, recursive, drill-down diagram.

This prompt is **technology-agnostic** — it works for a Rust service, a React app, a Python data pipeline, a Go microservice mesh, a monolith, anything. It is engineered with explicit role-setting, a phased exploration-before-synthesis workflow, grounding/anti-hallucination rules, and a self-verification loop.

---

## PROMPT

````
You are a principal software architect performing a ground-truth architecture audit of THIS repository. Your single deliverable is a file `docs/architecture/architecture.json` that conforms exactly to the JSON Schema  that is given below at end of this prompt, (read it first; if it is not present, ask me to paste it before continuing). A generic viewer (`explorer.html`) will render this file as an interactive, recursive module map with directional arrows, so accuracy and correct references are not optional — broken references produce broken arrows.

Work in distinct phases. Do not jump to writing JSON. Think first, explore widely, ground every claim in real files, then synthesize, then verify.

────────────────────────────────────────
PHASE 0 — Contract & ground rules
────────────────────────────────────────
- Read `docs/architecture/schema.json` fully. Treat it as the output contract.
- GROUNDING RULE (most important): every node, connection, key file, and flow step must correspond to something that ACTUALLY EXISTS in this codebase. Open files and confirm. If you cannot point to code for a relationship, do not assert it. Never invent endpoints, classes, tables, or dependencies. Prefer "unknown / not found" over a plausible guess.
- FIDELITY RULE: use the real identifiers from the code (class/module/function/service/table/endpoint names), not idealized ones.
- SCOPE RULE: describe the system AS BUILT, including warts and TODOs — not the aspirational design.

────────────────────────────────────────
PHASE 1 — Reconnaissance (breadth first)
────────────────────────────────────────
Build a mental model before deciding structure. Determine:
- The dominant language(s), build system, and how the app is composed (entry point(s), DI/wiring, module/package boundaries, manifests, lockfiles, compose/helm/IaC).
- The top-level decomposition: monorepo packages? layered architecture? services? feature folders? Identify the 5–12 MAJOR subsystems — the things a new engineer must understand first.
- All EXTERNAL dependencies the code actually talks to: databases, caches, queues, third-party/internal APIs, auth providers, object storage, OS/hardware/devices, mail/SMS, payment, etc. Find them via clients, SDKs, connection strings, env config, and network calls — not by assumption.
- The cross-cutting concerns: auth/security, persistence, error handling, configuration, logging/observability, concurrency/scheduling.
Capture findings as a short written outline (subsystems, externals, and the main runtime flow) BEFORE building JSON. Show me this outline.

Search strategy (adapt to the stack): start from entry points and dependency-injection/wiring files; follow imports outward; read interface/contract definitions; grep for client/SDK instantiation, route/endpoint registration, DB/schema/migration files, config schemas, and message handlers. Read representative implementations, not just signatures.

────────────────────────────────────────
PHASE 2 — Choose the vocabularies (legend)
────────────────────────────────────────
Design `legend.nodeTypes` (≈6–12) to fit THIS project's real roles (e.g. entrypoint, service, api-client, renderer, store, security, ui, orchestrator, domain, worker, external...). Always include an "external" type. Give each a distinct hex color.
Design `legend.edgeKinds` for the relationship types that genuinely occur (e.g. control/call, data, network, persistence, secret, event/async, device). Give each a color. Every `connection.kind` and `node.type` you use later MUST be one of these ids.

────────────────────────────────────────
PHASE 3 — Model the tree (recursive nodes)
────────────────────────────────────────
- Top-level `nodes[]` = the major subsystems from Phase 1.
- Recurse with `children[]`: inside each subsystem, model its important modules/services/classes; inside those, only recurse further when there is real internal structure worth showing. Target depth 2–3; go deeper only where it adds understanding. Do NOT explode into every file — choose the load-bearing pieces. A leaf node should map to a real unit (class/module/function group).
- For each node provide: a UNIQUE kebab-case `id`, the real `name`, a `type` from the legend, a grounded `summary` (what + why), `tech`, `tags` (invariants/patterns/requirement-ids), `responsibilities` (verbs + real names), and `keyFiles` (real repo-relative paths you actually opened, each with a short note).
- Put external systems in the top-level `externals[]` array (type = your external type), with summary/tech/tags.

────────────────────────────────────────
PHASE 4 — Model the connections (the arrows)
────────────────────────────────────────
On each node, add `connections[]` for the directed relationships that ORIGINATE there. The arrow points from the node to `to`.
- `to` must be the id of another node, a child, or an external — and it MUST resolve somewhere in the document.
- `kind` = a legend edgeKind id. `label` = short verb phrase. `payload` = what actually flows (data shape, protocol, call semantics).
- Model connections at the RIGHT altitude: high-level subsystem→subsystem (and subsystem→external) edges at the top level; fine-grained class→class edges inside a subsystem's children. This keeps each level's diagram readable.
- Capture the real direction of dependency/data flow. If A calls B and B returns data, that is one edge A→B (note the payload as the returned data); add a second edge only if there is a genuinely separate channel (e.g. a callback/event).
- Ensure the externals get incoming edges from whatever talks to them.

────────────────────────────────────────
PHASE 5 — Model end-to-end flows
────────────────────────────────────────
Add `flows[]` for the 2–5 most important sequences (e.g. request lifecycle, auth/login, the core job/pipeline, startup/recovery). Each step names the acting node id (must resolve) and a grounded action. These read as the "story" of the system.

────────────────────────────────────────
PHASE 6 — Self-verification (do this before finishing)
────────────────────────────────────────
Validate, then FIX, then re-validate. Confirm ALL of:
1. The JSON parses and validates against `schema.json`.
2. Every `id` is unique across nodes (all depths) + externals.
3. Every `connection.to` and every `flows[].steps[].node` resolves to an existing id. (Run a quick script or trace it — list any that don't and fix them.)
4. Every `node.type` is a `legend.nodeTypes` id; every `connection.kind` is a `legend.edgeKinds` id.
5. Every `keyFiles[].path` exists in the repo (spot-check by opening several).
6. No invented facts: re-read your summaries/responsibilities and remove anything you cannot tie to code.
7. Readability: no level has so many sibling nodes or edges that it becomes a hairball — if it does, regroup into children or raise edges to a coarser altitude.
Report the verification results (counts: nodes, externals, edges, flows; and "0 broken references") before declaring done.

────────────────────────────────────────
OUTPUT
────────────────────────────────────────
- Write the final document to `docs/architecture/architecture.json` only. No prose inside the JSON.
- Then give me a brief summary: the subsystem list, the externals, and any place where the code was ambiguous or you had to make a judgment call.
- If something is genuinely undeterminable from the code, model it honestly (e.g. a node tagged `unverified`) rather than guessing.

Begin with PHASE 0 and PHASE 1, and show me the Phase 1 outline before building the full JSON.

## output json schema.json

{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/architecture-explorer/schema.json",
  "title": "Architecture Explorer Document",
  "description": "Technology-agnostic description of a software system's modules, their connections, data flows, external dependencies, and end-to-end flows. Consumed by explorer.html (a generic recursive viewer). Designed to be produced by analyzing any codebase regardless of language or stack.",
  "type": "object",
  "required": ["schemaVersion", "meta", "legend", "nodes"],
  "additionalProperties": false,
  "properties": {
    "schemaVersion": {
      "type": "string",
      "description": "Version of THIS schema the document targets.",
      "const": "1.0"
    },
    "meta": {
      "type": "object",
      "description": "Top-level identity of the system being described.",
      "required": ["title"],
      "additionalProperties": false,
      "properties": {
        "title": { "type": "string", "description": "System / product name." },
        "subtitle": { "type": "string", "description": "One-line positioning: what it is and the phase/scope." },
        "description": { "type": "string", "description": "2–5 sentence prose overview of purpose, key constraints, and design philosophy." },
        "tech": { "type": "array", "items": { "type": "string" }, "description": "Primary stack (languages, frameworks, infra). Shown as pills in the header." },
        "repo": { "type": "string", "description": "Repository name or path (optional)." }
      }
    },
    "legend": {
      "type": "object",
      "description": "The controlled vocabularies for node types and edge kinds, plus their display colors. node.type and connection.kind MUST reference ids defined here.",
      "required": ["nodeTypes", "edgeKinds"],
      "additionalProperties": false,
      "properties": {
        "nodeTypes": {
          "type": "array",
          "description": "The categories a module can have. Keep to ~6–12 meaningful, project-specific roles. Always include one for external systems.",
          "minItems": 1,
          "items": { "$ref": "#/$defs/legendEntry" }
        },
        "edgeKinds": {
          "type": "array",
          "description": "The kinds of relationships an arrow can represent (e.g. control/call, data flow, network, persistence, secret, device).",
          "minItems": 1,
          "items": { "$ref": "#/$defs/legendEntry" }
        }
      }
    },
    "externals": {
      "type": "array",
      "description": "Systems OUTSIDE this codebase that it talks to (databases, third-party APIs, OS services, message brokers, hardware). Rendered as dashed cards. Their ids may be referenced by any node's connection.to.",
      "items": { "$ref": "#/$defs/node" }
    },
    "nodes": {
      "type": "array",
      "description": "The root modules of the system (top level of the drill-down tree). Each node may contain children, recursively, to any depth.",
      "minItems": 1,
      "items": { "$ref": "#/$defs/node" }
    },
    "flows": {
      "type": "array",
      "description": "End-to-end sequences (e.g. request lifecycle, auth handshake, job pipeline). Each step references a node id so the viewer can jump to it.",
      "items": { "$ref": "#/$defs/flow" }
    }
  },

  "$defs": {
    "legendEntry": {
      "type": "object",
      "required": ["id", "label", "color"],
      "additionalProperties": false,
      "properties": {
        "id": { "type": "string", "description": "Stable key referenced by node.type or connection.kind.", "pattern": "^[a-z0-9][a-z0-9-]*$" },
        "label": { "type": "string", "description": "Human-readable name shown in the legend." },
        "color": { "type": "string", "description": "CSS color (hex recommended), used for the node dot / edge stroke.", "pattern": "^#?[0-9a-zA-Z(),.% /-]+$" }
      }
    },

    "node": {
      "type": "object",
      "description": "One module at any level. A node can be a whole subsystem, a service/class, an external system, or a domain concept. Children let the same shape recurse.",
      "required": ["id", "name", "type"],
      "additionalProperties": false,
      "properties": {
        "id": {
          "type": "string",
          "description": "Globally UNIQUE, stable, kebab-case id. Referenced by connection.to and flow step.node. Must be unique across nodes, children (any depth), and externals.",
          "pattern": "^[a-z0-9][a-z0-9-]*$"
        },
        "name": { "type": "string", "description": "Display name (the real class/module/service name where possible)." },
        "type": {
          "type": "string",
          "description": "One of legend.nodeTypes[].id. Drives the color. Unknown values fall back to a neutral color."
        },
        "summary": {
          "type": "string",
          "description": "1–3 sentences: what this module is responsible for and WHY it exists. Grounded in the actual code, not generic boilerplate."
        },
        "tech": { "type": "array", "items": { "type": "string" }, "description": "Concrete technologies/libraries this module uses." },
        "tags": { "type": "array", "items": { "type": "string" }, "description": "Short keywords: cross-cutting properties, invariants, patterns, requirement ids. First 3 are shown on the card." },
        "responsibilities": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Bullet list of the concrete things this module does. Prefer verbs and real method/endpoint names."
        },
        "keyFiles": {
          "type": "array",
          "description": "Real source files that implement this node, with a short note. Paths MUST exist in the repo (relative to repo root).",
          "items": {
            "type": "object",
            "required": ["path"],
            "additionalProperties": false,
            "properties": {
              "path": { "type": "string", "description": "Repo-relative file path." },
              "note": { "type": "string", "description": "What this file contributes (optional)." }
            }
          }
        },
        "connections": {
          "type": "array",
          "description": "Directed relationships FROM this node TO another node/external. Represents real calls, data flows, or dependencies found in the code. Drives the arrows.",
          "items": { "$ref": "#/$defs/connection" }
        },
        "children": {
          "type": "array",
          "description": "Sub-modules contained within this node. Recursing into a node shows these as its own diagram. Omit or leave empty for leaf nodes.",
          "items": { "$ref": "#/$defs/node" }
        },
        "external": {
          "type": "boolean",
          "description": "Optional explicit marker for an external system. Items under top-level `externals` are treated as external automatically."
        }
      }
    },

    "connection": {
      "type": "object",
      "description": "A directed edge. The arrow points from the owning node to `to`.",
      "required": ["to", "kind"],
      "additionalProperties": false,
      "properties": {
        "to": {
          "type": "string",
          "description": "id of the target node or external. MUST resolve to some node.id / external.id somewhere in the document."
        },
        "kind": {
          "type": "string",
          "description": "One of legend.edgeKinds[].id. Drives the edge color and meaning."
        },
        "label": {
          "type": "string",
          "description": "Short verb phrase for the relationship (e.g. \"claim / heartbeat\", \"reads token\", \"GET /api/...\")."
        },
        "payload": {
          "type": "string",
          "description": "What actually flows across this edge: the data shape, message, protocol detail, or call semantics."
        }
      }
    },

    "flow": {
      "type": "object",
      "required": ["id", "name", "steps"],
      "additionalProperties": false,
      "properties": {
        "id": { "type": "string", "pattern": "^[a-z0-9][a-z0-9-]*$" },
        "name": { "type": "string", "description": "Name of the end-to-end sequence." },
        "summary": { "type": "string", "description": "One line on what the flow accomplishes and any key invariant." },
        "steps": {
          "type": "array",
          "minItems": 2,
          "description": "Ordered steps. Each names the node that acts and what it does.",
          "items": {
            "type": "object",
            "required": ["node", "action"],
            "additionalProperties": false,
            "properties": {
              "node": { "type": "string", "description": "id of the node performing this step (must resolve)." },
              "action": { "type": "string", "description": "What happens at this step, grounded in the code." }
            }
          }
        }
      }
    }
  }
}

````

---

## Why this prompt is engineered the way it is

| Technique | Where | Purpose |
|---|---|---|
| **Role / persona priming** | "principal software architect … ground-truth audit" | Raises the bar for rigor and sets the analytical stance. |
| **Single, concrete deliverable + output contract** | schema.json referenced as "the contract" | Removes ambiguity about format; lets the model self-check. |
| **Phase decomposition (plan-then-act)** | PHASE 0–6 | Forces exploration *before* synthesis — the #1 cause of inaccurate diagrams is writing structure before reading code. |
| **Breadth-first reconnaissance + explicit search strategy** | PHASE 1 | Stack-agnostic guidance on *how* to discover structure and externals. |
| **Grounding / anti-hallucination rules** | PHASE 0 GROUNDING/FIDELITY/SCOPE | "Point to code or don't assert it." Strongly reduces invented endpoints/classes. |
| **Controlled vocabulary first** | PHASE 2 | Guarantees `type`/`kind` values are legal and colors are meaningful. |
| **Altitude guidance for edges** | PHASE 4 | Prevents the cross-level hairball; keeps each drill-down readable. |
| **Self-verification loop with a checklist** | PHASE 6 | The model validates references and file existence and *fixes* before finishing — this is what makes the arrows render correctly. |
| **Honest-uncertainty escape hatch** | OUTPUT | "Model it as unverified rather than guess" keeps the artifact trustworthy. |
| **Human checkpoint** | end of PHASE 1 | You approve the decomposition before the model invests in full JSON. |

## Quick start in a new project
1. Copy `schema.json` and `explorer.html` (and optionally this file) into the new repo, e.g. `docs/architecture/`.
2. Paste the **PROMPT** block to Claude Code in that repo.
3. Review the Phase 1 outline it shows you; nudge if a subsystem or external is missing.
4. Let it write `docs/architecture/architecture.json` and report "0 broken references".
5. Open `explorer.html` (drag the JSON onto it if opened via `file://`).

## Tuning knobs (optional additions to the prompt)
- **Depth:** "Limit recursion to depth 2" for a quick map, or "go to depth 4 in the `payments` subsystem" for a focused deep-dive.
- **Focus:** "Only model the backend services and their externals; ignore the marketing site."
- **Audience:** "Write summaries for an engineer new to the team" vs "for a security reviewer — emphasize trust boundaries and data-at-rest."
- **Validation:** "After writing, run a Node/Python one-liner that loads the JSON and asserts every `connection.to` resolves; paste the output."

