# Agent guidelines

## Docs are the source of truth
- Read `docs/Tasks.md` before starting a task, and mark the step you are about to do as `[~]`.
- Mark it `[x]` when done, or `[-]` with a one-line reason if it turned out wrong or unnecessary. Never leave a step silently skipped.
- The spec is `docs/specs/PRD.md`. The per-channel column mapping, measured from real exports, is `docs/specs/DATA-MAPPING.md`. Do not guess a header name, a mask rule, or an empty-rate — read the file. If you find something that contradicts it, fix the doc in the same change.
- Design tokens live in `docs/design/cloudflare.design.md`.

## Data rules that must not be broken
- One row = one order item. `line_no` is mandatory on every ingest. An order with several items is several rows.
- Idempotency is keyed on `row_hash` over **immutable** columns only. Exclude `status`, `shipped_at`, and `delivered_at` from the hash so a status change becomes an `UPDATE`, not a duplicate row.
- Exclusion keywords come from the shop owner and live in the `exclusion_rules` table. Never hardcode them. Channel column mappings are a property of the marketplace, so those do live in code (`app/domains/ingest/templates/`).
- Run exclusion before identity matching. Excluded rows stay visible and reversible; they are flagged, not deleted.
- Never run `pg_trgm` over a masked string. Extract features first, then score.
- Money is `BIGINT` rupiah. Never float, never numeric.

## Operating principles
- Act as an engineering peer, not an agreeable assistant. Do not default to agreement or false praise. If an idea has flaws, risks, or simpler alternatives, challenge it directly with concrete reasons before proceeding.
- Before writing any code, apply the Karpathy guidelines: think first, surface assumptions explicitly, keep solutions simple, make surgical edits, and define verifiable criteria before touching files.
- Never run git commit or git push without explicit permission from the user.
- Commit messages must follow the format `type(scope): short message under 100 character`.
- Apply the Humanizer skill to all user-facing copy and documentation: write naturally, drop AI tells, fluff, and staged marketing language.
- Maximize developer experience (DX) and follow industry-standard documentation practices: keep setup instructions reproducible, configurations explicit, and error messages actionable.

## Your training data is stale

Libraries, frameworks, and platform APIs evolve faster than your training cutoff (January 2025). Assume every signature, convention, config schema, and CLI flag you remember may have changed.

**Before writing any code involving:**
- Go, Goravel, PostgreSQL, Redis
- Next.js, React, Bun, Tailwind CSS, shadcn/ui
- Any third-party SDK or cloud service API

**You must:**
1. Query Context7 for current documentation (`mcp__plugin_context7_context7__resolve-library-id` then `query-docs`)
2. Verify the API signature, config format, or convention from the returned docs
3. Never rely on memory when Context7 coverage exists

**Do not:**
- Guess function parameters, import paths, or config keys
- Assume "this probably hasn't changed"
- Skip Context7 because "it's a simple case"

If Context7 returns no results, then fall back to WebSearch or direct doc URLs. Memory is the last resort, not the first.

## Architecture and repository structure
- Maintain separate repositories or decoupled directories for the backend service and the frontend web application. Keep configuration, package names, and documentation agnostic of temporary project codenames.
- Frontend uses Bun as the JavaScript runtime and package manager (`bun install`, `bun run dev`, `bun run build`).
- Backend uses Go with Goravel, backed by PostgreSQL for persistence and Redis for queue dispatching and analytical caching.
- Invalidate related dashboard cache keys in Redis whenever new data is ingested or catalog mappings change.

## Code standards and best practices

### Go and Goravel
- Follow Effective Go idioms: early returns (guard clauses), zero-value usability, and concise package names without stutter.
- Use a domain-driven slice architecture (`app/domains/`) instead of rigid Service-Repository wrappers. Group logic by functional domain: `ingest`, `matching`, `catalog`, and `analytics`.
- Avoid empty wrapper interfaces around database operations. Query models directly using `facades.DB()` for basic reads and writes. Centralize analytical queries inside `app/domains/analytics/`.
- Keep parsing and data sanitization logic in pure Go functions within `app/domains/ingest/` without database or HTTP couplings. Write table-driven unit tests for all edge cases (missing headers, dirty numbers, unexpected date layouts).
- Manage resources explicitly: always pair resource acquisition with `defer` (such as `defer file.Close()` and `defer rows.Close()`).
- Prevent memory exhaustion on large files like TikTok Excel exports: stream rows using `io.Reader` and `excelize` stream iterators instead of loading full workbooks into memory.
- Bound concurrent processing using buffered channels as semaphores. Do not spawn unbounded goroutines.
- Keep Goravel controllers thin: validate input using form requests, store uploaded files via `facades.Storage()`, and dispatch heavy processing to `facades.Queue()`.
- Run file ingestion inside asynchronous queue jobs (`app/jobs`) with explicit retry and failure policies.
- Define database schemas using Goravel migrations (`database/migrations`) instead of ad-hoc manual SQL scripts.
- Pass `context.Context` through database operations, I/O calls, and long-running routines.
- Wrap errors using `fmt.Errorf` with `%w`. Inspect errors using `errors.Is` and `errors.As`. Never ignore returned errors.
- Use structured logging (`log/slog` or Goravel logger) with relevant key-value pairs instead of plain console prints.
- Avoid premature abstractions. Build single-purpose functions and structs first.

### Frontend (Next.js, React, and Bun)
- Use App Router with Server Components by default for layout structure, page rendering, and initial data fetching.
- Build UI components using Tailwind CSS and shadcn/ui, styled with the design tokens in `design/cloudflare.design.md` (Inter Variable typography, crisp 1px borders, subtle radius, structured content-first layouts).
- Limit Client Components (`'use client'`) to interactive leaves: file upload dropzones, filter selectors, and chart canvases.
- Pass only serializable props across the Server-to-Client component boundary.
- Store dashboard filter state (date ranges, active channels, selected SKU) in URL search parameters (`searchParams`) rather than isolated client state. This keeps views bookmarkable and server-renderable.
- Prevent request waterfalls: fetch independent dashboard widgets in parallel using `Promise.all` or stream them independently inside React Suspense boundaries.
- Follow WCAG AA accessibility standards and use semantic HTML elements.

### Database (PostgreSQL)
- Define explicit column types, constraints, and composite indexes for common query patterns (such as `channel, order_date`).
- Index every foreign key column (`customer_id`, `order_id`, `master_sku_id`) to prevent table scans on joins and deletes.
- Store financial values in integer types (`BIGINT`) representing base currency units. Never store currency in floating-point columns.
- Ensure import idempotency using database constraints (`ON CONFLICT (channel, row_hash) DO NOTHING` or `DO UPDATE`).
- Ingest large datasets using multi-row batch inserts within transactions, or PostgreSQL `COPY` for bulk imports.
- Keep database transactions small and scoped to essential writes. Configure connection pooling and statement timeouts on the database connection pool.

### General
- Prefer standard library utilities before adding new third-party dependencies.
- Write the minimum amount of code required to solve the task correctly.
