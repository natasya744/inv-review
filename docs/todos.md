# Invoice Review — build & optimization plan

Ordered, slice-by-slice plan. Each slice is a working vertical increment: build it,
verify it, update `docs/build-along.md` in the same commit. Do not skip ahead to
cloud providers before the deterministic core works.

## Guiding principles

1. **Deterministic first, cloud second.** Rules, storage, and flow must work with
   stubbed extraction before any Azure call is made.
2. **Respect the boundaries** (`docs/architecture.md`): provider SDK types never
   leave `backend/app/providers/`; rules stay pure in `invoices/validation.py`;
   routes/service/repository stay separate.
3. **No new dependencies** without approval; keep installs locked.
4. **Verify every slice**: ruff backend; type-check + lint + build frontend;
   manual browser walkthrough of the story slice adds.

---

## Phase 0 — Baseline hygiene

- [ ] Confirm locked installs reproduce: `uv sync --locked` and
      `pnpm install --frozen-lockfile`.
- [ ] Run `./scripts/dev.sh --check` (or create it if missing) and confirm health
      endpoint + starter screen load.
- [ ] Copy `.env.example` → `.env` for backend/frontend; confirm no secrets are
      committed.
- [ ] Remove stray root files (`example.py`) and add `.DS_Store` handling if needed.

## Phase 1 — Domain core (pure Python, no cloud)

- [ ] Define normalized financial-document models (invoice + receipt) in Pydantic:
      identity, VAT IDs, dates, PO, currency, totals, provenance tags.
- [ ] Add the GL catalog and selection validation in `backend/app/accounting/`.
- [ ] Implement pure invoice policy in `backend/app/invoices/validation.py`
      (blocking errors vs warnings per client brief, EUR 0.01 tolerance, date order,
      duplicate vendor/invoice keys).
- [ ] Implement receipt policy (required merchant/date/currency/total/VAT;
      subtotal+VAT reconciliation).
- [ ] Add EU VAT format/checksum validation via `python-stdnum` (already a planned
      dependency — confirm before adding).
- [ ] Verify by exercising rules against `samples/manifest.json` expectations with a
      small throwaway script or REPL demo (no committed test suite).

## Phase 2 — Persistence & orchestration skeleton

- [ ] SQLite schema + `backend/app/repository.py` (documents, reviews, statuses,
      deletion for re-demo of duplicates).
- [ ] Local file storage for uploads under a gitignored directory.
- [ ] `backend/app/service.py` orchestration skeleton: upload → extract → merge →
      validate → persist, with extraction behind an interface so it can be stubbed.
- [ ] Duplicate detection keyed on vendor + invoice number.
- [ ] Routes in `backend/app/routes.py`: upload, list, get, delete, approve/reject.
      Verify end-to-end with stubbed extraction via curl.

## Phase 3 — Extraction providers

- [ ] `providers/azure_document_intelligence.py`: prebuilt-invoice / prebuilt-receipt,
      normalized output only (SDK types stop here). Confidence captured.
- [ ] `providers/` document-review adapter: Azure OpenAI Responses API, Entra auth,
      strict structured output returning classification + independent fields.
- [ ] Deterministic merge: Document Intelligence primary; LLM fills missing fields
      only; conflicts surfaced, provenance recorded per field.
- [ ] Wire providers into service behind feature flags/settings so offline runs still
      work with the stub.
- [ ] Exercise the 13-document corpus manually through upload and record results.

## Phase 4 — Review policies & GL suggestion

- [ ] Apply invoice/receipt policies to merged data; store findings with severity.
- [ ] GL categorizer provider: normalized fields only → structured suggestion;
      reviewer can override; invalid selections rejected by `accounting/` validation.
- [ ] Approval gate: blocked while any error-level finding is open or no valid GL
      account selected.

## Phase 5 — Frontend review experience

- [ ] Env module `frontend/src/lib/env.ts`; typed API client.
- [ ] Flow screens: welcome → upload/preview (4 MB limit, PDF/PNG/JPEG) → processing
      status → review.
- [ ] Review screen: extracted fields with provenance badges, conflicts highlighted,
      findings (errors/warnings), VAT checks, editable GL selection, approve/reject.
- [ ] History list with local delete (enables duplicate re-demo).
- [ ] Correction-email draft: on-demand request, Copy + Close modal; never sends.

## Phase 6 — Polish & final verification

- [ ] Full manual walkthrough with the fictional corpus: happy paths (01–04), each
      failure mode (05–10), scan quality (11), two-page (12), receipt (13).
- [ ] Confirm every value's provenance is visible; LLM never overrides a conflicting
      primary value.
- [ ] Ruff clean; frontend tsc/ESLint/production build clean.
- [ ] Update `docs/build-along.md` with final outcome, commands, observations, and
      checkpoint ticks.
- [ ] Final secrets sweep: no `.env`, uploads, keys, or SQLite DB staged for commit.

---

## Suggested commit cadence

One commit per checklist item within a phase; one "slice complete" commit per phase
that includes the `docs/build-along.md` update.
