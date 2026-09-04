# Consumption Central for Microsoft Copilot

A Power BI (PBIP) report template covering Copilot credit consumption and cost across four
products: Cowork/Work IQ (Viva Insights), Copilot Studio, GitHub Copilot, and Azure AI Foundry.
There is no application code to build/run — this repo's "product" is a set of `.pbit` template
files plus the Power Query / TMDL / DAX source that produces them.

## Repository layout

- `1. Local CSV/` — CSV-based template + `sample-data/` (synthetic) + `pull_azure_ai.py`.
- `2. Fabric/` — Lakehouse/SQL-based template, ingestion `notebooks/` (Fabric notebooks, one per
  source), Power Automate `flows/`, and `docs/DATA-DICTIONARY.md` (the table contract the
  notebooks write to and the model's partitions read from).
- `3. Viva Direct/` — a Viva Insights connector-only variant (no files), with an unverified
  connector assumption documented in `TEST-PROCEDURE.md`.
- `docs/` — reference docs (`DATA-SOURCES.md`, `MEASURES.md`, `INTERPRETING.md`, `ORG-DATA.md`,
  `COMMERCIAL-TERMS.md`, `BUILD.md`, `TESTING.md`, `VIVA-CONNECTOR.md`) plus `docs/scripts/`
  validators.
- `archive/` — dated snapshots of previously-shipped `.pbit` files; do not edit, just history.

All three templates share **one model and report** (27 tables, 284 measures, 14–15 pages) — only
the data-source partitions differ between the CSV and Fabric variants. Treat measure/report logic
changes as needing to be validated (or ported) across whichever variants are affected.

## Build / validate commands

There is no compiled build. The `.pbit` files are exported manually from a working PBIP project in
Power BI Desktop — see `docs/BUILD.md` for the exact steps (this cannot be scripted, Desktop has
no CLI template export).

**Two rules for editing the PBIP model/report (TMDL/JSON):**
- Edit only with Power BI Desktop **closed** — Desktop holds an in-memory copy and overwrites
  on-disk changes when it saves.
- **Save the model in Desktop before closing it** — external edits only persist to TMDL once
  Desktop writes them; force-closing discards them.

Validators live in `docs/scripts/` and each catch a specific class of failure in ~1 second, well
before a ~2.5-minute failed Desktop load. Run the relevant one(s) after touching related files:

| Script | Run when you touch... | Catches |
|---|---|---|
| `check_m_syntax.py` | Power Query (M) | Unbalanced brackets, `let`/`in` mismatch, bare `try`, unquoted identifiers like `[@upn]` |
| `check_tmdl_indent.py` | TMDL model files | Indentation damage that stops the whole model loading; also fails on any UTF-8 BOM |
| `validate_model.py` | measures/model | Duplicate measures, DAX reserved words used as VAR names, dangling references |
| `validate_schema.py` | `visual.json` files | Malformed report visuals |
| `validate_layout.py` | report pages | Overlapping or off-canvas visuals |
| `validate_narrative.py` | narrative measures | References to something that no longer exists |
| `audit_template_safety.py` | any card/title/measure | Hardcoded figures, dates, or interpretations (this is a template — every number must be computed, never baked in) |
| `check_pbit_defaults.py <path-to.pbit>` | before exporting a `.pbit` | Parameter defaults that leaked a real customer path/endpoint/rate instead of shipping placeholders |

Example: `python docs/scripts/check_pbit_defaults.py "1. Local CSV/Consumption Central - Local CSV.pbit"`

Before opening a PR that changes model or report files, confirm (per `CONTRIBUTING.md`):
- The project opens in Power BI Desktop without error.
- A full refresh completes against the sample data.
- Every page renders (a visual can silently vanish without any file being invalid).
- No new hardcoded numbers, dates, or interpretations were introduced.

## CI

`.github/workflows/checks.yml` runs on push/PR and checks: Fabric notebooks are valid JSON with
parseable code cells, every internal Markdown doc link (including heading anchors) resolves, and
no committed CSV contains a real-looking tenant email address (sample data must stay synthetic).

## Key conventions

- **Loaders alias-match on column names.** New export shapes with extra/renamed columns can
  usually be absorbed without breaking existing loads — this is intentional, keep it that way
  when touching ingestion queries/notebooks.
- **Fabric partitions must degrade gracefully.** `GetTable` wraps reads in `try ... otherwise
  null` so a missing Lakehouse table returns null instead of raising (a missing table raises by
  default, unlike a missing CSV which already returns null).
- **`studio_agent` / `studio_user` are month-to-date snapshots** — always filter to the latest
  `snapshot_month`, never sum across snapshots, or credits will be double-counted.
- **Blank values in `isKey` columns invalidate the whole table** — filter blanks out of bridge/key
  tables (e.g. `Agent Bridge`, `Studio Environment`) rather than letting them through; keep the
  underlying fact rows intact.
- **Never write TMDL with Python's `utf-8-sig`** — it adds a BOM and Power BI Desktop refuses to
  load the whole project with only a vague error naming the first offending file.
- **Two separate templates (CSV vs. Fabric), not one with a mode switch.** Power Query registers
  data sources at parse time, so branching on a single parameter combines CSV and SQL sources in
  one partition and trips `Formula.Firewall`. Keep the CSV and Fabric projects/templates separate.
- **`.pbit` files store parameter *defaults*, not data.** Before exporting, parameters must be
  reset to shipping placeholders (see the table in `docs/BUILD.md`) — this is the step most likely
  to leak a real customer path/rate or (if skipped entirely) ship `null` parameters that crash
  every table via `VivaPeriodStart`.
- **Two Viva export shapes must both keep working**: de-identified (`PersonId` /
  `PeopleHistoricalId`) and identified (`UserPrincipalName` / `EntraId`, no person-map files). Test
  ingestion changes against both — see `1. Local CSV/sample-data/` for the de-identified sample.
- **Derive narrative, never hardcode it.** Any card/title text describing a trend must be computed
  from measures, since every customer's data differs (`audit_template_safety.py` enforces this).
