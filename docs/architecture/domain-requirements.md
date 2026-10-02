# Domain requirements (carried over from Sprint)

> Status: **digest, v1** (2026-10-02). What the Sprint product really needs, read from Sprint's docs, migrations and code at `main` (`edd49e3`, PR #106). It is the *product*, not the design: the rebuild keeps these rules and behaviours, and is free to enforce them differently. Paths below are in the Sprint repo unless they say otherwise. "Mig 00NN" means `packages/db/migrations/00NN_*.sql`. Migrations 0018–0020 don't exist (numbers were skipped).

## 0. The product in one paragraph

One person (multi-tenant later) plans life and work in **1-week sprints (Mon–Sun, ISO weeks, user's time zone)**. Everything is an **object** in one **context graph**: tasks, ideas, sprints, recurring series, improvements, pages, journal entries, links, tags, projects, files, insights, plus user-defined types. Objects are joined by **typed, time-valid edges with provenance**. Every change lands in an **append-only event log**. A **semantic index** and rule-based **insights** feed AI features (summary, planning, link summaries, later "Ask your OS"), all of which **suggest, never act** without an accept. The sprint closes itself at the boundary; rituals become review-and-confirm.

Fixed product rules from `docs/open-questions.md` → Resolved (R1–R46, 11–17) that shape the data:

| Rule | Source |
|---|---|
| One sprint for everything; Mon→Sun; ISO week-year (a W53 counts in its ISO year) | R1, R4, R24 |
| Board: To do → In progress → Done; done stays visible until close | R2, R37 |
| Priority P0–P5, size XS–XL (points 1, 2, 3, 5, 8); "heavy day" by points (> 8) | R5, R38, `work-items/grading.ts` |
| Everything in a sprint is a **task**; everything in Capture is an **idea**; conversion is in place | R11, R20, R21 |
| Unfinished tasks carry over as **pending**; pending is not in the sprint until accepted; acceptance time = time added | R12, R22 |
| Baseline = end of Monday (local); later additions are "added mid-sprint" | R13 |
| Rating 0–1, two decimals, **always manual, AI never suggests or judges** | R14, R23 |
| Overdue is shown, never auto-moved | R15 |
| Every week gets a sprint, even unused weeks; their reviews queue | R39 |
| Recurring occurrences appear at their exact time, count as *planned*; unfinished → *missed* unless series says carry over | R40–R42 |
| Search: no stemming (`simple`), accent-insensitive, prefix match on every word | R46 |
| Archive, not delete, for fields/types/relation types; custom-type objects never join sprints | 13, 14 |

## 1. Built-in object types, relation types and database rules

### 1.1 Object types (`app.object_types`, seeded mig 0003, `file` mig 0015)

| Key | Typed table / properties | Converts to | DB-enforced rules |
|---|---|---|---|
| `task`, `idea` | `work_items` (status, priority, size, planned_on, due_on, completed_at); props `{kept_at?, fields?}` | each other | Must have a `work_items` row by commit (deferred trigger, mig 0004); `status='done' ⇔ completed_at not null`; enums checked (0004) |
| `sprint` | `sprints` (iso_year, iso_week, starts_on, ends_on, status, baseline_at, rating, reflection, ai_summary(+meta), metrics, ai_plan(+meta), closed_at) | – | Row by commit (0004); `unique(workspace, iso_year, iso_week)`; starts_on is a Monday, ends_on = +6, ISO year/week match starts_on; `status='closed' ⇔ closed_at`; rating numeric(3,2) in [0,1]; status ∈ planned/active/review_pending/closed (0004); metrics/ai_* jsonb objects (0007, 0009, 0024) |
| `recurrence` | props: rrule, time_zone, starts_on, time_of_day, priority, size, carry_over, active, next_at | – | Zod only (`recurrence/series.ts`) |
| `improvement` | props `{status: open/kept/done/dropped}` | – | Zod only |
| `page` | props `{icon?, favourite?, position?, template?{tag?}, fields?}`; body = BlockNote doc | – | Zod; body ≤ ~1 MB (app) |
| `journal_entry` | props `{date}` | – | `date ~ YYYY-MM-DD` check; **unique (workspace, date)** (mig 0012) |
| `link` | props: url, original_url?, kind, status, saved_from, note, kind_set, fetch{state,attempts,fetched_at,error?}, description, image, site_name, favicon, channel, price, currency, shop, ai_summary{text,model,generated_at}, fields? | – | `url ~ ^https?://`, `saved_from ∈ capture/page/journal`, status belongs to kind (video: to_watch/done; article: to_read/done; product: want/bought/dropped; page: saved), wrapped in `coalesce(…,false)`; **unique (workspace, url) incl. archived** (mig 0016) |
| `tag` | title = name; props `{color?}` | – | Name 1–32 chars, no whitespace or `#` (check); **unique lower(title) among live tags** (mig 0006) |
| `project` | props `{status active/done, goal ≤280, ends_on?, fields?}` | – | Check on props (mig 0022) |
| `file` | title = name; props `{content_type, size, status pending/ready}` | – | Check: size number, content_type string, status in set (mig 0015) |
| `insight` | props from `insights/schema.ts` (pattern key, numbers, dismissed/useful, changed_at) | – | Zod only |
| `person` | – | – | Seeded, hidden, unused (decision 15) |

Common `objects` columns (mig 0003): `id uuid` (UUIDv7 from app), `workspace_id`, `type` FK → object_types, `title ≤ 1000`, `body jsonb`, `properties jsonb object`, `source ∈ user/system/ai`, `created_by` default current user, `created_at`, `updated_at` (trigger), `archived_at`. `body_text` = generated column (mig 0011). Objects can't move workspaces (`check_object_update`, 0003).

### 1.2 Relation types (`app.relation_types`, mig 0003, narrowed in 0022)

| Key | From → To | max per from | Edge properties (Zod) |
|---|---|---|---|
| `in_sprint` | task → sprint | ∞ (but live edges = current membership) | `added_at`, `origin` (planned / pulled_from_capture / carried_over / created_mid_sprint / recurring), `state` (pending/accepted), `outcome?` (completed / carried_over / removed / missed) |
| `tagged_with` | any → tag | ∞ | – |
| `instance_of` | task/idea → recurrence | 1 | `occurrence_at` (canonical ISO); **unique (series, occurrence_at) incl. ended** (mig 0008) |
| `subtask_of` | task/idea → task/idea | 1 | `position` (one level only: app rule) |
| `blocks` | task/idea → task/idea | ∞ | – |
| `mentions` | any → any | ∞ | `block_id` (first block that mentions it) |
| `child_of` | page → page | 1 | `position`; **no cycles** (mig 0013 recursive walk, archived pages included) |
| `extracted_from` | task/idea → page/journal_entry | 1 | `block_id` |
| `on_date` | journal_entry → sprint | 1 | `date` |
| `reflects_on` | sprint → improvement | ∞ | – |
| `addresses` | task/idea → improvement | ∞ | – |
| `part_of` | task/idea/page → project | **1** (0022) + **partial unique index** `relations_one_project_idx` | – |
| `related_to` | any → any (symmetric) | ∞ | `similarity?`, `duplicate?` |
| `evidence_for` | any → insight | ∞ | – |

Edge columns (mig 0003): `source` (user/system/ai), `status` (suggested/accepted/rejected), `confidence ∈ [0,1]`, `valid_from`, `valid_to` (`≥ valid_from`), `from_id ≠ to_id`, `created_by`. **Unique active edge per (from, type, to)** (`relations_one_active_edge_idx where valid_to is null`).

### 1.3 Integrity rules the database enforces (whoever writes)

| Rule | Where |
|---|---|
| Relation endpoints, type and workspace can't change; end the edge and make a new one | `check_relation` (0003, 0005, 0027) |
| Both ends in the edge's workspace | `check_relation` |
| From/to types allowed by the relation type; **skipped for edges inserted already ended** (history imports) | 0003; ended-insert skip 0005 |
| `max_active_per_from` and (custom) `max_active_per_to` counted when an edge becomes active, **under `pg_advisory_xact_lock` per (end, type)** | 0003; locks 0027 |
| A relation type must be built-in or the edge's own workspace; archived custom relation types refuse new active edges (except `app.importing='on'`) | 0027 |
| Type change only to `convertible_to` (task↔idea); refused while any active edge would no longer fit the new type | `check_object_update` (0003) |
| Every task/idea has a `work_items` row, every sprint a `sprints` row, by commit | deferred constraint trigger (0004) |
| Typed rows match their object's type and workspace and never move | `check_typed_row` (0004) |
| Page tree is a tree | `check_page_tree` (0013) |
| One occurrence per (series, moment), forever | unique index (0008) |
| One journal entry per day; one link per canonical URL; one live tag per name; one project per item | 0012, 0016, 0006, 0022 |
| Custom-type objects are in the type's workspace; nothing new joins an archived type (import excepted) | `check_object_type` (0026) |
| Custom field values fit live definitions of the same workspace/scope (deferred) | `check_object_fields` (0025/26/28) |

**Archive semantics.** Nothing user-visible is hard-deleted. Objects get `archived_at` (restorable; their edges stay). Edges are *ended* (`valid_to`), never deleted. Deleting a tag = end its `tagged_with` edges + archive it (name is free again). Dropping a task archives it and ends its live `in_sprint` with `outcome=removed`. Archiving a page can archive or re-parent its sub-pages; restore restores those archived with it. Archiving a custom type/field/relation type only sets `archived_at` on the definition; objects and values stay, readers filter by a join. `sprint_app` has **no DELETE grant** on objects, relations, events, ontology tables (only `object_chunks` can be deleted).

**Conversion idea ↔ task.** Same id, type flips, `object.type_changed` event. Rule of thumb (enforced by a test): every relation an idea can have, a task can have too, except `in_sprint`. Pull from Capture → idea becomes task + `in_sprint {origin: pulled_from_capture, state: accepted}`. Back to Capture → live `in_sprint` ends with `removed`, status reset to `todo`, task becomes idea. Custom fields share scope `work_item`, so values survive conversion.

## 2. The editable ontology (ADR-0005, as built, mig 0025–0028)

| Definition | Storage | Key | Limits | Validation |
|---|---|---|---|---|
| Custom field | `app.object_fields` (id, workspace, scope, key, label, data_type, settings, position, featured, archived_at) | slug `^[a-z][a-z0-9_]{0,39}$`, `_2` on clash; unique per (workspace, scope) incl. archived; never changes | **30 live per scope** (advisory lock + count) | settings per type: number `{decimals 0–4, unit ≤8}`; select/multi `{options ≤50: {id o_xxxxxx, label 1–60 unique ci, color ∈ 9 names, archived?}}`; others `{}` (`field_settings_ok`) |
| Field scopes | `work_item` (task+idea), `page`, `link`, `project`, or a `c_` custom type key of the same workspace | – | – | scope check + trigger (0028) |
| Custom object type | row in `app.object_types` with `workspace_id` (null = built-in), plural, icon (24 Lucide names), title_label, position, archived_at | `c_` + 10 random [a-z0-9], made by DB/app; global PK | **30 live types** | label/plural 1–40, unique ci among live types (label and plural cross-checked), never a built-in label (except hidden `person`); `convertible_to = {}` |
| Custom relation type | row in `app.relation_types` with `workspace_id`, `max_active_per_to`, position, archived_at | `r_` + 10 random | **30 live** | labels 1–40; exactly one from-type and one to-type: `{task,idea}`, `{page}`, `{link}`, `{project}` or a same-workspace custom type (live at creation); cardinality "one" or none per side; key, workspace and ends never change; many→one refused while any object has >1 live edge (waits on an exclusive per-type lock, links hold it shared) |

**Values** live on the object: `properties.fields = {key: value}`; absent = unset (no null/""/[]). Types: text (1–500, trimmed, non-blank), number (finite, |x| ≤ 1e15), date (`YYYY-MM-DD`, real date), select (option id), multi_select (1–20 distinct ids, option order), checkbox (`true` only), url (http(s), ≤ 2000, host with a dot). Archived options may stay on values already holding them, never chosen anew; inserts may bring archived values (imports). Writes **merge** (`jsonb_set(… || patch) - removed`), never replace. Changing a field's type or deleting an option rewrites affected objects in one transaction, with `caused_by` (from `app.caused_by` setting) on each object event; a deferred trigger refuses commit if any stored value no longer fits (`check_field_values`).

**Tenancy**: built-in rows readable by all, writable by none (RLS `using` never matches null workspace); custom rows member-only; column-level update grants exclude key/workspace/ends. **History**: `ontology.field_*`, `ontology.type_*`, `ontology.relation_*` events with no object. **Opt-in sites**: search, mentions, semantic index, related, backlinks, tag pages, Markdown export include live custom types; sprints, Capture, projects, duplicates refuse them. Collections (`core/collections/query.ts`): `?sort=`, `?f.<key>=`, `?q=`, `?archived=1`, `?r.`/`?ri.` relation filters; DB-side keyset paging, empty values last; partial GIN on `properties->'fields'` for `c_` types (0026).

## 3. The event log

**Shape** (`app.events`, mig 0003): `id bigint identity` (total order), `workspace_id`, `occurred_at = clock_timestamp()`, `actor_id`, `actor_kind ∈ user/system/ai/automation` (from transaction setting `app.actor_kind`, default user), `object_id?`, `relation_id?`, `kind`, `payload` (before/after per changed column; body changes only `{changed:true}`).

**Writers: only database triggers** (app role has SELECT only):

| Trigger | Kinds |
|---|---|
| `log_object_event` (0003, `caused_by` in 0025) | `object.created` (type, title, properties, has_body, source), `object.updated`, `object.type_changed`, `object.archived`, `object.restored`; no event if only `updated_at` changed |
| `log_relation_event` (0003) | `relation.created`, `relation.ended`, `relation.status_changed`, `relation.updated` (object_id = from_id) |
| `log_typed_row_event` (0004) | `work_item.created/updated`, `sprint.created/updated` (column diffs) |
| field/type/relation-type triggers (0025–0027) | `ontology.*` |
| `import_events()` (0005, SECURITY DEFINER) | replaces the import transaction's own events with the original log (owner only, empty workspace, every event must point at this workspace's rows), then appends `workspace.imported` |

**Actor kinds in practice:** sprint close and recurring materialization = `system`; saving an AI summary = `ai`; unrequested link fetches = `system`.

**Readers (as built):** the item panel's **History** (`items/history.ts`, `itemHistory`, joins relation type and target title; marks system steps "(automatic)"; diffs `fields` with labels); Capture **staleness** ("last touched" = newest event on the idea, 90 days); the **export** (full log) and import; Settings → Usage (log size). **Metrics do not read the log**: they come from `in_sprint` edge properties and are frozen in `sprints.metrics` at close. Insights read closed sprints, memberships and improvements; tag-suggestion feedback reads `tagged_with` edge statuses over 90 days. The docs promise undo and "what changed this week" from the log; not built.

## 4. Time and scheduling

### 4.1 Rules

- **Workspace time zone** (IANA, validated against `pg_timezone_names`, mig 0001; first value from a browser cookie). All boundaries computed with `Intl` (`core/time/zones.ts`): `sprintWindow(week, tz)` → `startsAt` Mon 00:00 local, `endsAt` next Mon 00:00, `baselineAt` Tue 00:00 local. `startOfLocalDay` handles a **skipped midnight** (DST at 00:00, e.g. Chile).
- **ISO week-year** for sprint keys (`2026-W40`); W53 handled (`time/calendar.ts`).
- **Core functions take `now` and the time zone as parameters**; no hidden clock.
- **Metrics** (`sprints/engine.ts` `sprintMetrics`): only accepted edges count. Planned = `added_at ≤ baseline_at` or origin `recurring`; Added = after baseline; Completed / Carried over / Missed / Removed by outcome; completion = completed / (planned + added − removed), weighted by size points; by priority. Snapshot saved at close and never recomputed.
- **Carry-over count `↻ n`** = number of the task's `in_sprint` edges with outcome `carried_over`.

### 4.2 `ensureSprintState(tx, workspace, now)` (lazy, idempotent; `packages/db/src/sprint-engine.ts`)

Runs on **every signed-in request** before pages read, and from the daily job.
1. Fast path without lock: plan (`planSprintCatchUp`), "any series with `next_at ≤ now`?", current sprint exists → return.
2. Else `pg_advisory_xact_lock('sprint-state:'+ws)`, **re-plan** (another request may have done it), act as `system`.
3. Create missing weeks (every week from the first open past sprint, or after the latest, up to now).
4. For each past open week, oldest first: materialize recurring occurrences due up to `endsAt − 1 ms`, then close: each live edge → `closeDecision` (archived → removed; accepted+done → completed; recurring without carry_over → missed; else carried_over + new **pending** edge into next week with `added_at = boundary`); unaccepted pending edges move forward again; edges get outcome and are ended; metrics snapshot; status `review_pending`.
5. Materialize occurrences due up to `now`; activate current sprint.
6. After commit, the request sends `sprint/sprint.closed` per closed week.

Future sprints are created early only when something is planned into them (per-week advisory lock in `ensureSprint`).

### 4.3 Recurring (RRULE subset, `core/recurrence/rule.ts`)

FREQ daily/weekly/monthly/yearly, INTERVAL, BYDAY (with ordinals ±1–5 for monthly/yearly, e.g. `-1FR`), BYMONTHDAY (−1 = last), BYMONTH, COUNT, UNTIL (date). Evaluated on **local dates in the series' zone**; 31st / 29 Feb skipped where absent; spring-forward gap → time moves forward by the jump; fall-back repeat → first instance. Series state: `next_at` (next not-yet-created occurrence). Occurrence = task, planned for its day, series' priority/size/tags, `instance_of {occurrence_at}` + `in_sprint {origin: recurring, state: accepted}`. Editing changes only future occurrences; a schedule change restarts from today; pause/resume skips what fell in the pause; end = archive series. Upcoming occurrences count in planning's load bar (`recurrence/upcoming.ts`). Quick-capture grammar `every …` (`capture/parse.ts`); form ↔ RRULE (`recurrence/form.ts`).

### 4.4 Background work (every trigger)

| Job | Trigger / frequency | Runs as | Side effects | Retry / concurrency |
|---|---|---|---|---|
| `GET /api/cron/daily` | Vercel cron `0 4 * * *` UTC (Hobby: any time that hour); `Bearer CRON_SECRET` | none | DB ping (keeps free Supabase awake: pauses after 7 idle days); sends `sprint/daily.tick`; **link fetch retries** (failed with < 3 attempts or never finished, last try > 1 h ago; ≤ 20/run, 4 parallel; via `app.links_to_refetch`, mig 0017); without Inngest, runs the semantic sweep inline | – |
| `daily-maintenance` | `sprint/daily.tick` | each workspace's earliest owner (`app.safety_net_workspaces`, mig 0021) | `now` fixed in its own step; per workspace `ensureSprintState` (safety net); `sprint/sprint.closed` per closed week; **semantic sweep** ≤ 500 objects/workspace/day (missing, stale hash, other model; also backfill) + duplicate/tag checks for fresh items | 2 retries; one step per workspace; failures → Sentry, others continue |
| `sprint-summary` | `sprint/sprint.closed` (from request or daily job) | user who closed | AI summary into `sprints.ai_summary` (+meta), keeps an existing one | 3 retries; concurrency 1 per (workspace, week) |
| `sprint-insights` | `sprint/sprint.closed` | same | recompute `insight` objects + `evidence_for`, archive vanished patterns | 3 retries; 1 per workspace; advisory lock |
| `sprint-plan` | `sprint/sprint.closed` → plans next week | same | AI plan into `sprints.ai_plan`; keeps existing | **Defined but not in the exported `functions` list, so never registered** (see §11) |
| `embed-object` | `sprint/object.changed` (sent after commit by saves of pages, journal, items, ideas, links, quick capture, link fetch) | user who saved | replace chunks if hash changed; duplicate check and tag suggestions for fresh items in the same transaction | debounce 60 s per object; 3 retries |
| `test-ping` | `sprint/test.ping` (Diagnostics) | – | none | – |
| Link metadata fetch | `after()` right after a link save (not a job) | user | `recordLinkFetch` under `select … for update` | cron retries |
| `ensureTemplates` | first visit (request) | user | seeds 5 page templates once per workspace (advisory lock) | – |

Planned, not built: user automation rules ("when X then Y"), reminders, stale-Capture is a per-browser monthly cookie nudge, not a job.

## 5. Rich text, mentions, pages, journal

- **Format:** `objects.body` = one **BlockNote JSON document** (array of blocks with stable string `id`, `type`, `props`, `content` inline array or `tableContent`, `children`). Packages: `@blocknote/core`, `@blocknote/react`, `@blocknote/shadcn` **0.55, MPL-2.0 only; no `xl-*` (GPL) packages**. Used by pages, templates, journal entries, task/idea descriptions, and the sprint's reflection (stored as the sprint body since mig 0014; `sprints.reflection` keeps plain text with mentions written out).
- **Blocks:** paragraph, heading 1–3, quote, divider, bulleted/numbered/to-do/toggle lists, code, table, image, file + custom: `callout {emoji}`, `subPage {pageId}`, `youtube {videoId, linkId}`, `bookmark {linkId, url}`, `taskRef {taskId}` (no text; the task is the source of truth). Inline custom: `mention {id}`, `dateMention {date}`.
- **Body cap:** ~1 MB JSON (`PAGE_BODY_MAX_BYTES`), saved whole on each autosave (1 s debounce; title 0.6 s; one save at a time, in order).
- **Plain text:** `body_text` generated column (`app.blocks_text`, mig 0011): inline text joined per block, blocks/table rows/children on new lines; mirrored by `documentText` in core, a test compares them. `taskRef` contributes no text.
- **Mentions sync** (`core/mentions/mentions.ts` `extractMentions`, `planMentionSync`; db `syncMentions`): on every save, **in the same transaction as the body**, mention/bookmark/youtube `linkId` refs → `mentions` edges: add new, end removed, update moved `block_id`. One live edge per (doc, target). No edge for self, missing or other-workspace targets. Date mentions resolve to that day's journal entry (created on demand; future days or days before the first sprint make no edge).
- **Picker grammar** (`mentions/query.ts`): titles; days (`today`, `last fri`, `12 oct`, `2026-10-12`, both sides of today); sprints (`W40`, `2026-W40`); `new task …` / `new idea …` creates inline; `[[` alias.
- **Extract to-do** (`extractTodo`): `/task` or `/idea` turns a line into `taskRef` with `extracted_from {block_id}`; idempotent per (doc, block) under advisory lock.
- **Pages tree:** parent = one active `child_of` with `position` (fractional between neighbours); top-level order in `properties.position`; moves serialized per workspace (advisory lock); cycle refused by app (`canMoveUnder`) and DB (0013). Templates = pages with `template` prop, excluded from tree/search; copying a template **re-ids every block**.
- **Journal:** one entry per day (unique, mig 0012 + per-day advisory lock), titled "Wednesday 30 September 2026", linked `on_date` to that week's sprint; days allowed from the Monday of the first sprint up to today. Review shows the week's entries; AI summary reads ≤ 1,200 chars/day, ≤ 6,000 total.

## 6. Search

| Part | As built |
|---|---|
| Full-text (mig 0010) | No column: GIN **expression index** `objects_search_idx` on `app.search_document(title, body)` = `setweight(to_tsvector('simple', unaccent(title)),'A') ‖ setweight(…body_text…, 'B')`. `app.search_normalize` pins the `unaccent` dictionary so the function is IMMUTABLE. Queries must repeat the expression. Prefix match on every word (`prefixQuery`), ≥ 2 chars, title above content, archived excluded (⌘K), snippets with highlights (`snippetParts`). Links have no body: a separate branch searches title, description, note, domain |
| Semantic index (ADR-0004, mig 0023) | `app.object_chunks (object_id, chunk 0–63, workspace_id, content_hash, model, dimensions=512, embedding halfvec(512), embedded_at)`, PK (object, chunk); trigger keeps chunk in object's workspace; RLS; **no vector index** (exact scan by `workspace_id` btree; HNSW + `hnsw.iterative_scan` planned past ~50k chunks/workspace). Model **Voyage `voyage-4-lite`, 512 dims**, `input_type` document/query. Text: title + one string per top-level block (links: title, description, note, AI summary; sprints: reflection); chunks ≤ 400 tokens on block boundaries, ≤ 32 chunks/object, title prefix ≤ 200 chars. Embedded types: task, idea, page, journal_entry, link, improvement, insight, sprint, live custom types; not tags, files, series, templates. Derived: not in the event log or export. Writes are delete+insert under a per-object advisory lock |
| Hybrid | `searchEverything` = full-text ‖ meaning, merged by **reciprocal rank fusion k = 60**, exact titles first; meaning-only hits need cosine ≥ 0.3 and show the closest chunk as snippet; query embedding has an **800 ms budget**, no retries, LRU cache 200 queries/server; falls back to full-text alone |
| Related / duplicates | `SIMILARITY = {related 0.6, duplicate 0.85}`, ≤ 5 related, ≤ 3 duplicates among ideas/tasks captured in last 24 h; excludes pairs with any relation (a rejected `related_to` = "not related", remembered forever) |

## 7. AI features

Rules for all: suggest, never act; output marked (`source = ai`, AI badge); never overwrites user text; never rates or judges (R14). AI is off until keys exist (`ANTHROPIC_API_KEY`, `VOYAGE_API_KEY`). Every call goes through `complete()` (`apps/web/src/server/ai/client.ts`): explicit `effort`, system prompt cached (`cache_control: ephemeral`), server-side fallbacks on refusal, optional JSON-schema structured output, and a **usage ledger row** (`app.ai_usage`: feature, model actually used, input/output/cache tokens, `cost_usd` from `AI_PRICES`), written even if the reply is unusable; ledger failure never throws.

| Feature | Status | Model / effort / max tokens | Input | Output → graph |
|---|---|---|---|---|
| Weekly sprint summary | Built (R1.7) | the large Claude model, medium, 16k | metrics snapshot, every task of the week (priority, size, origin, outcome, weekdays, earlier carry-overs, tags), ≤ 4 earlier weeks' completion, open improvements, week's journal (capped); **never rating or reflection** | `sprints.ai_summary` + `ai_summary_meta {model, generatedAt, editedAt}`, saved as actor `ai`; 3–5 sentences, facts only, ≤ 1 question; user edits keep the AI badge |
| Link summary | Built (R4.4b), on demand (`s`) | the large Claude model, low, 8k | page fetched by the guarded fetcher, main text (`<article>` > `<main>`), trimmed to ~6,000 tokens; refused < 200 chars; page wrapped in `<page>`, `</page>` defused, marked untrusted | `link.properties.ai_summary {text, model, generated_at}` |
| Monday plan | Built (R4.6), "Suggest again" works; auto-draft job unregistered | the large Claude model, medium, 16k, JSON schema | capacity (avg completed points of last 4 closed sprints), pending carry-overs, rule-based candidates, ideas near open improvements, insights, last reflection, last 30 decided rows | `sprints.ai_plan {rows: {id, action, itemId, relationId, title, size, day, reason ≤200, decision pending/accepted/skipped/stale}, feedback}` + meta; ≤ 12 rows; `validatePlan` drops unknown ids and moves the manual actions wouldn't allow, caps load; Accept runs the manual action |
| Tag suggestions | Built (R4.4a), **no LLM** | – | nearest neighbours (≥ 0.5) tags, ≥ 2 neighbours, ≥ 0.25 share, ≤ 3; ×2 / ×0.25 by 90-day accept/reject history | `tagged_with` with `source=ai, status=suggested, confidence`; rejected stays (never re-offered); only accepted edges count as tags anywhere |
| Link kind suggestion | Built, no LLM | – | ≥ 3 nearest links disagree ≥ 60%, not when user set the kind | not stored |
| Duplicate hints, related | Built (R4.3), no LLM | – | §6 | suggested `related_to {similarity, duplicate:true}` |
| Insights | Built (R4.5), **rules, no LLM** | – | closed sprints (≥ 4), ≥ 3 items each: carry-overs by size/priority/tag, mid-sprint additions vs rating (≥ 3 additions), load vs capacity (6 weeks), weekdays work gets done (≥ 8 tasks, ≥ 60%), recent improvements (6) | `insight` objects (stable `pattern` key, template sentence, numbers) + `evidence_for`; archived when gone; re-shown after dismiss only on a change ≥ 0.1 |
| Ask your OS | **Specced (R4.7), not built** | the large Claude model, low; tool runner with caps | read-only tools as the user under RLS: `search` (hybrid), `get_object`, `neighbours` (1–2 hops), `sprints`, `journal`, `insights`; web content as data | streamed answer with object chips; not stored; "Save as page"; budget: notice at 80%, pause at 100% |
| Settings → AI, budget | Specced (R4.8) | – | ledger by month in workspace tz | per-feature on/off, model, monthly budget (recommended $10) |

**Costs (one heavy user, `build-plan.md` → R4 → Costs):** ≈ $2.50/month without Ask, $7–10 with Ask most days; embeddings $0 (Voyage free 200M tokens, then $0.02/M). Prices in `core/semantic/cost.ts`: the large Claude model $4/$20 per M (cache read $0.20, write $5). Open questions 6–10 (budget, cheaper model, chat history, summaries policy, Voyage card) are still open. Bring-your-own-key is wanted for the desktop app.

## 8. Files and links

- **Files** (mig 0015, `server/storage.ts`, `core/files/files.ts`): `file` object (name ≤ 200, content_type, size ≤ **25 MB**, status pending→ready). Bytes in a private bucket at `<workspace>/<file id>`, reachable only with the server secret. Flow: server creates pending row + one-time signed upload URL → browser uploads direct (Vercel request limit 4.5 MB) → server checks Storage holds exactly the announced size → ready. Blocks store `/files/<id>`, never a storage URL; the route checks RLS then redirects to a 1-hour signed URL. Images inline; everything else (SVG included) as download. No cleanup of orphaned files yet. Free plan 1 GB shown in Usage.
- **Links** (`core/links/*`, mig 0016/0017): **canonical URL** (`canonicalUrl`, `url.ts`): lowercase scheme/host, no userinfo, default port, fragment, trailing slash; tracking params dropped (`utm_*`, fbclid, gclid, `mc_*`, `si`, `ref`, `feature`, `share`…), others sorted; every YouTube form → `https://www.youtube.com/watch?v=ID`; real host with a dot, no bare IP; `www.x` → https. Pasted form kept as `original_url`. Kind from URL (`detectKind`), refined by fetch only while user hasn't set it (`kind_set`) and status is still first. Saving again finds/restores the existing link (per-URL advisory lock). "From a page" = only ever saved by paste; direct save clears it for good.
- **Guarded fetcher** (`apps/web/src/server/links/fetch-page.ts` + `address.ts`): http(s) only; **every resolved address must be public, checked inside the socket's own DNS lookup** (defeats DNS rebinding) and again for each redirect (≤ 3); literal IPs checked separately (Node skips lookup for them); blocked ranges: 0/8, 10/8, 100.64/10, 127/8, 169.254/16 (cloud metadata), 172.16/12, 192.0.0/24, 192.0.2/24, 192.88.99/24, 192.168/16, 198.18/15, docs ranges, multicast, 240/4 (+ IPv6 equivalents); **5 s total including body**, **1 MB decompressed cap** (stops decompression bombs), gzip/deflate/br, HTML or JSON only, 2xx only, charset from header or `<meta>`. Never throws.
- **Metadata parse** (`core/links/metadata/*`, no deps): first source wins — title og → twitter → JSON-LD → `<title>`; description og → twitter → meta; image og:image(secure) → twitter → JSON-LD; favicon icon → apple-touch → `/favicon.ico`; kind from og:type then JSON-LD @type; price from JSON-LD offers / AggregateOffer.lowPrice / priceSpecification then `product:price:amount`; "1.299,90" and "1,299.90" → `1299.90`; YouTube via oEmbed. RFC 3986 resolver; only http(s) image/icon URLs; title ≤ 300, description ≤ 1000. Fetch state `{state done/failed, attempts, fetched_at, error?}`; a fetch only replaces the title while it's still the address-derived one.
- **Library views**: Watch later (video, to_watch), Reading list (article, to_read), To buy (product, want; price totals per currency, exact decimals, never converted). "Buy it" → task "Buy X" mentioning the link (per-link advisory lock; one open buy item).

## 9. Auth, tenancy, workspaces, RLS

- **Identity:** Supabase Auth magic link (Google later); server verifies the token (`getClaims`), then the DB never trusts the browser. Cross-device link needs the custom `token_hash` email template. Optional `ALLOWED_EMAILS` allowlist (`@domain` entries), enforced in the request path too, not only on the form. Built-in email sender: a few emails per hour.
- **First sign-in:** `ensureAccount` creates `app.users` + workspace "Personal" + owner membership atomically (`app.create_workspace`, SECURITY DEFINER), idempotent under a **per-user advisory lock**. Workspace tz from cookie; invalid → UTC.
- **Tenancy model** (mig 0001, `docs/architecture/data-access.md`): `workspace_id` on every row; `workspace_members (role owner/member)`; app connects as `sprint_app` (no superuser, no BYPASSRLS, owns nothing); every request in `withUser(userId)` = transaction with `set_config('app.user_id', …, true)` (transaction-local, safe on pooled connections); policies use `app.is_workspace_member(workspace_id)` (SECURITY DEFINER helper to avoid policy recursion); no user → nothing visible (fail closed). Column-level grants; no deletes. All SECURITY DEFINER functions pin `search_path = ''`.
- **Cross-tenant jobs:** only via SECURITY DEFINER functions that **refuse when `app.user_id` is set** and return minimal columns (`safety_net_workspaces`: id + earliest owner; `links_to_refetch`: link id, url, owner); the job then acts as the owner under RLS. Jobs triggered by a user act as that user.
- **Every new table**: `workspace_id`, RLS, explicit grants, and a test that another workspace can't read or write it (`security-invariants.test.ts`, `tenancy.test.ts`).

## 10. Volumes and performance targets

| Item | Number | Source |
|---|---|---|
| Objects per heavy user | 10k–100k over years | context-graph.md → Guardrails |
| Backfill estimate | ~10,000 objects × ~400 tokens ≈ 4M tokens | build-plan R4 Costs |
| Chunks | ~15,000 ≈ 17 MB (`halfvec(512)` ≈ 1.1 KB each); add HNSW past ~50k/workspace | ADR-0004 |
| Free-tier caps that shaped design | DB 500 MB (warn 80%, critical 95%), Storage 1 GB, Inngest ~100k runs/month, Vercel cron daily, Supabase pause after 7 idle days | ADR-0002, `usage/storage.ts` |
| Custom ontology | 30 fields per scope, 30 types, 30 relation types, 50 options, 20 multi-select values | mig 0025–0027 |
| Bodies / files / fetch | body ~1 MB, file 25 MB, fetch 1 MB / 5 s / 3 redirects | core, fetcher |
| Semantic sweep | 500 objects/workspace/day, Voyage batches ≤ 128 texts / ~100k tokens, 429/5xx retry (retry-after capped 60 s) | `jobs/embed.ts`, `ai/voyage.ts` |
| Latency targets | capture "instant" / save a link ≈ 2 s; find anything "in seconds"; full-text "under a second"; hybrid "under a second" (800 ms embed budget); new item searchable by meaning ~1 min after save; Ask < 30 s | vision, specs, build-plan |
| FB-1 speed work | Home add took ~5 s → optimistic add shows in ~4 ms; queries per add 82–101 → 48; pool `max: 3` let layout/page/cards run in parallel: Home 2.56 s → 1.48 s (30 ms DB RTT); remaining cost: `prepare: false` (transaction pooler) = 2 RTT per query, ~110 ms Warsaw↔us-east RTT (Frankfurt move shelved for R5) | build-plan → Feedback fixes, runbooks/move-to-frankfurt.md |
| Monthly cost | infra $0; AI ≈ $2.50 (no Ask) to $7–10 | build-plan R4 |

## 11. Hard-won lessons (don't relearn)

| # | Lesson | Where it's recorded |
|---|---|---|
| 1 | **Counting under a trigger races.** `max_active_per_from` was counted without a lock; two concurrent `part_of` writes could both pass. Fixed by a partial unique index (0022), then generalized: advisory lock per (end, type) before counting (0027). Same pattern for 30-item limits | mig 0022, 0027 |
| 2 | **Lazy close needs a lock *and* a re-read.** Fast path reads without lock; on work, take the workspace lock and re-plan, so a concurrent request finds nothing to do. Every boundary-ish operation is idempotent (journal per day, link per URL, template seeding, extract-to-do, buy, insights, AI plan, sprint per week, first sign-in) | `sprint-engine.ts`, list of advisory locks in `packages/db/src` |
| 3 | **Read-modify-write of jsonb loses edits.** `keepIdea` rewrote all `properties` and could undo a field edit; `changeType` re-parsed without a row lock (BUG-2). Rule: merge writes (`properties - removed ‖ patch`), `select … for update` before re-parse; link fetch writes under `for update` | build-plan FB fixes, `link-metadata.ts` |
| 4 | **Derived-index writers race too.** The sweep and `embed-object` raced on one object's chunks (BUG-1) → per-object advisory lock | build-plan BUG-1, `semantic.ts` |
| 5 | **Unique races in creation** (tags): insert inside a savepoint, on unique violation re-read | `tags.ts` |
| 6 | **SSRF / DNS rebinding:** check addresses inside the connection's DNS lookup, recheck per redirect, check literal IPs yourself, cap time including the body, cap decompressed size | `fetch-page.ts`, `address.ts` |
| 7 | **DST:** midnight can be skipped (start of day = first instant whose local date matches); RRULE times in a spring gap move forward by the jump, repeated times take the first; 31st / 29 Feb skipped (RFC 5545); week boundaries always from local dates | `time/zones.ts`, `recurrence/rule.ts` |
| 8 | **Uniqueness on timestamps needs a canonical form:** `occurrence_at` always `toISOString()` | mig 0008 |
| 9 | **Pending isn't membership:** pending carry-overs never count; acceptance time is "added". Recurring always "planned". Snapshot metrics at close so later edits don't rewrite history | sprints.md |
| 10 | **Job replays must be deterministic:** pick `now` in its own step; one step per tenant so one failure doesn't block others; duplicate events must not overwrite user edits (keep existing summary/plan) | `jobs/functions.ts` |
| 11 | **A defined job isn't a running job:** `sprintPlan` is defined but missing from the exported `functions` array, so Inngest never registers it; the auto-draft after a close never runs (only "Suggest again" does). Register jobs from one list and test it | `apps/web/src/server/jobs/functions.ts` (found in this digest) |
| 12 | **Vercel Deployment Protection blocked Inngest** from syncing production; auto-sync never fired → own GitHub workflow `PUT /api/inngest`. Inngest's run lookup is cached ~15 s | jobs-and-monitoring.md |
| 13 | **Thinking models:** the large Claude model always thinks and thinking counts toward `max_tokens` (a 16-token limit returned no text); set effort explicitly; org-level API keys are rejected (need workspace-scoped); log usage before checking the reply | jobs-and-monitoring.md, `ai/client.ts` |
| 14 | **Prompt injection from fetched pages:** wrap untrusted text in a delimited block, defuse the closing tag, say so in the system prompt, test it; Ask's tools are read-only | smarter.md R4.4b |
| 15 | **`unaccent` is only STABLE:** pin the dictionary to make an IMMUTABLE wrapper for an index. Index expressions must be repeated verbatim in queries | mig 0010 |
| 16 | **pgvector:** `halfvec` needs ≥ 0.7 (Ubuntu 24.04 ships 0.6); `extensions` schema isn't on `search_path` outside Supabase → `operator(extensions.<=>)`; ANN index + RLS returns too few rows without iterative scan | ADR-0004, smarter.md |
| 17 | **Transaction pooler:** no prepared statements (`prepare: false`) → 2 round trips per query; serverless pool small (3) | `db/src/client.ts`, `web/server/db.ts` |
| 18 | **Imports need deferred checks** (any row order), an `app.importing` escape for archived definitions, id remapping inside bodies/properties/event payloads (incl. `/files/<id>`), and key remapping for global custom keys | mig 0004, 0005, 0026, ADR-0005 |
| 19 | **Block ids are references:** mentions, extracted tasks, `#block-<id>` scrolls point at them; copying a template must re-id every block | pages.md R2.7 |
| 20 | **Serving user files:** never render uploaded HTML/SVG inline; store an app path, not an expiring storage URL; verify uploaded size before "ready" | pages.md R2.8 |
| 21 | **Quick-capture parsing:** first of each field wins, repeats stay in the title; ambiguous short words (`fri`, `sun`) count only in the trailing run or after `on/due/by`; a bare number isn't a time; `every` without a schedule stays a word | capture.md |
| 22 | **UI speed:** optimistic updates and one request per change; double render from `revalidatePath` + `router.refresh()`; prefetch storms from links | build-plan FB-1 |
| 23 | **E2E flakiness:** expose `aria-busy` while saving and wait on it | build-plan FLAKE-1 |
| 24 | **Free Supabase pauses** after 7 idle days → daily ping; no point-in-time recovery on free → export + `pg_dump` before destructive migrations; migrations forward-only, each PR documents its manual reversal | data-access.md |
| 25 | **Event noise:** skip events when only `updated_at` changed; log body change as `{changed:true}`, not content | mig 0003 |

## 12. What to carry into sprintOS

### 12.1 Docs to copy as requirements (rename "As built" to "Reference behaviour")

| Sprint doc | Why |
|---|---|
| `docs/vision.md` | Goals, principles, success criteria |
| `docs/open-questions.md` → Resolved table (R1–R46, 11–17) | The product decisions; the most compact spec there is |
| `docs/specs/sprints.md`, `recurring.md`, `capture.md`, `home.md`, `task-detail.md` | The sprint loop, metrics definitions, close semantics |
| `docs/specs/pages.md`, `library.md`, `projects.md` | Pages, journal, mentions, links, projects |
| `docs/specs/objects.md` + `docs/decisions/0005-editable-ontology.md` | Editable ontology: limits, keys, value rules, archive semantics |
| `docs/specs/smarter.md` + build-plan R4 "Costs" and "Decisions" | AI features, thresholds, cost envelope |
| `docs/architecture/context-graph.md` | The five layers and guardrails (drop the Postgres-specific parts) |
| `docs/ux/wireframes-r1.md`, `docs/decisions/0003-brand-identity.md` | UI input for the desktop app |
| This file | The enforcement rules and lessons |

Not as requirements: ADR-0001/0002/0004 and `data-access.md`, `jobs-and-monitoring.md` (design of the old stack; keep as evidence only).

### 12.2 Code worth porting (pure TypeScript; `@sprint/core` depends only on `zod` and `uuid`)

`packages/core/src` is ~10,600 LOC of source and ~8,300 LOC of tests (≈ 520 test cases, counting each `it.each` once). Most of it ports as is to any TS runtime (desktop, web, phone, server).

| Module (`packages/core/src/…`) | What | Src LOC | Tests (≈) | Port? |
|---|---|---|---|---|
| `time/calendar.ts`, `zones.ts`, `timezone.ts` | ISO weeks, W53, `sprintWindow`, DST-safe start of day, tz normalize | 234 | 18 | **Yes, first** |
| `sprints/` (engine, review, summary, year, board, planning, membership) | catch-up plan, close decision, metrics, review check, year view, board order, rule-based suggestions | 914 | 58 | **Yes** |
| `recurrence/` (rule, series, form, upcoming) | RRULE subset, series state, form ↔ RRULE, upcoming load | 795 | 30 | **Yes** |
| `capture/` (parse, view, repeat, nudge) | quick-capture grammar, Capture grouping/filters, staleness | 790 | 23 | **Yes** |
| `ontology/` (object-types, relation-types, custom-key, custom-types, custom-relations) | built-in ontology as Zod, custom-type rules | 704 | 30 | **Yes** (as the single ontology source) |
| `fields/` (definitions, values) | field settings, value cleaning, type change planning, filters, history lines | 888 | 27 | **Yes** |
| `collections/query.ts` | list query parsing (sort, filters, relation filters) | 198 | 8 | Yes |
| `links/` (`url.ts` 156, `metadata/*` ~600, `links.ts`, `library.ts`, `fetch.ts`, `buy.ts`, `summary/*`) | canonical URL, kind detection, metadata parsers, fetch merge, price totals, link-summary prompt + page text extraction | 1,477 | 56 | **Yes** |
| `mentions/` | extract mentions from BlockNote, sync plan, picker grammar | 471 | 18 | Yes (if BlockNote stays) |
| `pages/` (page, tree, templates) | body validation, `documentText`, tree moves, default templates | 460 | 25 | Yes |
| `semantic/` (embedding, chunks, hybrid, cost) | what to embed, chunking, content hash, RRF, cost table | 442 | 43 | Yes |
| `related/`, `suggestions/vote.ts` | thresholds, tag/kind vote | 248 | 16 | Yes |
| `insights/detect.ts` + schema | pattern detectors | 393 | 20 | Yes |
| `planning/ai-plan.ts`, `sprints/summary.ts` prompts | prompts, output schema, `validatePlan` | 401 (+181) | 14 | Yes |
| `search/search.ts` | prefix query, snippets, result targets | 245 | 19 | Partly (depends on new FTS engine) |
| `items/history.ts` | event log → human history lines | 253 | 7 | Yes |
| `export/` (format, markdown, library-markdown) | JSON export schema, BlockNote → Markdown with front matter, Obsidian links | 905 | 23 | Markdown converter yes (on-disk format); export format no (no export for now) |
| `work-items/grading.ts`, `tags/`, `projects/`, `journal/`, `home/today.ts`, `files/` | small domain rules | ~500 | ~50 | Yes |

Outside core:

| File (Sprint repo) | LOC | Port? |
|---|---|---|
| `apps/web/src/server/links/fetch-page.ts` + `address.ts` (+ tests, 8 cases) | 370 | **Yes** for any server/hub or desktop fetcher (Node `http`/`dns`; rewrite for another runtime but keep every guard) |
| `apps/web/src/server/ai/voyage.ts` | 198 | Yes (batching, 429/5xx backoff) |
| `apps/web/src/server/ai/client.ts` `complete()` | ~130 | Pattern yes: ledger-first, explicit effort, cached system prompt |
| `packages/db/migrations/*.sql` | ~2,900 | **As a spec, not code**: the trigger rules in §1.3 and §2 are the integrity contract the new store must enforce (or prove another way). `app.blocks_text` and `app.field_value_ok` define behaviour parity |
| `packages/db/test/*.test.ts` (~390 cases) | – | Mine for scenarios: concurrency, tenancy, import round-trips, engine catch-up |
