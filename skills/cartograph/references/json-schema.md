# Cartograph JSON Schema (V0)

This is the contract between the extraction skill and the visualizer. Output must conform exactly to this structure.

## Top-level structure

```json
{
  "meta": { ... },
  "surfaces": [ ... ],
  "features": [ ... ],
  "entities": [ ... ],
  "relationships": [ ... ],
  "operations": [ ... ],
  "flows": [ ... ],
  "compartments": [ ... ],
  "fileTree": [ ... ],
  "codeHealth": { ... }
}
```

## `meta`

```json
{
  "name": "my-app",
  "analyzedAt": "2025-01-15T10:30:00Z",
  "version": "0.1.0",
  "rootDir": "."
}
```

- `name` — repo/project name (from package.json or directory name)
- `analyzedAt` — ISO 8601 timestamp
- `version` — schema version, always `"0.1.0"` for V0
- `rootDir` — root directory analyzed (usually `"."`)

## `codeHealth`

Persistent engineering health metrics. This key is **optional for backwards compatibility**; when absent, the visualizer should show a graceful "No code health data" state.

```json
{
  "analyzedAt": "2025-01-15T10:30:00Z",
  "metrics": [
    {
      "id": "co-location",
      "name": "Co-location",
      "description": "Measures whether files live close to their consumers",
      "score": 92,
      "thresholds": { "green": 90, "yellow": 70 },
      "summary": "8 of 103 evaluated files are misplaced",
      "findings": [
        {
          "file": "lib/format-date.ts",
          "verdict": "fail",
          "reason": "Used exclusively by app/chat — should be co-located",
          "consumers": ["app/chat/[personaSlug]/components/message-bubble.tsx"],
          "recommendation": {
            "action": "move",
            "target": "app/chat/[personaSlug]/lib/format-date.ts"
          }
        }
      ]
    },
    {
      "id": "dryness",
      "name": "DRYness",
      "description": "Measures duplicated or near-duplicated functionality",
      "score": 78,
      "thresholds": { "green": 90, "yellow": 70 },
      "summary": "3 duplication clusters found across 7 files",
      "scalingFactor": 0.97,
      "findings": [
        {
          "id": "prompt-remix-duplication",
          "title": "Prompt remix experience duplicated across surfaces",
          "severity": "high",
          "implementations": [
            {
              "location": "app/chat/[personaSlug]/components/prompt-remix.tsx",
              "description": "Chat-embedded prompt remix modal with conversation context"
            },
            {
              "location": "app/admin/components/remix-panel.tsx",
              "description": "Admin panel for remixing prompts during content review"
            }
          ],
          "sharedLogic": [
            "Prompt transformation state machine",
            "FAL API client for remix generation",
            "Prompt history tracking"
          ],
          "recommendation": {
            "action": "Extract shared logic",
            "target": "features/prompt-remix/",
            "shared": ["remix state machine", "API client", "prompt history hook"],
            "keepSeparate": ["admin UI chrome", "chat modal wrapper", "conversation context injection"]
          }
        }
      ]
    },
    {
      "id": "dead-code",
      "name": "Dead Code",
      "description": "Measures how much of the codebase is unreachable, orphaned, or unused",
      "score": 95,
      "thresholds": { "green": 90, "yellow": 70 },
      "summary": "12 dead items found: 8 dead files, 1 orphaned surface, 1 orphaned feature, 2 dead entities. 3 test-only files flagged.",
      "findings": [
        {
          "id": "dead-file-lib-old-utils",
          "kind": "dead-file",
          "severity": "high",
          "target": "lib/old-utils.ts",
          "reason": "No file in the codebase imports this file",
          "evidence": {
            "importers": []
          },
          "recommendation": {
            "action": "remove",
            "detail": "Delete this file — nothing imports it and it is not an entry point."
          }
        }
      ]
    }
  ]
}
```

- `analyzedAt` — ISO 8601 timestamp for when the health metrics were computed
- `metrics[]` — pluggable array of engineering metrics. The visualizer dispatches on `metric.id` to render each metric's finding type
  - `id` — stable kebab-case metric identifier (for example `"co-location"` or `"dryness"`)
  - `name` — human-readable label shown in the visualizer
  - `description` — 1 sentence explanation of what this metric measures
  - `score` — 0–100 integer or float
  - `thresholds.green` — score at or above this is green
  - `thresholds.yellow` — score at or above this is yellow and below green; anything lower is red
- `summary` — short plain-language summary of the current result
- `findings[]` — metric-specific finding objects
- `scalingFactor` — optional metric-specific scalar (currently used by DRYness)
- Metrics may define additional finding-level fields beyond the shared envelope. Dead-code findings use `target`, `evidence`, and `recommendation.detail`.

### Co-location metric

The co-location metric uses `metric.id = "co-location"` and binary pass/fail findings for every misplaced file.

```json
{
  "file": "lib/format-date.ts",
  "verdict": "fail",
  "reason": "Used exclusively by app/chat — should be co-located",
  "consumers": ["app/chat/[personaSlug]/components/message-bubble.tsx"],
  "recommendation": {
    "action": "move",
    "target": "app/chat/[personaSlug]/lib/format-date.ts"
  }
}
```

- `file` — misplaced file path
- `verdict` — `"pass"` or `"fail"`; only failing findings need to be emitted in the metric summary list
- `reason` — plain-language explanation of why the file is misplaced
- `consumers[]` — importing files that justify the verdict
- `recommendation.action` — `"move"` or `"promote"`
- `recommendation.target` — suggested new path or destination directory

### DRYness metric

The DRYness metric uses `metric.id = "dryness"` and emits duplication clusters or near-duplicate implementations.

```json
{
  "id": "prompt-remix-duplication",
  "title": "Prompt remix experience duplicated across surfaces",
  "severity": "high",
  "implementations": [
    {
      "location": "app/chat/[personaSlug]/components/prompt-remix.tsx",
      "description": "Chat-embedded prompt remix modal with conversation context"
    },
    {
      "location": "app/admin/components/remix-panel.tsx",
      "description": "Admin panel for remixing prompts during content review"
    }
  ],
  "sharedLogic": [
    "Prompt transformation state machine",
    "FAL API client for remix generation",
    "Prompt history tracking"
  ],
  "recommendation": {
    "action": "Extract shared logic",
    "target": "features/prompt-remix/",
    "shared": ["remix state machine", "API client", "prompt history hook"],
    "keepSeparate": ["admin UI chrome", "chat modal wrapper", "conversation context injection"]
  }
}
```

- `id` — stable finding identifier
- `title` — plain-language title of the duplication
- `severity` — `"high"`, `"medium"`, or `"low"`
- `implementations[]` — duplicated locations and what they do
  - `location` — file or directory path
  - `description` — why this implementation exists / what is special about it
- `sharedLogic[]` — specific duplicated logic worth extracting
- `recommendation.action` — consolidation strategy
- `recommendation.target` — suggested destination for shared logic
- `recommendation.shared[]` — logic that should be extracted
- `recommendation.keepSeparate[]` — logic that should remain implementation-specific

### Dead Code metric

The dead-code metric uses `metric.id = "dead-code"` and emits product-level and code-level dead-code findings. Test-only files are informational: they appear in `findings[]`, but they do **not** affect the dead-code score.

```json
{
  "id": "dead-file-lib-old-utils",
  "kind": "dead-file",
  "severity": "high",
  "target": "lib/old-utils.ts",
  "reason": "No file in the codebase imports this file",
  "evidence": {
    "importers": []
  },
  "recommendation": {
    "action": "remove",
    "detail": "Delete this file — nothing imports it and it is not an entry point."
  }
}
```

- `id` — stable finding identifier
- `kind` — `"dead-file"`, `"test-only-file"`, `"orphaned-surface"`, `"orphaned-feature"`, or `"dead-entity"`
- `severity` — `"high"`, `"medium"`, or `"low"`
- `target` — dead item identifier (file path, surface id, feature id, or entity id)
- `reason` — plain-language explanation of why the item was flagged
- `evidence` — kind-specific supporting evidence
- `recommendation.action` — `"remove"` or `"review"`
- `recommendation.detail` — concrete next step for an engineer

#### Dead-code evidence shapes

**`dead-file`:**

```json
{
  "importers": []
}
```

**`test-only-file`:**

```json
{
  "importers": ["__tests__/utils.test.ts", "__tests__/helpers.test.ts"]
}
```

**`orphaned-surface`:**

```json
{
  "route": "/settings/legacy",
  "navigationReferences": []
}
```

**`orphaned-feature`:**

```json
{
  "surfaceIds": ["legacy-settings"],
  "allSurfacesOrphaned": true,
  "implementationFiles": ["app/settings/legacy/components/feature-x.tsx"],
  "allImplementationsDead": true
}
```

**`dead-entity`:**

```json
{
  "entityKind": "db-model",
  "referencingOperations": [],
  "referencingFeatures": [],
  "prismaQueryCount": 0
}
```

## `surfaces[]`

A surface is a self-contained entry point or standalone piece of functionality in the app. Each app is a collection of surfaces — distinct experiences that work on their own (e.g., "Prompt Wizard", "Admin Review Dashboard", "Explore Feed"). Surfaces are the top-level organizational unit; entities are scoped to surfaces via `surfaceIds`.

```json
{
  "id": "prompt-wizard",
  "name": "Prompt Wizard",
  "description": "Interactive editing experience for creating and refining image generation prompts",
  "entrypoint": {
    "file": "app/create/page.tsx",
    "route": "/create"
  },
  "actor": "user",
  "entityIds": ["creation", "prompt-template", "model-config"],
  "operationIds": ["generate-preview", "refine-prompt"],
  "flowIds": ["creation-flow"],
  "compartmentIds": ["creation-studio-ui", "prompt-engine"],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `name` — human-readable name for this surface
- `description` — 1-2 sentence explanation of what this surface does
- `entrypoint` — where this surface lives in code and routing
  - `file` — the main page/route file (relative path)
  - `route` — the URL route (e.g., `/create`, `/admin/reviews`)
- `actor` — primary actor: `"user"`, `"admin"`, or `"system"`
- `entityIds` — references to entity ids that this surface touches
- `operationIds` — references to operation ids used within this surface
- `flowIds` — references to flow ids that belong to this surface
- `compartmentIds` — references to compartment ids for code areas that serve this surface. Populated during Phase 9 (compartment dependency mapping)
- `confidence` — `"high"` | `"medium"` | `"low"`

### Entity exposure

Every entity should appear in at least one surface's `entityIds`. Entities that appear in only one surface are **surface-scoped** (private to that experience). Entities that appear in many surfaces are **cross-cutting** (shared infrastructure). This exposure mapping is the key output — it shows how tightly coupled or independent each surface is.

## `features[]`

A feature is a standalone capability embedded within one or more surfaces. Surfaces are "where you go" (pages); features are "what you can do" (capabilities). Features are the reusable functional building blocks that surfaces compose — things like "the like system", "the prompt wizard", "the star credit system."

```json
{
  "id": "prompt-wizard",
  "name": "Prompt Wizard",
  "description": "Multi-step guided/custom interface for constructing image generation requests with AI-powered prompt suggestions",
  "kind": "tool",
  "surfaceIds": ["chat-thread"],
  "entityIds": ["creation", "message"],
  "implementations": [
    { "file": "app/chat/[personaSlug]/components/wizard-modal/index.tsx", "description": "Wizard modal entry point" },
    { "file": "app/chat/[personaSlug]/wizard-actions/core.ts", "description": "Server actions for wizard submission and FAL polling" }
  ],
  "compartmentIds": ["prompt-wizard-ui", "image-generation-pipeline"],
  "files": [],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `name` — human-readable name
- `description` — 1-2 sentence explanation of what this feature does
- `kind` — one of:
  - `"tool"` — interactive multi-step experience (wizards, editors, sandboxes)
  - `"interaction"` — single-action engagement pattern (like, save, follow)
  - `"transaction"` — money/credit flow (purchase, tip, unlock)
  - `"gate"` — access control mechanism (age verification, NSFW filter, auth wall)
  - `"infrastructure"` — backend capability used by other features (AI generation, media processing)
  - `"workflow"` — multi-step admin/system process (review pipeline, scan workflow)
- `surfaceIds` — which surfaces embed this feature
- `entityIds` — which entities this feature reads/writes
- `implementations[]` — key files (2-5 most important)
  - `file` — relative file path
  - `description` — what this file does for the feature
- `compartmentIds` — references to compartment ids for code areas that implement this feature. This is the primary code-mapping mechanism — use compartments to understand which code belongs to this feature. Populated during Phase 9
- `files` — *(deprecated, kept for backwards compatibility)* exhaustive list of every file that participates in this feature. **No longer populated** — replaced by `compartmentIds`. Each entry:
  - `file` — relative file path
  - `role` — short label: `"component"`, `"action"`, `"hook"`, `"lib"`, `"style"`, `"test"`, `"type"`, `"config"`, `"api"`, `"other"`
- `confidence` — `"high"` | `"medium"` | `"low"`

### Feature vs. Surface

A **surface** has a URL and is something users navigate to. A **feature** is a capability that could theoretically be extracted and reused elsewhere. If you can describe it without naming a specific page ("the like system", "the payment flow"), it's a feature. If it's inherently tied to a route ("the explore page"), it's a surface.

## `entities[]`

```json
{
  "id": "user",
  "name": "User",
  "kind": "db-model",
  "description": "A registered user of the application",
  "source": { "file": "prisma/schema.prisma", "line": 12 },
  "fields": [
    { "name": "id", "type": "string", "description": "Unique identifier" },
    { "name": "email", "type": "string", "description": "User email address" },
    { "name": "name", "type": "string?", "description": "Display name" }
  ],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier (used for cross-references)
- `name` — PascalCase display name
- `kind` — one of: `"db-model"`, `"dto"`, `"state"`, `"derived"`, `"config"`, `"enum"`
- `description` — 1-2 sentence plain-language explanation
- `source` — where defined in code. `file` is relative to rootDir
- `fields[]` — key properties. Include 3-8 most important fields, not every field
  - `name` — field name as in code
  - `type` — type as a readable string (e.g., `"string"`, `"number"`, `"Post[]"`, `"Date?"`)
  - `description` — what this field represents
- `confidence` — `"high"` | `"medium"` | `"low"`
- `surfaceIds` — (derived, not stored in entity) computed by the visualizer from `surfaces[].entityIds`. Shows which surfaces reference this entity. Entities in 1 surface are "surface-scoped"; entities in 3+ surfaces are "cross-cutting".

## `relationships[]`

```json
{
  "id": "user-has-many-posts",
  "from": "user",
  "to": "post",
  "type": "has-many",
  "description": "A user can create multiple posts",
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `from` — entity id (source)
- `to` — entity id (target)
- `type` — one of: `"has-many"`, `"belongs-to"`, `"has-one"`, `"references"`, `"derives-from"`
- `description` — plain-language explanation
- `confidence` — `"high"` | `"medium"` | `"low"`

## `operations[]`

```json
{
  "id": "create-post",
  "name": "Create Post",
  "entityId": "post",
  "type": "create",
  "description": "Creates a new post with image and caption",
  "implementation": {
    "file": "app/api/posts/route.ts",
    "function": "POST"
  },
  "sideEffects": ["Creates an Activity entry", "Triggers notification to followers"],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `name` — human-readable name
- `entityId` — which entity this operates on (references entity id)
- `type` — one of: `"create"`, `"read"`, `"update"`, `"delete"`, `"domain"`
- `description` — what this operation does
- `implementation` — where in code
  - `file` — relative file path
  - `function` — function/handler name
- `sideEffects` — array of strings describing other effects (can be empty `[]`)
- `confidence` — `"high"` | `"medium"` | `"low"`

## `flows[]`

```json
{
  "id": "post-creation-flow",
  "name": "Post Creation",
  "trigger": "User taps the + button and submits the post form",
  "actor": "user",
  "description": "End-to-end flow for creating and publishing a new post",
  "steps": [
    {
      "order": 1,
      "description": "User fills in caption and selects image",
      "operationId": null,
      "entityId": null,
      "implementation": { "file": "app/create/page.tsx", "function": "CreatePostForm" }
    },
    {
      "order": 2,
      "description": "Form submits to create post API",
      "operationId": "create-post",
      "entityId": "post",
      "implementation": { "file": "app/api/posts/route.ts", "function": "POST" }
    },
    {
      "order": 3,
      "description": "Post appears in the user's profile grid",
      "operationId": "read-user-posts",
      "entityId": "post",
      "implementation": { "file": "app/profile/page.tsx", "function": "ProfileGrid" }
    }
  ],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `name` — short descriptive name for the flow
- `trigger` — what initiates this flow (plain language)
- `actor` — one of: `"user"`, `"admin"`, `"system"`
- `description` — 1-2 sentence summary
- `steps[]` — ordered list of steps
  - `order` — 1-indexed step number
  - `description` — what happens at this step (plain language)
  - `operationId` — reference to operation id (nullable — some steps are pure UI with no data operation)
  - `entityId` — reference to entity id (nullable)
  - `implementation` — where in code
    - `file` — relative file path
    - `function` — function/component name
- `confidence` — `"high"` | `"medium"` | `"low"`

## `compartments[]`

A compartment is a logical grouping of related files that form a cohesive unit of functionality. Compartments bridge the product-side view (surfaces, features) with the underlying code structure. They are nestable (via `parentId`), non-exclusive (a file can appear in multiple compartments), and exhaustive (every non-generated file belongs to at least one compartment).

```json
{
  "id": "image-generation-pipeline",
  "name": "Image Generation Pipeline",
  "description": "Server actions, FAL client integration, and prompt assembly for generating images via the wizard and content studio.",
  "parentId": "content-generation",
  "tags": ["business-logic", "api-integration"],
  "files": [
    { "file": "app/chat/[personaSlug]/actions/generate-image.ts", "role": "action" },
    { "file": "lib/fal-client.ts", "role": "lib" },
    { "file": "lib/prompt-assembly.ts", "role": "lib" }
  ],
  "featureIds": ["image-generation", "prompt-wizard"],
  "surfaceIds": ["chat-thread", "admin-content-studio"],
  "dependsOn": ["media-delivery", "database-access"],
  "confidence": "high"
}
```

- `id` — unique kebab-case identifier
- `name` — human-readable compartment name (name after what it does, not folder names)
- `description` — 1-2 sentence explanation of what this code area does
- `parentId` — ID of parent compartment, or `null` for top-level compartments. Enables unlimited nesting depth
- `tags` — semi-structured tags from the suggested vocabulary below; the AI may also add custom tags as needed
- `files[]` — all files in this compartment with their roles
  - `file` — relative file path from repo root
  - `role` — one of: `"component"`, `"hook"`, `"action"`, `"api"`, `"lib"`, `"type"`, `"config"`, `"style"`, `"test"`, `"other"` (same role enum as feature files)
- `featureIds` — IDs of features this compartment implements (can be empty)
- `surfaceIds` — IDs of surfaces this compartment serves (can be empty)
- `dependsOn` — IDs of other compartments this one depends on (imports from). Only inter-compartment dependencies, no self-references
- `confidence` — `"high"` | `"medium"` | `"low"`

### Tag vocabulary (semi-structured)

Suggested core tags — the AI should use these when applicable and may create additional custom tags:

- `ui` — visual components, layouts, styling
- `data-access` — database queries, Prisma operations, data fetching
- `business-logic` — domain rules, calculations, validation
- `api` — API routes, server actions, external service integrations
- `api-integration` — third-party service clients (FAL, Clerk, Stripe, etc.)
- `infrastructure` — config, build tooling, dev tooling, CI/CD
- `shared` — utilities used across many compartments
- `state-management` — stores, context providers, state hooks
- `auth` — authentication and authorization
- `testing` — test files, test utilities, fixtures

### Compartment guidelines

- Don't create compartments with only 1 file — merge small groupings into their parent
- Keep top-level compartments to 8–15 for a typical web app; more sub-compartments are fine
- Prefer meaningful groupings over 1:1 folder mapping
- Every non-generated file must appear in at least one compartment
- Config/infra files go into a "Project Infrastructure" compartment

## `fileTree[]`

*(Experimental — fully isolated from existing data. See `specs/spec-file-tree.md` for the full spec.)*

A flat array of per-file entries with AI-estimated feature weight breakdowns. Used by the Feature Map visualizer tab to render feature-composition bars next to each file and folder in the tree.

```json
{
  "file": "app/chat/[personaSlug]/components/wizard-modal/index.tsx",
  "featureWeights": [
    { "featureId": "prompt-wizard", "weight": 0.7 },
    { "featureId": "image-generation", "weight": 0.3 }
  ]
}
```

- `file` — relative file path from repo root
- `featureWeights[]` — AI-estimated feature breakdown. All weights for a file must sum to 1.0
  - `featureId` — references a feature id from `features[]`, or the special value `"__infrastructure__"` for config/shared/unassigned files
  - `weight` — proportion (0.0–1.0)

### The `__infrastructure__` pseudo-feature

Files that don't belong to any product feature (config, build tooling, shared infra, generic type definitions) are assigned `"__infrastructure__"` with weight 1.0 (or partial weight if the file also serves a real feature). Rendered as neutral gray in the visualizer.

### Notes

- This key is **optional**. If missing, the Feature Map tab shows a "no data" message. All other tabs work normally.
- No existing keys are modified by this feature. Removing the `fileTree` key fully removes the feature.
- Feature weights are the AI's independent estimate — they don't need to be consistent with compartment-to-feature mappings.

## ID Convention

All IDs are kebab-case and globally unique within their array. Use descriptive names:
- Surface: `"explore-feed"`, `"admin-review"`, `"chat-thread"`
- Feature: `"prompt-wizard"`, `"star-credits"`, `"content-unlock"`, `"like-system"`
- Entity: `"user"`, `"blog-post"`, `"auth-session"`
- Relationship: `"user-has-many-posts"`, `"post-belongs-to-user"`
- Operation: `"create-post"`, `"delete-comment"`, `"publish-draft"`
- Flow: `"post-creation-flow"`, `"user-onboarding-flow"`
- Compartment: `"image-generation-pipeline"`, `"auth-system"`, `"project-infrastructure"`, `"ui-primitives"`

## Confidence Guidelines

- **high** — explicit definition found (Prisma model, typed interface, named route handler, server action)
- **medium** — inferred from usage patterns (e.g., a type used in API responses but not explicitly defined as an entity)
- **low** — best guess from limited evidence (e.g., an object shape seen once in a function parameter)
