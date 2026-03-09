---
shaping: true
pipeline_id: 20260306-p1-insight-pipeline
stage: impl-plan
---

# Implementation Plan: P1 Insight Creation Pipeline

**Goal:** Index 26 existing research reports as insight nodes in the knowledge graph via `tankyu insight from-report`, and update the dive skill to self-index future reports.

**Architecture:** File + Graph Pointer Pattern — reports stay as markdown files, insight nodes are graph pointers with metadata and edges. `ReportFrontMatterSchema` is the machine-readable contract. `InsightFromReport` is the domain orchestrator; `report-parser.ts` handles all file I/O.

**Tech Stack:** TypeScript + ESM, Zod v4 (`import { z } from 'zod/v4'`), js-yaml (already installed), Commander.js, vitest, tsx.

**Appetite:** Small (5 sessions across 3 waves).

---

## Wave Structure

```
Wave 0 (serial)
  └── 0.1: schema-foundation   [types, ports, stores, insight-writer]

Wave 1 (serial after W0)
  └── 1.1: domain-core         [report-parser, InsightFromReport class + batch + dry-run]

Wave 2 (parallel after W1)
  ├── 2.1: cli-insight         [CLI command + program.ts wire-up]
  ├── 2.2: retro-script        [scripts/add-front-matter.ts]
  └── 2.3: dive-skill-spec     [SKILL.md update + report-front-matter-spec.md]
```

---

## Wave 0: Foundation

### Task 0.1: Schema Foundation

**Topic:** `p1-schema-foundation`
**Worktree:** `git worktree add .claude/worktrees/p1-schema-foundation -b p1-schema-foundation`
**Depends on:** None
**Complexity:** Medium

**Files to create:**
- `src/domain/types/report.ts` — `ReportFrontMatterSchema` (Zod v4)
- `src/domain/types/report.test.ts` — schema unit tests

**Files to modify:**
- `src/domain/types/insight.ts` — remove `topicId: z.string().uuid()` field
- `src/domain/types/insight.test.ts` — remove `topicId` from `validInsight` fixture; remove `topicId` rejection test; add test confirming `topicId` is NOT a valid field
- `src/domain/ports/index.ts` — remove `listByTopic(topicId: string): Promise<Insight[]>` from `IInsightStore`
- `src/infrastructure/stores/insight-store.ts` — remove `listByTopic` method
- `src/infrastructure/stores/insight-store.test.ts` — remove `listByTopic` test cases
- `src/features/research/insight-writer.ts` — `topicId: string` → `topicIds: string[]` in `ResearchNoteInput` and `SynthesisInput`; update `writeResearchNote` and `writeSynthesis` to create one `tagged-with` edge per topicId (no topicId field on the Insight object itself)
- `src/features/research/insight-writer.test.ts` — update for new `topicIds[]` interface

**Steps:**

1. Create `src/domain/types/report.ts` with `ReportFrontMatterSchema`:
   ```typescript
   import { z } from 'zod/v4';
   export const ReportTypeSchema = z.enum(['research-note', 'synthesis', 'briefing']);
   export const ReportFrontMatterSchema = z.object({
     id: z.string().uuid().optional(),        // absent on first index; written back after
     type: ReportTypeSchema,
     title: z.string().min(1),
     date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
     topics: z.array(z.string().min(1)).min(1),
     subject: z.string().min(1),
     subject_url: z.string().url().optional(),
     tags: z.array(z.string()),
     key_points: z.array(z.string().min(1)),
     relates_to: z.array(z.string().uuid()).default([]),
     status: z.enum(['draft', 'complete']),
   });
   export type ReportFrontMatter = z.infer<typeof ReportFrontMatterSchema>;
   ```

2. Write `src/domain/types/report.test.ts` covering:
   - Valid fully-populated block passes
   - Valid block without `id` (optional) passes
   - Valid block without `subject_url` (optional) passes
   - `topics: []` fails (min 1)
   - Missing required field (title, date, topics, subject, key_points, status) fails
   - `date` not matching `YYYY-MM-DD` pattern fails
   - `status` not in enum fails
   - `id` present but not UUID fails
   - `relates_to` defaults to `[]` when absent

3. Remove `topicId: z.string().uuid()` from `InsightSchema` in `src/domain/types/insight.ts`. Run `npm run typecheck` — expect compile errors in insight-writer.ts and insight-store.ts; fix those next.

4. Update `src/domain/types/insight.test.ts`:
   - Remove `topicId` from `validInsight` fixture
   - Remove the `it('rejects non-UUID topicId', ...)` test
   - Add `it('rejects unknown topicId field', () => { expect(InsightSchema.safeParse({ ...validInsight, topicId: 'xxx' }).success).toBe(false); })` — this should PASS (Zod v4 strips unknown fields by default; confirm this expectation with the actual schema behavior)

5. Remove `listByTopic(topicId: string): Promise<Insight[]>` from `IInsightStore` interface in `src/domain/ports/index.ts`.

6. Remove the `listByTopic` method from `src/infrastructure/stores/insight-store.ts` and its test file.

7. Update `src/features/research/insight-writer.ts`:
   - Change `ResearchNoteInput.topicId: string` → `topicIds: string[]`
   - Change `SynthesisInput.topicId: string` → `topicIds: string[]`
   - In `writeResearchNote`: remove `topicId` from the `Insight` object; after `insightStore.create`, loop over `topicIds` and call `graphStore.addEdge({ edgeType: 'tagged-with', fromId: insight.id, fromType: 'insight', toId: topicId, toType: 'topic', reason: 'Research note indexed under topic', createdAt: now })` for each
   - In `writeSynthesis`: same treatment for `topicIds`
   - Remove `topicId` from the `Insight` object literals in both methods

8. Update `src/features/research/insight-writer.test.ts` to pass `topicIds: ['<uuid>']` instead of `topicId: '<uuid>'`. Verify `tagged-with` edges are created (mock graphStore).

9. Run `npm run typecheck && npm test` — all tests pass, no type errors.

**Acceptance:**
- `npm run typecheck` exits 0
- `npm test` passes (all existing tests pass with fixture updates; new report.test.ts passes)
- `InsightSchema` does not contain `topicId` field
- `IInsightStore` does not declare `listByTopic`
- `InsightWriter` accepts `topicIds: string[]` and creates one `tagged-with` edge per element

---

**Session Prompt:**

```
You are working on the Tankyu research intelligence tool (TypeScript, ESM, Zod v4).
Your task: Task 0.1 — Schema Foundation for the P1 Insight Creation Pipeline.

Context files to read first:
- docs/workspace/20260306-p1-insight-pipeline/shaping.md — Shape A parts A1/A9
- docs/workspace/20260306-p1-insight-pipeline/breadboard.md — V1 slice details
- src/domain/types/insight.ts — current InsightSchema (you will remove topicId)
- src/domain/types/insight.test.ts — current test fixture (update to remove topicId)
- src/domain/ports/index.ts — IInsightStore (remove listByTopic)
- src/infrastructure/stores/insight-store.ts — remove listByTopic implementation
- src/infrastructure/stores/insight-store.test.ts — remove listByTopic tests
- src/features/research/insight-writer.ts — update topicId → topicIds[]
- src/features/research/insight-writer.test.ts — update test fixtures

Key conventions:
- Zod v4: import { z } from 'zod/v4'
- ESM-only: all imports use .js extensions
- Path aliases: @domain/*, @infra/*, @features/*, @shared/*, @cli/*
- Tests colocated: *.test.ts next to source

What to build:
1. NEW: src/domain/types/report.ts — ReportFrontMatterSchema (see breadboard V1 for field spec)
   Fields: id (optional uuid), type (research-note|synthesis|briefing), title (min 1),
   date (YYYY-MM-DD regex), topics (string[] min 1), subject (min 1),
   subject_url (optional url), tags (string[]), key_points (string[] min 1),
   relates_to (uuid[] default []), status (draft|complete)
2. NEW: src/domain/types/report.test.ts — unit tests for the schema
3. MODIFY: src/domain/types/insight.ts — remove topicId field from InsightSchema
4. MODIFY: src/domain/types/insight.test.ts — remove topicId from fixtures and tests
5. MODIFY: src/domain/ports/index.ts — remove listByTopic from IInsightStore
6. MODIFY: src/infrastructure/stores/insight-store.ts — remove listByTopic method
7. MODIFY: src/infrastructure/stores/insight-store.test.ts — remove listByTopic tests
8. MODIFY: src/features/research/insight-writer.ts — topicId → topicIds: string[] in
   ResearchNoteInput and SynthesisInput; create one tagged-with edge per topicId;
   remove topicId from the Insight object literal
9. MODIFY: src/features/research/insight-writer.test.ts — update fixtures

EdgeSchema fields (from src/domain/types/edge.ts):
  { id: uuid, fromId, fromType, toId, toType, edgeType, reason: string (min 1), createdAt }
EdgeType 'tagged-with' is already in EdgeTypeSchema.
NodeType 'insight' and 'topic' are already in NodeTypeSchema.

Verify: npm run typecheck && npm test — all pass, no errors.
```

---

## Wave 1: Domain Core

### Task 1.1: Domain Core

**Topic:** `p1-domain-core`
**Worktree:** `git worktree add .claude/worktrees/p1-domain-core -b p1-domain-core`
**Depends on:** `0.1-schema-foundation` (merged to main before starting)
**Complexity:** High

**Files to create:**
- `src/features/research/report-parser.ts` — file I/O layer: `parseReport`, `hasFrontMatter`, `writeIdBack`
- `src/features/research/report-parser.test.ts` — unit tests using tmp files
- `src/features/research/insight-from-report.ts` — `InsightFromReport` class: `ingest()` + `ingestDirectory()`
- `src/features/research/insight-from-report.test.ts` — integration tests with mocked stores

**Steps:**

1. Create `src/features/research/report-parser.ts`:

   ```typescript
   import fs from 'fs/promises';
   import yaml from 'js-yaml';
   import { ReportFrontMatterSchema, type ReportFrontMatter } from '@domain/types/report.js';
   import { TankyuError } from '@shared/lib/errors.js';

   export async function hasFrontMatter(filePath: string): Promise<boolean> {
     const content = await fs.readFile(filePath, 'utf-8');
     return content.startsWith('---');
   }

   export async function parseReport(filePath: string): Promise<ReportFrontMatter> {
     const content = await fs.readFile(filePath, 'utf-8');
     if (!content.startsWith('---')) {
       throw new TankyuError(`No YAML front matter in ${filePath}. Add it with: tsx scripts/add-front-matter.ts ${filePath} --topic <name>`);
     }
     const end = content.indexOf('\n---', 3);
     if (end === -1) throw new TankyuError(`Malformed front matter in ${filePath}: no closing ---`);
     const yamlBlock = content.slice(4, end);
     const raw = yaml.load(yamlBlock);
     const result = ReportFrontMatterSchema.safeParse(raw);
     if (!result.success) {
       const issues = result.error.issues.map(i => `${i.path.join('.')}: ${i.message}`).join('; ');
       throw new TankyuError(`Invalid front matter in ${filePath}: ${issues}`);
     }
     return result.data;
   }

   export async function writeIdBack(filePath: string, id: string): Promise<void> {
     const content = await fs.readFile(filePath, 'utf-8');
     // Insert `id: <uuid>` as first field after opening `---\n`
     const updated = content.replace(/^---\n/, `---\nid: ${id}\n`);
     await fs.writeFile(filePath, updated, 'utf-8');
   }
   ```

2. Write `src/features/research/report-parser.test.ts` covering:
   - `hasFrontMatter` returns true for file starting with `---`
   - `hasFrontMatter` returns false for file without front matter
   - `parseReport` returns valid `ReportFrontMatter` for a well-formed file
   - `parseReport` throws TankyuError if no front matter
   - `parseReport` throws TankyuError if front matter is missing required field
   - `writeIdBack` inserts `id: <uuid>` as first line after `---`
   - `writeIdBack` on a file that already has `id:` does not double-insert (idempotent via the replace pattern — note: the `replace` only replaces the first `---\n`, so if `id:` is already there it won't add again. Verify this edge case.)

   Use `tmp` dir via `os.tmpdir()` + `fs.mkdtemp` + `fs.writeFile` for real file I/O in tests; clean up in `afterEach`.

3. Create `src/features/research/insight-from-report.ts`:

   Define types:
   ```typescript
   export interface IngestReportResult {
     filePath: string;
     insightId: string;
     title: string;
     action: 'created' | 'updated';
   }
   export interface BatchError {
     filePath: string;
     error: string;
   }
   export interface BatchResult {
     results: IngestReportResult[];
     errors: BatchError[];
   }
   export interface IngestOptions {
     dryRun?: boolean;
   }
   ```

   `InsightFromReport` constructor takes `(insightStore: IInsightStore, graphStore: IGraphStore, topicManager: TopicManager)`.

   `async ingest(filePath: string, opts: IngestOptions = {}): Promise<IngestReportResult>`:
   - Call `hasFrontMatter(filePath)` — throws if false
   - Call `parseReport(filePath)` → `fm`
   - For each name in `fm.topics`: call `topicManager.get(name)` — throws `NotFoundError` with message augmented to include `tankyu topic create <name>` hint
   - If `fm.id` exists: call `insightStore.get(fm.id)` → existing?
   - Call `computeBody(fm.key_points, fm.subject)` → body string
   - Build `Insight` object (no `topicId` field): `{ id: fm.id ?? uuidv4(), type: fm.type, title: fm.title, body, keyPoints: fm.key_points, citations: [], createdAt: now, updatedAt: now }`
   - **If dry-run**: return `{ filePath, insightId: insight.id, title: insight.title, action: existing ? 'updated' : 'created' }` — no writes
   - If new: `insightStore.create(insight)`, `writeIdBack(filePath, insight.id)`
   - If existing: `insightStore.update(insight.id, { title, body, keyPoints, updatedAt })` — do NOT call writeIdBack (UUID already in file)
   - Edge idempotency: `getEdgesByNode(insight.id)` → filter `e.edgeType === 'tagged-with'` → `removeEdge(e.id)` for each stale edge → `addEdge(tagged-with)` for each resolved topic
   - Edge for `relates_to`: for each uuid in `fm.relates_to`, `addEdge(relates-to)` (no cleanup for simplicity — relates-to are additive, not replaced)
   - Return `IngestReportResult`

   `computeBody(keyPoints: string[], subject: string): string`:
   - If `keyPoints.length > 0`: `keyPoints.map(p => '• ' + p).join('\n')`
   - Else: `'Research report on ' + subject + '.'`

   `async ingestDirectory(dir: string, opts: IngestOptions = {}): Promise<BatchResult>`:
   - `glob('**/*.md', { cwd: dir })` — use the `glob` package (check if available; if not, use `fs.readdir` with filter)
   - For each file: `await ingest(path.join(dir, file), opts)` in a try/catch; push to `results` or `errors`
   - Return `{ results, errors }`

   **Note on glob package**: Check `package.json` — if `glob` is not installed, use `fs.readdir` recursive (Node 20 supports `{ recursive: true }`) and filter `.md` files manually. Do not add new dependencies without checking first.

4. Write `src/features/research/insight-from-report.test.ts`:
   - Mock `IInsightStore`, `IGraphStore`, `TopicManager` using `vi.fn()`
   - Test `ingest`: new file → creates insight, writes id back, creates tagged-with edges
   - Test `ingest`: existing file (id in fm) → updates insight, clears+recreates tagged-with edges
   - Test `ingest`: dry-run → no writes to stores, no writeIdBack called
   - Test `ingest`: missing front matter → throws TankyuError
   - Test `ingest`: topic not found → throws with `tankyu topic create X` hint in message
   - Test `ingestDirectory`: processes multiple files, collects errors without aborting
   - Test `ingestDirectory` dry-run: returns results, no writes

5. Run `npm run typecheck && npm test` — all pass.

**Acceptance:**
- `parseReport` correctly validates against `ReportFrontMatterSchema`
- `ingest` on a new file: creates insight node + writes UUID back to file + tagged-with edges
- `ingest` on existing file: updates in place, no duplicate edges
- `ingest --dryRun`: reads + validates but zero writes
- `ingestDirectory`: continues past per-file errors, returns `BatchResult`
- All tests pass

---

**Session Prompt:**

```
You are working on Tankyu (TypeScript, ESM, Zod v4). Task 1.1 — Domain Core.
Wave 0 (schema foundation) is already merged. Start by pulling main.

Context files to read first:
- docs/workspace/20260306-p1-insight-pipeline/shaping.md — parts A2, A3, A4, A6
- docs/workspace/20260306-p1-insight-pipeline/breadboard.md — V2, V3, V4 slice details + N10–N24 affordances
- src/domain/types/report.ts — ReportFrontMatterSchema (built in W0)
- src/domain/types/insight.ts — InsightSchema (topicId removed in W0)
- src/domain/ports/index.ts — IInsightStore (listByTopic removed), IGraphStore
- src/domain/types/edge.ts — EdgeSchema, EdgeTypeSchema, NodeTypeSchema
- src/features/topic/topic-manager.ts — TopicManager.get(name) signature
- src/shared/lib/errors.ts — TankyuError, NotFoundError hierarchy
- package.json — check if 'glob' package is installed (use fs.readdir fallback if not)

Key conventions:
- Zod v4: import { z } from 'zod/v4'
- ESM-only: all imports use .js extensions
- Path aliases: @domain/*, @features/*, @shared/*
- js-yaml is already installed: import yaml from 'js-yaml'
- uuid: import { v4 as uuidv4 } from 'uuid'
- Tests colocated, use vitest (vi.fn() for mocks, tmp dirs for file I/O tests)

What to build:
1. NEW: src/features/research/report-parser.ts
   - hasFrontMatter(filePath): Promise<boolean> — checks content.startsWith('---')
   - parseReport(filePath): Promise<ReportFrontMatter> — slices YAML between --- delimiters,
     yaml.load, ReportFrontMatterSchema.safeParse; throw TankyuError with actionable message on failure
   - writeIdBack(filePath, id): Promise<void> — insert 'id: <uuid>' as first line after '---\n'

2. NEW: src/features/research/report-parser.test.ts — use real tmp files (os.tmpdir + mkdtemp)

3. NEW: src/features/research/insight-from-report.ts — InsightFromReport class
   - constructor(insightStore, graphStore, topicManager)
   - ingest(filePath, opts?): Promise<IngestReportResult>
     Full flow: hasFrontMatter → parseReport → topicManager.get × n → insightStore.get →
     computeBody → [if !dryRun: insightStore.create/update + writeIdBack + edge cleanup + addEdge]
   - ingestDirectory(dir, opts?): Promise<BatchResult>
     Glob *.md, per-file try/catch, collect results + errors
   - computeBody(keyPoints, subject): string (private)
   - Dry-run: all reads/validates but zero writes to stores or files

   Exported types: IngestReportResult, BatchError, BatchResult, IngestOptions

   Edge idempotency for tagged-with:
     getEdgesByNode(insightId) → filter edgeType === 'tagged-with' → removeEdge each
     → addEdge tagged-with per resolved topic
   Edge for relates-to: addEdge for each relates_to UUID (additive, no cleanup)

   EdgeSchema required fields: { id: uuidv4(), fromId, fromType, toId, toType, edgeType,
     reason: string, createdAt: ISO string }
   Use reason strings like: 'Research report tagged with topic' and 'Report relates to insight'

4. NEW: src/features/research/insight-from-report.test.ts

Verify: npm run typecheck && npm test — all pass.
```

---

## Wave 2: Surface Layer (parallel)

### Task 2.1: CLI Command

**Topic:** `p1-cli-insight`
**Worktree:** `git worktree add .claude/worktrees/p1-cli-insight -b p1-cli-insight`
**Depends on:** `1.1-domain-core` (merged to main before starting)
**Complexity:** Low

**Files to create:**
- `src/cli/commands/insight.ts`

**Files to modify:**
- `src/cli/program.ts` — add `registerInsightCommand(program)` call

**Steps:**

1. Create `src/cli/commands/insight.ts`:

   ```typescript
   import { Command } from 'commander';
   import fs from 'fs/promises';
   import { buildContext, handleCommandError } from '@cli/utils.js';
   import { InsightFromReport } from '@features/research/insight-from-report.js';
   import { TopicManager } from '@features/topic/topic-manager.js';

   export function registerInsightCommand(parent: Command): void {
     const insight = parent.command('insight').description('Manage insight nodes');

     insight
       .command('from-report <path>')
       .description('Index a research report (or directory of reports) as insight nodes')
       .option('--dry-run', 'Show what would be indexed without writing anything')
       .action(async (inputPath: string, opts: { dryRun?: boolean }) => {
         try {
           const ctx = await buildContext(parent.opts().tankyuDir);
           const indexer = new InsightFromReport(ctx.insightStore, ctx.graphStore, ctx.topicManager);
           const dryRun = !!opts.dryRun;

           const stat = await fs.stat(inputPath);

           if (stat.isFile()) {
             const result = await indexer.ingest(inputPath, { dryRun });
             if (dryRun) {
               console.log(`[dry-run] would ${result.action}: ${result.title} (topics: ${/* need topics from result */})`);
             } else if (result.action === 'created') {
               console.log(`✓ indexed: ${result.title} [${result.insightId}]`);
             } else {
               console.log(`↑ updated: ${result.title}`);
             }
           } else if (stat.isDirectory()) {
             const batch = await indexer.ingestDirectory(inputPath, { dryRun });
             if (dryRun) {
               for (const r of batch.results) {
                 console.log(`[dry-run] would ${r.action}: ${r.title}`);
               }
             }
             for (const e of batch.errors) {
               console.error(`✗ ${e.filePath}: ${e.error}`);
             }
             if (!dryRun) {
               const indexed = batch.results.filter(r => r.action === 'created').length;
               const updated = batch.results.filter(r => r.action === 'updated').length;
               console.log(`${indexed} indexed, ${updated} updated, ${batch.errors.length} errored`);
             }
           } else {
             throw new Error(`${inputPath} is neither a file nor a directory`);
           }
         } catch (err) {
           handleCommandError(err, parent.opts().json ?? false);
         }
       });
   }
   ```

   **Note on dry-run topic display:** For dry-run to show `(topics: X, Y)`, `IngestReportResult` needs to include the resolved topic names or the front matter topics array. Extend `IngestReportResult` with `topics: string[]` in `insight-from-report.ts` if it doesn't already have it. Check the type definition — if it only has `insightId, title, action`, add `topics: string[]` and populate it from `fm.topics` in the `ingest` method.

2. Wire into `src/cli/program.ts`:
   ```typescript
   import { registerInsightCommand } from './commands/insight.js';
   // ...
   registerInsightCommand(program);
   ```

3. Run `npm run typecheck` — exits 0.

4. Smoke test: `npm run dev -- insight from-report --help` — prints command description and `--dry-run` option.

**Acceptance:**
- `tankyu insight from-report <file>` indexes a single file and prints `✓ indexed: <title> [<id>]`
- `tankyu insight from-report <dir>` runs batch mode and prints summary
- `--dry-run` prints preview lines and exits without writing
- `✗ <file>: <message>` printed to stderr for each error; batch continues
- `npm run typecheck` passes

---

**Session Prompt:**

```
You are working on Tankyu (TypeScript, ESM). Task 2.1 — CLI Command for insight from-report.
Waves 0 and 1 are merged. Pull main before starting.

Context files to read first:
- src/cli/program.ts — existing command registration pattern
- src/cli/utils.ts — buildContext(), handleCommandError(), CommandContext type
- src/cli/commands/topic.ts — existing command file as pattern reference
- src/features/research/insight-from-report.ts — InsightFromReport class + IngestReportResult type
- docs/workspace/20260306-p1-insight-pipeline/breadboard.md — N1–N4, U1–U7 affordances (P1 CLI place)

Key conventions:
- ESM-only: all imports use .js extensions
- Path aliases: @cli/*, @features/*, @domain/*
- Commander subcommand pattern: parent.command('insight').command('from-report <path>')
- buildContext(parent.opts().tankyuDir) — reads --tankyu-dir from parent program
- handleCommandError(err, parent.opts().json ?? false)

What to build:
1. NEW: src/cli/commands/insight.ts — registerInsightCommand(parent: Command)
   - Subcommand: insight from-report <path> --dry-run
   - fs.stat(path) to detect file vs directory
   - Single file: call indexer.ingest(); print ✓/↑/dry-run output
   - Directory: call indexer.ingestDirectory(); print per-file dry-run lines + error lines + summary
   - Output format:
     created: '✓ indexed: <title> [<id>]'
     updated: '↑ updated: <title>'
     dry-run: '[dry-run] would <action>: <title>'
     error: '✗ <filepath>: <error message>' (to stderr)
     batch summary: 'N indexed, N updated, N errored'

2. MODIFY: src/cli/program.ts — add registerInsightCommand import + call

Check if IngestReportResult includes a 'topics' field for dry-run display.
If not, you may need to add it to the type in insight-from-report.ts (acceptable small change).

Verify: npm run typecheck exits 0.
Smoke test: npm run dev -- insight from-report --help
```

---

### Task 2.2: Retroactive Script

**Topic:** `p1-retro-script`
**Worktree:** `git worktree add .claude/worktrees/p1-retro-script -b p1-retro-script`
**Depends on:** `1.1-domain-core` (for `hasFrontMatter` import)
**Complexity:** Medium

**Files to create:**
- `scripts/add-front-matter.ts`
- `scripts/add-front-matter.test.ts` — unit tests for extraction functions

**Steps:**

1. Create `scripts/add-front-matter.ts`. This is a standalone `tsx`-runnable script (no Commander dependency needed — use `process.argv` directly):

   ```typescript
   #!/usr/bin/env tsx
   import fs from 'fs/promises';
   import path from 'path';
   import yaml from 'js-yaml';
   import { hasFrontMatter } from '../src/features/research/report-parser.js';

   // Extraction functions (exported for testing)
   export function extractTitle(content: string): string { ... }
   export function extractDate(content: string): string { ... }
   export function extractSubject(filePath: string): string { ... }
   export function extractSubjectUrl(content: string): string | undefined { ... }
   export function extractKeyPoints(content: string): string[] { ... }
   export function buildFrontMatter(fm: Record<string, unknown>): string { ... }
   export async function writeFrontMatter(filePath: string, fm: Record<string, unknown>): Promise<void> { ... }
   async function processFile(filePath: string, topic: string, type: string): Promise<'skipped' | 'wrote'> { ... }
   async function main(): Promise<void> { ... }

   if (process.argv[1] === import.meta.filename) main();
   ```

   **Extraction logic (from shaping A5):**

   - `extractTitle(content)`: Find first line matching `/^# (.+)/` → strip leading `# `. Fallback: `'Untitled'`.
   - `extractDate(content)`: Find line matching `/\*\*Date\*\*:\s*(.+)/` → trim. Fallback: today's date `new Date().toISOString().slice(0, 10)`.
   - `extractSubject(filePath)`: `path.basename(filePath, '.md').replace(/-research-report$/, '').replace(/-research-notes?$/, '')`.
   - `extractSubjectUrl(content)`: Find line matching `/\*\*(Repository|Context)\*\*:\s*(https?:\/\/\S+)/` → capture the URL. Return `undefined` if not found.
   - `extractKeyPoints(content)`:
     1. Find "Executive Summary" or "Recommendations" section (look for `## Executive Summary` or `## Recommendations`), collect bullet lines (`^- ` or `^\* `) until next `##` heading or EOF. Take up to 7.
     2. If none found, collect all `## ` headings (excluding `Executive Summary`, `Linked Sources`, `Open Questions`, `References`, `Introduction`), return them as key_points strings (strip `## ` prefix). Limit 7.
     3. Fallback: `[]` (empty array is valid per schema since min is on `key_points.min(1)` — wait, schema says `key_points: z.array(z.string().min(1))` not `.min(1)` on the array itself, so empty is valid).
   - `buildFrontMatter(fm)`: `'---\n' + yaml.dump(fm, { lineWidth: 120 }) + '---\n\n'`
   - `writeFrontMatter(filePath, fm)`: Read file content, prepend `buildFrontMatter(fm)` to it, write back.

   **`processFile(filePath, topic, type)`:**
   - If `await hasFrontMatter(filePath)`: return `'skipped'`
   - Read content
   - Build fm object: `{ type, title, date, topics: [topic], subject, subject_url (if present), tags: [], key_points, relates_to: [], status: 'draft' }`
   - Note: NO `id` field — UUID is assigned by `from-report` after indexing
   - Call `writeFrontMatter(filePath, fm)`
   - Return `'wrote'`

   **`main()`:**
   - Parse `process.argv`: `[tsx, scriptPath, <path>, --topic, <name>, --type?, <type>]`
   - Validate required `--topic` arg; default type to `'research-note'`
   - If path is a file: `processFile`, print result
   - If path is a directory: `readdir(path, { recursive: false })`, filter `.md`, process each, print per-file results
   - Print summary at end

2. Write `scripts/add-front-matter.test.ts` covering all extraction functions:
   - `extractTitle`: finds `# Heading`; returns `'Untitled'` if none
   - `extractDate`: finds `**Date**: 2026-03-05`; returns today's date if none
   - `extractSubject`: strips `-research-report` suffix; strips `-research-notes` suffix
   - `extractSubjectUrl`: finds `**Repository**: https://...`; finds `**Context**: https://...`; returns undefined if none
   - `extractKeyPoints`: returns bullets from Executive Summary section (limit 7); falls back to H2 titles; excludes boilerplate headings
   - `buildFrontMatter`: output starts with `---\n` and ends with `---\n\n`

3. Run `npm test -- scripts/add-front-matter.test.ts` — all pass.
4. Manual smoke: `tsx scripts/add-front-matter.ts research/nanograph.md --topic tankyu-v2`
   - Prints `Wrote front matter: research/nanograph.md`
   - Re-run: `Skipped (has front matter): research/nanograph.md`

**Acceptance:**
- All extraction functions have unit tests that pass
- `tsx scripts/add-front-matter.ts <file> --topic <name>` prepends valid YAML front matter
- Re-run on a file that already has front matter: prints skipped, does not modify file
- Front matter block is parseable by `ReportFrontMatterSchema` (after adding `id` manually or via `from-report`)
- Script runs without error on all 26 reports in `research/`

---

**Session Prompt:**

```
You are working on Tankyu (TypeScript, ESM). Task 2.2 — Retroactive Front Matter Script.
Waves 0 and 1 are merged. Pull main before starting.

Context files to read first:
- docs/workspace/20260306-p1-insight-pipeline/shaping.md — part A5 (extraction logic spec)
- docs/workspace/20260306-p1-insight-pipeline/breadboard.md — V5 slice + N30–N37 affordances
- src/domain/types/report.ts — ReportFrontMatterSchema (fields to populate)
- src/features/research/report-parser.ts — hasFrontMatter() to import and reuse
- research/ directory — look at 2–3 real reports to understand the actual format

What to build:
1. NEW: scripts/add-front-matter.ts — tsx-runnable, no Commander needed (use process.argv)

   Extraction logic:
   - extractTitle(content): first '# ' heading; fallback 'Untitled'
   - extractDate(content): '**Date**: ' line; fallback today YYYY-MM-DD
   - extractSubject(filePath): basename without .md, strip '-research-report' and '-research-notes?' suffixes
   - extractSubjectUrl(content): '**Repository**: ' or '**Context**: ' line → URL only; undefined if absent
   - extractKeyPoints(content):
     1. Try bullets (- or *) under '## Executive Summary' or '## Recommendations' section (up to next ## or EOF), limit 7
     2. Fallback: collect ## headings excluding ['Executive Summary','Linked Sources','Open Questions','References','Introduction'], limit 7
     3. Final fallback: []
   - writeFrontMatter(filePath, fm): prepend '---\n' + yaml.dump(fm) + '---\n\n' to file
   - processFile(filePath, topic, type): check hasFrontMatter → skip; else extract + write; return 'skipped'|'wrote'
   - main(): parse argv, process file or directory, print per-file results + summary

   Front matter built by script (NO id field — UUID assigned by from-report later):
   { type, title, date, topics: [topic], subject, subject_url?, tags: [], key_points, relates_to: [], status: 'draft' }

   Import hasFrontMatter from '../src/features/research/report-parser.js'
   Import js-yaml: import yaml from 'js-yaml'

2. NEW: scripts/add-front-matter.test.ts — unit tests for all extraction functions

Conventions:
- ESM-only: .js extensions on all imports within src/; direct paths for scripts/
- js-yaml and uuid are already installed

Output format:
  'Wrote front matter: <filename>'
  'Skipped (has front matter): <filename>'
  Summary: 'N wrote, N skipped'

Verify:
  npm test -- --testPathPattern=add-front-matter — all pass
  tsx scripts/add-front-matter.ts research/nanograph.md --topic tankyu-v2 — writes front matter
  Re-run same command — prints skipped
```

---

### Task 2.3: Dive Skill + Spec Doc

**Topic:** `p1-dive-skill-spec`
**Worktree:** `git worktree add .claude/worktrees/p1-dive-skill-spec -b p1-dive-skill-spec`
**Depends on:** `1.1-domain-core` (spec must reference finalized schema and CLI)
**Complexity:** Low

**Files to create:**
- `docs/report-front-matter-spec.md` — authoritative front matter contract (A7)

**Files to modify:**
- `/Users/cmbays/Github/tankyu/skills/dive/SKILL.md` — add report-writing step (A8)

> **IMPORTANT:** `SKILL.md` lives in the main repo at `/Users/cmbays/Github/tankyu/skills/dive/SKILL.md`, NOT in the worktree. Edit it directly there. The `docs/report-front-matter-spec.md` lives inside this worktree and will be merged via PR.

**Steps:**

1. Create `docs/report-front-matter-spec.md`:

   ```markdown
   # Research Report Front Matter Spec

   ## Purpose
   YAML front matter is the machine-readable contract between report authors
   (human or agent) and the `tankyu insight from-report` indexer.

   ## Schema

   | Field         | Type                                   | Required | Description |
   |---------------|----------------------------------------|----------|-------------|
   | id            | UUID string                            | No*      | Assigned on first index; written back to file. Do not set manually. |
   | type          | research-note \| synthesis \| briefing | Yes      | Report classification |
   | title         | string (min 1)                         | Yes      | Human-readable title |
   | date          | YYYY-MM-DD                             | Yes      | Report date |
   | topics        | string[] (min 1 element)               | Yes      | Topic names this report belongs to (must exist in tankyu) |
   | subject       | string (min 1)                         | Yes      | Short slug for what was researched (e.g. repo name, concept) |
   | subject_url   | URL string                             | No       | Canonical URL for the subject (GitHub repo, docs page, etc.) |
   | tags          | string[]                               | Yes      | Free-form tags (may be empty: []) |
   | key_points    | string[] (each min 1 char)             | Yes      | Key findings; used as body text in insight node (may be empty: []) |
   | relates_to    | UUID[]                                 | Yes      | IDs of related insight nodes; creates relates-to graph edges (default: []) |
   | status        | draft \| complete                      | Yes      | Authoring status |

   *`id` is absent on first authoring and written in by the indexer.

   ## Example

   ---yaml block---

   ## What the Retroactive Script Populates

   (describe each field's extraction logic)

   ## What the Dive Skill Must Produce

   (describe dive skill's responsibility)

   ## Indexing Rules

   - `tankyu insight from-report <file>` requires valid front matter; fails with actionable error if absent
   - Add front matter to legacy files: `tsx scripts/add-front-matter.ts <file> --topic <name>`
   - Idempotent: re-running updates the insight node in place, no duplicates
   - Topics must exist before indexing: create with `tankyu topic create <name>`
   ```

   Fill out the full spec with complete field descriptions, a real example block copied from a report after running add-front-matter, and complete indexing rules.

2. Modify `/Users/cmbays/Github/tankyu/skills/dive/SKILL.md` — add a new section before "Conversational Integration":

   ```markdown
   ### Writing a research report

   After completing synthesis for a deep dive:

   1. Write a research report to `research/<topic>-<date>-<slug>-dive-notes.md` with YAML front matter:
      ```yaml
      ---
      type: research-note
      title: <descriptive title>
      date: <YYYY-MM-DD today>
      topics:
        - <topic-name>
      subject: <slug>
      subject_url: <url if applicable>
      tags: []
      key_points:
        - <key finding 1>
        - <key finding 2>
        - ... (up to 7)
      relates_to: []
      status: complete
      ---
      ```
      See `docs/report-front-matter-spec.md` for the full field reference.

   2. After writing the file, index it:
      ```bash
      tankyu insight from-report research/<filename>.md
      ```
      This creates an insight node linked to the topic, writes the UUID back into the file's front matter, and prints `✓ indexed: <title> [<uuid>]`.
   ```

3. Run `npm run typecheck` — no impact (docs only, but verify the worktree is clean).

**Acceptance:**
- `docs/report-front-matter-spec.md` exists with complete field reference, a real example, extraction rules, and indexing rules
- `SKILL.md` "Writing a research report" section added before "Conversational Integration"
- SKILL.md step instructs agent to write file with front matter AND run `tankyu insight from-report <file>`
- Both documents reference consistent field names matching `ReportFrontMatterSchema`

---

**Session Prompt:**

```
You are working on Tankyu (TypeScript, ESM). Task 2.3 — Dive Skill Update + Front Matter Spec.
Waves 0 and 1 are merged. Pull main before starting.

Context files to read first:
- docs/workspace/20260306-p1-insight-pipeline/shaping.md — parts A7, A8
- src/domain/types/report.ts — ReportFrontMatterSchema (authoritative field definitions)
- /Users/cmbays/Github/tankyu/skills/dive/SKILL.md — existing dive skill (IN MAIN REPO, not worktree)
- research/ — read 1-2 real reports to use as a realistic example in the spec

What to build:
1. NEW: docs/report-front-matter-spec.md — authoritative contract doc
   Include:
   - Schema table (all fields, types, required/optional, descriptions)
   - Complete example YAML block (use a real report as basis)
   - What the retroactive script (add-front-matter.ts) populates for each field
   - What the dive skill must produce (front matter fields + status: complete)
   - Indexing rules (topics must exist, idempotent re-runs, UUID writeback)
   - How to add front matter to legacy files (tsx scripts/add-front-matter.ts)

2. MODIFY: /Users/cmbays/Github/tankyu/skills/dive/SKILL.md
   Add new section "### Writing a research report" BEFORE "## Conversational Integration"
   The section must instruct the agent to:
   a. Write research/<topic>-<date>-<slug>-dive-notes.md with complete YAML front matter
      (type, title, date, topics[], subject, tags:[], key_points[], relates_to:[], status: complete)
   b. Run: tankyu insight from-report research/<filename>.md
   c. Note that the UUID will be written back to the file automatically

IMPORTANT: SKILL.md is at /Users/cmbays/Github/tankyu/skills/dive/SKILL.md (main repo).
Edit it there directly. The spec doc goes in this worktree's docs/ directory.

Verify:
- docs/report-front-matter-spec.md created and complete
- SKILL.md updated with the new section
- No npm run typecheck errors (docs only)
```

---

## Session Sizing

| Wave | Session | Complexity | New Files | Modified Files |
|------|---------|------------|-----------|----------------|
| W0   | 0.1 schema-foundation | Medium | 2 | 6 |
| W1   | 1.1 domain-core | High | 4 | 0 |
| W2   | 2.1 cli-insight | Low | 1 | 1 |
| W2   | 2.2 retro-script | Medium | 2 | 0 |
| W2   | 2.3 dive-skill-spec | Low | 1 | 1 (main repo) |

---

## Dependency DAG

```
0.1-schema-foundation
       │
       ▼
1.1-domain-core
       │
       ├──────────────────┬─────────────────┐
       ▼                  ▼                 ▼
2.1-cli-insight   2.2-retro-script  2.3-dive-skill-spec
```

---

## Critical Path

`W0:schema-foundation → W1:domain-core → W2:cli-insight` (longest sequential chain; W2 parallel sessions do not extend it further)

---

## Merge Strategy

**Wave 2 parallel sessions touch non-overlapping files:**

| Session | Files owned |
|---------|-------------|
| 2.1 cli-insight | `src/cli/commands/insight.ts` (new), `src/cli/program.ts` |
| 2.2 retro-script | `scripts/add-front-matter.ts` (new), `scripts/add-front-matter.test.ts` (new) |
| 2.3 dive-skill-spec | `docs/report-front-matter-spec.md` (new), `skills/dive/SKILL.md` (main repo) |

No file is touched by more than one W2 session. Merge in any order — no conflicts expected.

---

## Post-Implementation: Retroactive Batch

After all sessions merge, run the batch to index all 26 existing reports:

```bash
# Step 1: Create the three topics
tankyu topic create doodlestein-ecosystem --description "Doodlestein / Jeffrey Emanuel repo research"
tankyu topic create tankyu-v2 --description "Tankyu v2 architecture, graph backend, and schema decisions"
tankyu topic create agentic-research-workflow --description "Research workflow patterns, flywheel design, agent coordination"

# Step 2: Add front matter to all 26 reports (see topic assignment table in shaping.md)
# Run per-report with the appropriate topic(s) — multi-topic reports need manual front matter editing after initial run
tsx scripts/add-front-matter.ts research/ --topic doodlestein-ecosystem  # start with primary topic per file

# Step 3: Dry-run to verify front matter is valid
tankyu insight from-report research/ --dry-run

# Step 4: Batch index
tankyu insight from-report research/

# Step 5: Verify
# Check ~/.tankyu/insights/ has 26 files
# Check edges.json has tagged-with edges
```

The topic assignment table from `shaping.md` (Section "Topic Mapping: 26 Existing Reports") lists primary and secondary topics for all 26 reports. For reports with multiple topics, manually add secondary topics to the `topics:` array in front matter before running `from-report`.

---

## Notes

1. **`glob` package**: Check `package.json` before using `glob` in `ingestDirectory`. If not installed, use `fs.readdir(dir, { recursive: false })` + `.filter(f => f.endsWith('.md'))`. Node 20 supports recursive readdir but for flat `research/` dirs, non-recursive is sufficient.

2. **Zod v4 unknown field behavior**: Verify whether `InsightSchema` with Zod v4 strips or rejects unknown fields. Test `InsightSchema.parse({ ...validInsight, topicId: 'xxx' })` — if it succeeds (strips), that's correct behavior. The test in W0 should reflect actual behavior.

3. **`writeIdBack` idempotency**: The regex `/^---\n/` in `writeIdBack` replaces the first `---\n` occurrence. If the file already has `id:` as first line, re-running `from-report` goes through the `update` path (N17), not `create` (N16), so `writeIdBack` is never called on re-runs. No double-insert risk.

4. **`insight-writer.ts` callers**: After W0 removes `topicId` from `ResearchNoteInput`, check for any other callers of `writeResearchNote` or `writeSynthesis` in the codebase. Run a grep for `topicId:` in `src/` to catch stragglers.

5. **SKILL.md location**: `skills/dive/SKILL.md` is in the main repo root, not inside any worktree. Task 2.3 edits it directly. It does not get merged via PR — it's a live edit to the main repo. Coordinate with other work to avoid conflicts.
```
