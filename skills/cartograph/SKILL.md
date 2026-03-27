---
name: cartograph
description: >
  Map any TypeScript/JS web app codebase into a structured vocabulary of surfaces,
  features, entities, relationships, operations, flows, compartments, and tech stack. Surfaces are
  entry points (pages/routes); features are standalone capabilities embedded across
  surfaces (e.g., "Prompt Wizard", "Star Credits", "Content Unlock"); compartments are
  logical groupings of related code files that bridge product concepts to the underlying
  codebase. Produces a JSON file and an interactive HTML visualizer. Use when the user
  wants to understand a codebase, map its structure, see what entities exist, understand
  data flows, examine code organization, or get a high-level overview of what an app does.
  Triggers on: "map this codebase", "what does this app do", "show me the entities",
  "cartograph", "extract the structure", "codebase overview", "understand this repo",
  "code map", "show me the code structure".
---

# Cartograph

Extract a structural map of any TypeScript/JS web app: surfaces, features, entities, relationships, operations, flows, compartments, and tech stack. Four orthogonal axes — surfaces are where you go (pages/entry points), features are what you can do (standalone capabilities), entities are what the app works with (data), and compartments are how the code is organized (logical file groupings that bridge product concepts to the underlying codebase). The tech stack provides a comprehensive inventory of all technologies, frameworks, and libraries the project uses.

## Workflow

The workflow runs in **5 waves plus a parallel health pass (Wave 3.5)**. Within each wave, spawn the listed agents **in parallel**, wait for all of them to finish, then move to the next wave. Each agent should return its results as a JSON array (or arrays) matching the schema in `references/json-schema.md`. Between waves, you are the orchestrator — collect agent outputs and pass them as context to the next wave's agents.

---

### Wave 0: Discover Codebase Structure

Run this yourself (no agent needed — it's fast and every later agent needs the results).

1. Read `package.json` for project name and dependencies (framework detection)
2. Glob for key structural files:
   - Schema: `**/*.prisma`, `**/schema.*`, `**/models/**`
   - Routes/Pages: `app/**/page.{tsx,ts,jsx,js}`, `app/api/**/*.{ts,js}`, `pages/**/*.{tsx,ts}`
   - Server actions: grep for `"use server"`
   - Components: `components/**/*.{tsx,jsx}`
   - Lib/services: `lib/**/*.{ts,js}`, `services/**/*.{ts,js}`
3. Read the directory tree to understand the overall shape
4. **Detect the tech stack** — scan `package.json` (dependencies + devDependencies), config files, and project structure to identify all technologies in use. For each detected technology, record:
   - **Name and version**: from `package.json` dependency entries
   - **Category**: language, framework, styling, database, auth, api, testing, deployment, ai, payments, monitoring, or other (see `references/json-schema.md` for the full category list)
   - **Source**: where the technology was detected (e.g., `"package.json"`, `"tailwind.config.ts"`)
   - **Confidence**: high (explicit dependency + config file), medium (dependency only), low (inferred from patterns)

   Detection signals:
   - `package.json` dependencies and devDependencies (primary source — extract version numbers)
   - Config files: `tsconfig.json` (TypeScript), `tailwind.config.*` (Tailwind), `prisma/schema.prisma` (Prisma), `next.config.*` (Next.js), `drizzle.config.*` (Drizzle), `.env*` files (service integrations), `jest.config.*` / `vitest.config.*` (testing), `playwright.config.*` (E2E testing), `Dockerfile` / `docker-compose.*` (Docker), `vercel.json` (Vercel), `sentry.*.config.*` (Sentry)
   - File patterns: `*.module.css` (CSS Modules), `*.scss` / `*.sass` (Sass), `*.graphql` / `*.gql` (GraphQL)
   - Import patterns in code: `@clerk/*`, `@auth/*`, `@stripe/*`, `openai`, `@anthropic-ai/*`, etc.

   Return the tech stack as a JSON array conforming to the `techStack[]` schema in `references/json-schema.md`.
5. Collect the full file inventory (all non-generated files). This is the "discover bundle" — pass it to every agent in later waves along with the detected tech stack.

---

### Wave 1: Surfaces + Entities (parallel)

Spawn **two agents in parallel**, wait for both to finish:

#### Agent 1 — Surfaces

Give this agent the discover bundle and ask it to identify all surfaces.

Surfaces are the top-level organizational axis — self-contained entry points or standalone pieces of functionality. Each app is fundamentally a collection of surfaces.

The agent should:
1. Walk the route tree (`app/**/page.tsx`) and identify each distinct user-facing experience
2. Group related routes into surfaces (e.g., `/create` + `/create/[id]/edit` = one "Creation Studio" surface)
3. Look for admin-only areas, standalone tools, dashboards, and onboarding flows
4. For each surface, determine:
   - **Entrypoint**: main page file and route
   - **Actor**: who uses it (user/admin/system)
   - **Description**: what this surface does as a standalone experience
5. Return: a JSON array of surfaces (without `entityIds`, `operationIds`, `flowIds`, or `compartmentIds` yet — those get populated in later waves)

#### Agent 2 — Entities + Relationships

Give this agent the discover bundle (specifically the schema/type file paths) and ask it to extract all entities and relationships.

**Entities** — read schema/type definitions and extract domain objects:
1. **DB models** (high confidence) — Prisma models, TypeORM entities, Mongoose schemas
2. **TypeScript types/interfaces** (medium confidence) — types used as API payloads, form data, state
3. **Enums** (high confidence) — enum definitions representing domain concepts
4. **Derived types** (medium confidence) — transformed versions (e.g., `PostWithAuthor`)

For each entity: id, name, kind, description, source location, key fields (3-8 most important), confidence.

**Relationships** — map connections between entities:
1. Foreign keys and references in schema → `has-many`, `belongs-to`, `has-one`
2. Nested includes/joins → confirms relationships
3. Type compositions → `derives-from`
4. Looser references → `references`

Return: two JSON arrays — `entities` and `relationships`.

---

### Wave 2: Features + Operations (parallel)

Spawn **two agents in parallel**, wait for both to finish. Pass each agent the discover bundle plus the Wave 1 outputs (surfaces, entities).

#### Agent 3 — Features

Give this agent the discover bundle, surfaces, and entities. Ask it to extract all features.

Features are standalone capabilities embedded within surfaces. They're not pages — they're the reusable functional building blocks that surfaces compose. A surface is "where you go"; a feature is "what you can do there."

Look for these patterns:
1. **Tools** — interactive multi-step experiences (wizards, editors, sandboxes). Look for modal components, multi-step forms, stateful composition flows
2. **Interactions** — single-action engagement patterns (like, save, follow, share). Look for optimistic-update hooks, toggle actions, engagement server actions
3. **Transactions** — money/credit flows (purchase, tip, unlock). Look for payment integrations, credit deduction/grant logic, checkout flows
4. **Gates** — access control mechanisms (age verification, NSFW filtering, auth walls). Look for middleware, overlay components, confirmation dialogs
5. **Infrastructure** — backend capabilities used by other features (AI generation, media processing, webhook handlers). Look for polling loops, queue submissions, external API clients
6. **Workflows** — multi-step admin/system processes (content review, scan pipelines, approval queues). Look for status machines, review UIs, batch processing

For each feature:
- **Name and description**: what this feature does as a standalone capability
- **Kind**: tool, interaction, transaction, gate, infrastructure, or workflow
- **surfaceIds**: which surfaces embed this feature
- **entityIds**: which entities this feature reads/writes
- **implementations**: key files (2-5 most important, not every file)

Features should feel independently describable — "the like system", "the prompt wizard", "the star credit system". If you can't describe it without referencing a specific page, it's probably part of a surface, not a feature.

**Separate implementations = separate features.** The same conceptual capability often exists as independent implementations in different surfaces — for example, a user-facing "Prompt Wizard" modal in chat and an admin "Prompt Remix Wizard" panel in the post management area. Always create a separate feature entry for each distinct implementation, even when they serve the same conceptual purpose. Two implementations are separate features if they have different UI components, different actors, or different capabilities. Name them distinctly. To avoid missing these: after extracting features from one surface, scan every other surface's component tree for similar patterns and grep for shared service imports.

Return: a JSON array of features (without `compartmentIds` yet — that gets populated later).

#### Agent 4 — Operations

Give this agent the discover bundle, entities, and the list of route/action/API files from discover. Ask it to identify all operations.

For each entry point (route handler, server action, API endpoint):
1. Which entity it targets
2. Operation type: `create`, `read`, `update`, `delete`, or `domain`
3. Descriptive name (e.g., "Publish Post", "Generate Preview")
4. Side effects on other entities
5. Implementation location (file + function)

Return: a JSON array of operations.

---

### Wave 3: Flows + Compartments + File Tree (parallel)

Spawn **three agents in parallel**, wait for all to finish. Pass each agent the discover bundle plus all Wave 1 and Wave 2 outputs (surfaces, entities, relationships, features, operations).

#### Agent 5 — Flows

Give this agent all prior outputs and ask it to synthesize flows.

1. Start from UI pages — what can a user do on each page?
2. Trace: UI action → handler → service → DB
3. Name each flow by its user-visible goal
4. Identify trigger and actor (user/admin/system)
5. List steps in order, linking to operations and entities

Return: a JSON array of flows.

#### Agent 6 — Compartments

Give this agent the discover bundle (especially the full file inventory), plus surfaces, features, entities, and operations. Ask it to group every non-generated file into compartments.

Compartments are logical groupings of related files that form cohesive units of functionality. They bridge the product-side view (surfaces, features) with the underlying code structure, so a developer can navigate from "what does this feature do?" to "where does that code live?"

The agent should:
1. Scan the full codebase file tree, using the already-extracted surfaces, features, entities, and operations as context
2. Group files into compartments using AI judgment based on multiple signals:
   - **Folder structure** — files in the same directory or subtree often belong together
   - **Import graph** — files that heavily import each other are likely in the same compartment
   - **Feature alignment** — files belonging to a feature should cluster into compartments that map to those features
   - **Domain proximity** — files dealing with the same entity or business concept belong together
   - **Naming conventions** — files with related names (e.g., `image-*.ts`, `*-generation.*`) suggest a compartment
   - **Shared infrastructure** — truly shared files (used by 3+ features) may warrant their own compartment or may appear in multiple compartments
3. Compartments are **nestable** — sub-compartments can be nested to any depth the AI deems appropriate. A typical web app might have 2–3 levels
4. Files are **non-exclusive** — a file can appear in multiple compartments (e.g., `lib/prisma.ts` in both "Database Access" and "Shared Infrastructure")
5. **Every file must appear in at least one compartment.** Config files, build tooling, etc. go into a "Project Infrastructure" compartment. Exclude generated files (`generated/`, `node_modules/`, `.next/`, `dist/`)
6. For each compartment, determine:
   - **Name and description**: what this code area does (name after what it does, not folder names — "Image Generation Pipeline" not "app/chat/actions")
   - **Tags**: semi-structured tags from the suggested vocabulary (see json-schema.md), plus custom tags as needed
   - **Files**: all files in this compartment with their role (component, hook, action, api, lib, type, config, style, test, other)
   - **parentId**: ID of parent compartment (null for top-level)
   - **featureIds**: which features this compartment implements
   - **surfaceIds**: which surfaces this compartment serves

**Compartment guidelines:**
- Don't create compartments with only 1 file unless it's a genuinely standalone module. Merge small groupings into their parent.
- Keep top-level compartments to 8–15 for a typical web app. More sub-compartments are fine.
- Prefer meaningful groupings over 1:1 folder mapping. If a folder contains unrelated files, split them. If related files span folders, group them.

Return: a JSON array of compartments (without `dependsOn` yet — that gets populated in Wave 4).

#### Agent 7 — File Tree Feature Weights

*(Experimental — fully isolated from other data. See `specs/spec-file-tree.md` for the full spec.)*

Give this agent the discover bundle (full file inventory) and the features array from Wave 2. Ask it to estimate, for every non-generated file, what percentage of the file's purpose is attributable to each feature.

The agent should:
1. Take the list of all non-generated files from the discover bundle.
2. For each file, read the file (or a representative sample for very large files) and estimate what proportion of the file serves each feature.
3. Files that don't belong to any product feature get `"__infrastructure__"` as their sole feature weight.
4. Files serving multiple features get proportional weights (e.g., a shared hook → 50/50).
5. All weights for a file must sum to 1.0.

**Estimation guidance:**
- Look at imports, function names, component names, and the overall purpose of the file.
- A file 100% dedicated to one feature → `[{featureId: "that-feature", weight: 1.0}]`.
- A shared utility used by multiple features → split proportionally.
- Config files, generic type definitions, build config, middleware → `__infrastructure__`.
- Prefer fewer features per file with higher weights over many features with tiny weights.

Return: a JSON array of `{file, featureWeights: [{featureId, weight}]}` entries — one per file.

---

### Wave 3.5: Code Health (parallel)

Spawn **three agents in parallel**, wait for all of them to finish. Pass each agent the discover bundle plus the relevant outputs from Waves 1–3.

#### Agent 8 — Co-location Analysis

Give this agent the discover bundle (especially the full file inventory), plus surfaces, features, compartments, and the project's co-location rules from `AGENTS.md`, `CLAUDE.md`, or equivalent repo instructions.

The agent should:
1. Read project instructions and extract explicit co-location conventions. If none are found, fall back to these universal heuristics:
   - Files used by a single surface should live inside that surface's directory
   - Files shared by multiple surfaces but representing one capability belong in `features/<capability>/`
   - Root `components/`, `lib/`, and `actions/` are reserved for truly global code used by 3+ surfaces/features
   - `components/ui/*` is always exempt and considered correctly placed
2. Evaluate every non-generated, non-infrastructure file:
   - Trace which files import it
   - Determine which surfaces/features actually consume it
   - Compare its current location to where it should live per the rules
   - Assign a binary `pass` / `fail` verdict
3. For each failing file, emit a concrete recommendation:
   - `"move"` when the file should be co-located inside a surface or feature directory
   - `"promote"` when the file should move up into `features/` because it serves multiple surfaces
4. Compute the score as `(passing files / total evaluated files) * 100`

Return: one metric object with:
- `id: "co-location"`
- `name`, `description`, `score`, `thresholds`, `summary`
- `findings[]` in the co-location finding shape from `references/json-schema.md`

#### Agent 9 — DRYness Analysis

Give this agent the discover bundle, plus surfaces, features, entities, operations, and compartments.

The agent should:
1. Use features and compartments as the starting map of the codebase's functional areas
2. Look for candidate duplication before reading file contents:
   - Features with the same `kind` and overlapping `entityIds` across different surfaces
   - Files in different surfaces with similar names or import patterns
   - Compartments with similar descriptions, tags, or overlapping `featureIds`
   - Hooks/actions/clients that wrap the same external API or workflow
3. Read the candidate implementations to confirm real overlap, weighing both:
   - **Functional overlap** — same product problem solved twice
   - **Structural similarity** — same technical pattern repeated with light variation
4. For each confirmed duplication finding, decide:
   - What logic is genuinely shared
   - What must stay implementation-specific
   - Where the shared logic should live, respecting the project's co-location rules
5. Compute the score with:
   - `K = 200 / totalNonInfrastructureFiles`
   - `score = max(0, 100 - (findingCount * K))`

Return: one metric object with:
- `id: "dryness"`
- `name`, `description`, `score`, `thresholds`, `summary`
- `scalingFactor`
- `findings[]` in the DRYness finding shape from `references/json-schema.md`

#### Agent 10 — Dead Code Analysis

Give this agent the discover bundle, plus surfaces, features, entities, operations, and compartments.

The agent should:
1. Build an import map for every non-generated file in the discover bundle:
   - Record which files import each file
   - Resolve relative imports and `@/` path aliases
2. Identify files that are always considered live entry points:
   - `app/**/page.{ts,tsx,js,jsx}`
   - `app/**/layout.{ts,tsx,js,jsx}`
   - `app/api/**/*.{ts,js}`
   - files containing a `"use server"` directive
   - config roots such as `*.config.*`, `next.config.*`, `tailwind.config.*`, `postcss.config.*`, `tsconfig.*`, `package.json`, `.env*`, `middleware.ts`
   - root app entry files such as `app/globals.css` and `app/manifest.ts`
3. Detect dead files and test-only files:
   - For each non-entry-point, non-generated file with zero importers, emit a `dead-file` finding
   - For each non-entry-point, non-test file whose importers are all test files, emit a `test-only-file` finding
   - Test-only files are informational and do not affect the score
4. Detect orphaned surfaces:
   - Search the codebase for navigation references to each surface route (`Link`, `href`, `router.push`, `router.replace`, `redirect`, nav config arrays)
   - Emit an `orphaned-surface` finding when a surface has no inbound navigation references
   - Exempt the root route, auth callback routes, and webhook or API-only routes
5. Detect orphaned features:
   - Emit an `orphaned-feature` finding when `surfaceIds` is empty or every referenced surface is orphaned
   - Include implementation file deadness as secondary evidence when all implementation files are dead
6. Detect dead entities:
   - Evaluate only entities with `kind` of `"db-model"` or `"dto"`
   - Emit a `dead-entity` finding when no operation references the entity, no feature references it, and no Prisma query references it
   - Exempt enums from dead-entity analysis
7. Compute the score with:
   - `totalEvaluated = filesEvaluated + surfacesEvaluated + featuresEvaluated + entitiesEvaluated`
   - `deadItems = deadFiles + orphanedSurfaces + orphanedFeatures + deadEntities`
   - `score = ((totalEvaluated - deadItems) / totalEvaluated) * 100`
   - Exclude test-only files from both numerator and denominator

Return: one metric object with:
- `id: "dead-code"`
- `name`, `description`, `score`, `thresholds`, `summary`
- `findings[]` in the dead-code finding shape from `references/json-schema.md`

---

### Wave 4: Compartment Dependencies

Spawn **one agent**. Pass it the compartments array from Wave 3, plus surfaces and features from earlier waves.

The agent should:
1. Walk the imports of every file in every compartment
2. Map each imported file to the compartment(s) it belongs to
3. Record these as `dependsOn` edges on each compartment (only inter-compartment, not self-references)
4. Populate `compartmentIds` on features — for each feature, determine which compartments implement it
5. Populate `compartmentIds` on surfaces — for each surface, determine which compartments serve it

Return: the updated compartments array (with `dependsOn` populated), plus a `featureCompartmentIds` map and a `surfaceCompartmentIds` map.

---

### Wave 5: Assemble + Output

Run this yourself (no agent needed). Merge all agent outputs into the final JSON:

1. Take the surfaces array and populate:
   - `entityIds`: from entities referenced in that surface's routes/pages
   - `operationIds`: from operations triggered within that surface
   - `flowIds`: from flows that belong to that surface
   - `compartmentIds`: from the Wave 4 mapping
2. Take the features array and populate:
   - `compartmentIds`: from the Wave 4 mapping
   - Set `"files": []` (empty array — `compartmentIds` is the primary code-mapping mechanism; `files` is kept for backwards compatibility)
3. Include the `techStack` array from Wave 0 as-is (no transformation needed)
4. Include the `fileTree` array from Agent 7 as-is (no transformation needed)
5. Add a top-level `codeHealth` object:
   - `codeHealth.analyzedAt = ISO timestamp`
   - `codeHealth.metrics = [coLocationMetric, drynessMetric, deadCodeMetric]`
6. Assemble the final JSON following the schema in `references/json-schema.md`
7. Write `cartograph.json` to the repo root
8. Tell the user: "Open the visualizer (`assets/visualizer.html` in this skill's directory) in your browser and load `cartograph.json` via the file picker."

## Important

- **Read-only** — never modify the codebase being analyzed
- **Prefer inclusion** — when unsure, include with lower confidence
- **Plain language** — descriptions should be understandable by a PM
- **Relative paths** — all file paths relative to repo root
- **Large repos** — analyze by feature/route directory and merge
- **Agent outputs are JSON** — each agent returns its results as JSON arrays conforming to the schema, making it easy to merge in Wave 5
- See `references/json-schema.md` for the exact output format
