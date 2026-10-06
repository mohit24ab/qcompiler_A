# PERSON A — FRONTEND, IR & CATALOG
## Claude Code Master Prompt

You are implementing the **frontend** of a compiler-driven data analytics framework.
Read `00-SHARED-CONTRACT.md` in the repo root before writing any code. It defines the
IR node shapes, module interfaces, and directory ownership. Treat it as law.

**You own:** `ir/`, `catalog/`, `frontend/`, and later `bench/harness/`.
**You must not modify:** `optimizer/`, `codegen/`, `runtime/`.

---

## EXECUTION DISCIPLINE — READ CAREFULLY

- Work on **exactly one phase at a time**. Never begin the next phase on your own.
- At the end of each phase, stop and output the **Phase Report** in the format below.
- Then wait. I will reply `NEXT` when I have verified the work. Only then continue.
- If a phase turns out to need a decision I have not specified, stop and ask before choosing.
- Write tests as part of each phase, not afterward. A phase with no tests is not complete.

**Phase Report format:**

```
## PHASE <id> COMPLETE — <name>
Files created/modified:  <list with line counts>
What it does:            <3-5 lines, plain language>
Tests added:             <names, and what each proves>
Test result:             <pytest output summary>
Contract impact:         <anything other members must know>
Known gaps:              <what is deliberately unfinished>
Ready for: PHASE <next id>
```

---

## PHASE A1 — IR core and plan printer  *(deliver within 3 days — B and C are blocked on this)*

- Create `ir/nodes.py` with every plan node in Contract §4 as a frozen dataclass.
- Create `ir/expr.py` with every expression node.
- Create `ir/dtype.py` with the `DType` enum.
- Implement `children`, `replace_children`, and a generic `transform_post_order(fn)` helper
  in `ir/visitor.py`. The optimizer will lean on this heavily, so make it clean.
- Implement `ir/printer.py` — renders a plan as an indented tree, e.g.

```
Aggregate[group=region, aggs=avg(amount) AS avg_amt]
  Filter[date >= '2024-01-01']
    Scan[sales]
```

  Every member will use this for demos and debugging. Make the output pretty.
- `schema()` on each node: for now, `Scan` may raise `NotImplementedError` until the
  catalog lands in A2. All other nodes derive schema from their children.
- Tests: construct three plans by hand, assert printer output, assert immutability
  (mutating a returned tree raises), assert `replace_children` preserves node type.

**Then STOP and report. Announce to B and C that the IR is available.**

---

## PHASE A2 — Catalog and data loading

- `catalog/catalog.py` implementing the three methods in Contract §6.
- `catalog/loader.py` — read Parquet and CSV via `pyarrow`, infer schema, register tables.
- `catalog/stats.py` — compute per-column `ndv` (exact is fine at our scale), min, max,
  null count on registration. Person B's cost model depends on this being correct.
- Wire `Scan.schema()` to the catalog.
- Tests: register a small CSV and a small Parquet file, assert schema inference,
  assert stats match hand-computed values.

**STOP. Report.**

---

## PHASE A3 — SQL to IR binder

- `frontend/binder.py` implementing `parse_and_bind(sql, catalog) -> PlanNode`.
- Use `sqlglot.parse_one(sql)` to get their AST, then walk it and construct OUR IR.
  Do not use any sqlglot optimizer or transformation function — only the parser.
- Supported surface for v1: `SELECT` list with expressions and aliases, `FROM` with one
  or more tables, `INNER JOIN ... ON`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`,
  arithmetic and comparison operators, `AND`/`OR`/`NOT`, the five aggregates.
- Build the canonical unoptimized shape: `Limit(Sort(Project(Filter(Aggregate(Filter(Join(Scan,Scan)))))))`.
  Deliberately naive — a full `Scan` with no pruning, filters sitting above joins.
  **This naive shape is the baseline the whole project measures against, so do not
  optimize anything here, even where it is obvious.**
- Tests: 10 queries of increasing complexity, each asserting the exact printed plan.

**STOP. Report.**

---

## PHASE A4 — Semantic analysis

- `frontend/resolver.py` — resolve every `ColumnRef` to a specific table, error clearly
  on ambiguous or unknown columns.
- `frontend/typecheck.py` — propagate `DType` through every expression, reject invalid
  operations (string minus int, aggregate inside `WHERE`, non-grouped column in a
  `GROUP BY` select list).
- Errors must carry the offending SQL fragment. Good error messages are demo material.
- Tests: 8 valid queries pass; 8 malformed queries each raise the specific expected error.

**STOP. Report.**

---

## PHASE A5 — Golden fixtures and query suite

- `tests/fixtures/queries/` — 20 queries: 6 single-table, 6 join, 5 aggregate, 3 nested/complex.
- For each, commit the bound (unoptimized) plan as a golden `.txt` file.
- `bench/data/generate.py` — generate the datasets these queries run against.
  Two scale factors: tiny (for tests, thousands of rows) and bench (millions of rows).
  Use a star-schema shape: one large fact table, three small dimension tables.
- Tests: every query binds without error; every plan matches its golden file.

**STOP. Report.**

---

## PHASE A6 — Contract freeze and handoff

- Verify every interface in Contract §5 exists with the exact stated signature.
- Write `docs/ir-reference.md` — one page per node, fields, schema rules, an example.
- Announce the freeze. Contract changes from here need all three members to agree.

**STOP. Report. From here your role changes.**

---

## PHASE A7 — Differential test harness  *(co-owned with Person C)*

- `bench/harness/differential.py` — for every query in the suite, run:
  naive interpret vs. generated-code execution of the optimized plan; compare row sets.
- Order-insensitive comparison unless the query has `ORDER BY`.
- On mismatch, print: the query, both plans, the first five differing rows.
  Debuggability here saves the team days later.
- Wire it into `pytest` so it runs on every commit.

**STOP. Report.**

---

## PHASE A8 — Results pipeline and report assets

- `bench/report.py` — run the full benchmark matrix and emit:
  a results table (query × configuration × runtime/rows-scanned/peak-memory),
  a bar chart of naive vs. fully-optimized runtime per query,
  and an ablation chart (each pass disabled in turn) using Person B's `PassTrace` data.
- Export as CSV plus PNG so they drop straight into the report and slides.

**STOP. Report.**

---

## Notes for you specifically

Your phases A1–A3 are on the critical path for the entire team. Speed matters more than
polish in the first week; you can harden the binder later, but B and C genuinely cannot
start until the IR exists. Ship A1 fast and rough if you have to.

Begin with **PHASE A1**. Do not proceed past it.
