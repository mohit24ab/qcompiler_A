# SHARED CONTRACT — READ BEFORE ANY MEMBER STARTS

All three members build against this document. It is frozen after Person A completes Phase A6.
Any change to this file after freeze requires all three members to agree, and must be
announced before the change is committed.

---

## 1. Project

**Compiler-Driven Data Analytics Pipeline Generation and Query Optimization Framework**

A SQL-like query goes in. An optimized relational plan is produced. Executable source code
is generated from that plan and run. Performance is measured against an unoptimized baseline.

---

## 2. Stack (non-negotiable, keeps three machines identical)

- Python 3.11
- `sqlglot` — SQL text parsing only (we do NOT use its optimizer; that is our project)
- `pyarrow` — Parquet/CSV reading, columnar buffers
- `numpy` — vectorized primitives in generated code
- `pytest` — all testing
- No Spark, no DuckDB, no Polars, no pandas in the execution path.
  DuckDB may be installed **only** as an oracle for correctness cross-checks in tests.

---

## 3. Repository layout

```
qcompiler/
  ir/            # Person A  — plan nodes, expressions, visitor utilities
  catalog/       # Person A  — table metadata, schemas, statistics
  frontend/      # Person A  — sqlglot AST -> our IR, name resolution, type checking
  optimizer/     # Person B  — pass framework, rewrite rules, cost model
  codegen/       # Person C  — plan -> Python source, emitted operator templates
  runtime/       # Person C  — naive reference executor, runtime helpers
  bench/         # Person A (later) + Person C — benchmark suite, harness, results
  tests/         # all three, namespaced: test_ir_*, test_opt_*, test_codegen_*
  data/          # generated benchmark datasets (gitignored except the generator script)
```

**Ownership rule:** you may READ any directory. You may WRITE only your own.
Cross-directory changes go through the owner.

---

## 4. The IR contract (the single most important section)

Every plan node is an immutable dataclass exposing:

```python
children  -> tuple[PlanNode, ...]
schema()  -> list[tuple[str, DType]]     # output column names and types
replace_children(new_children) -> PlanNode
```

Plan nodes:

| Node | Fields |
|---|---|
| `Scan` | `table: str`, `columns: list[str] \| None`, `pushed_predicate: Expr \| None` |
| `Filter` | `child`, `predicate: Expr` |
| `Project` | `child`, `exprs: list[tuple[Expr, str]]`  (expression, output alias) |
| `Join` | `left`, `right`, `condition: Expr`, `kind: 'inner' \| 'left'` |
| `Aggregate` | `child`, `group_keys: list[Expr]`, `aggs: list[tuple[AggCall, str]]` |
| `Sort` | `child`, `keys: list[tuple[Expr, bool]]`  (expression, descending) |
| `Limit` | `child`, `n: int` |

Expression nodes:

| Node | Fields |
|---|---|
| `ColumnRef` | `table: str \| None`, `name: str` |
| `Literal` | `value`, `dtype: DType` |
| `BinaryOp` | `op: str`, `left: Expr`, `right: Expr` |
| `UnaryOp` | `op: str`, `operand: Expr` |
| `AggCall` | `func: 'sum'\|'count'\|'avg'\|'min'\|'max'`, `arg: Expr \| None` |

`DType` is an enum: `INT`, `FLOAT`, `STRING`, `BOOL`, `DATE`.

**Invariant:** optimizer passes return a NEW tree. Nothing mutates in place. Ever.

---

## 5. Module interfaces

```python
# frontend/  (Person A)
def parse_and_bind(sql: str, catalog: Catalog) -> PlanNode: ...

# optimizer/  (Person B)
def optimize(plan: PlanNode, catalog: Catalog) -> tuple[PlanNode, list[PassTrace]]: ...

# codegen/  (Person C)
def generate(plan: PlanNode, catalog: Catalog) -> str: ...   # returns Python SOURCE TEXT
def compile_and_run(source: str, tables: dict) -> Table: ...

# runtime/  (Person C)
def interpret(plan: PlanNode, tables: dict) -> Table: ...    # naive baseline, correctness oracle
```

`PassTrace` records `(pass_name, plan_before, plan_after, changed: bool)` — this is what
produces the before/after evidence in the final report. Person B must populate it from day one.

---

## 6. Catalog contract

```python
catalog.schema(table)   -> list[tuple[str, DType]]
catalog.row_count(table)-> int
catalog.stats(table, column) -> ColumnStats(ndv, min, max, null_count)
```

Person B's cost model consumes exactly this and nothing more. If B needs another statistic,
B asks A for it rather than computing it inside the optimizer.

---

## 7. Definition of correct

A query is correct if `interpret(plan)` and `compile_and_run(generate(optimize(plan)))`
return identical rows (order-insensitive unless the query has `ORDER BY`).
This differential check runs on every query in the suite, on every commit.

**An optimizer that returns wrong answers quickly is worth zero marks.** Correctness gates everything.

---

## 8. Git discipline

- `main` stays green. Every push runs `pytest`.
- One branch per phase: `a/phase-3-binder`, `b/phase-4-pushdown`, `c/phase-5-fusion`.
- No member merges to main while `pytest` fails.
- Commit the generated Python source for a few sample queries into `bench/samples/` —
  examiners love seeing the actual emitted code.

---

## 9. Week-zero blocking risk

Persons B and C cannot start real work until the IR exists. Person A must deliver
Phase A1 (IR nodes + pretty printer) within the first three days, even if it is
hand-constructed and untested. B and C build against hand-written IR trees until
the parser lands.
