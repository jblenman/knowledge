# SQL Server Views and Performance

View semantics, when views cost you, and what `WITH SCHEMABINDING` actually does. Centered on T-SQL but most concepts transfer to other RDBMSs with cosmetic differences.

## Regular Views Are Macros

A non-indexed view is a saved SELECT statement, not a materialized result. The optimizer **inlines the view definition** at query time and optimizes the combined query as a single unit.

```sql
CREATE VIEW vw_customer_orders AS
SELECT customer_id, order_date, region FROM customer_orders_archive;

-- This query:
SELECT * FROM vw_customer_orders WHERE order_date >= '2025-10-01';

-- Becomes effectively:
SELECT customer_id, order_date, region FROM customer_orders_archive
WHERE order_date >= '2025-10-01';
```

Indexes on `order_date` get used. Execution plan is identical to a direct query. **Zero overhead from "going through the view."**

This is called *predicate pushdown* — the outer WHERE pushes into the view's body before execution.

## When Views Actually Hurt

Predicate pushdown breaks down when the view contains:

| Construct | Why it blocks pushdown |
|---|---|
| `DISTINCT`, `GROUP BY`, aggregations | Aggregate must compute over full set before outer filters apply (often) |
| Window functions (`ROW_NUMBER() OVER ...`) | Outer WHERE can't push past the window's PARTITION |
| `TOP` / `ORDER BY` | Forces materialization of top-N before outer filtering |
| Non-trusted foreign keys + joins | Optimizer can't prove join tables are unnecessary |
| Scalar UDFs (non-inlined) | Can force row-by-row execution |
| `SELECT *` + joins | May pull more columns/tables than outer query needs |
| Nested views (view-on-view-on-view) | Plan generation gets harder; plan-time limits hit |

**Diagnostic:** `SET STATISTICS IO ON; SET STATISTICS TIME ON;` and compare logical reads / CPU between the view and the equivalent direct query. Differing plan shapes (extra sort, hash aggregate, eager spool) reveal real cost.

## WITH SCHEMABINDING

Common misconception: `WITH SCHEMABINDING` improves view performance. **It does not by itself.**

What it actually does:

1. **Locks the view to underlying schema.** Can't `DROP TABLE` or `ALTER TABLE … DROP COLUMN` on anything the view references. Safety / integrity.
2. **Requires two-part names** (`dbo.tbl`, not `tbl`) and **forbids `SELECT *`** in the view body. Mechanical hygiene.
3. **Is a prerequisite for indexed views** — but schemabinding alone doesn't materialize anything. The optimizer still inlines a schemabound view exactly like any other view.

**To check if a view is actually indexed (materialized):**

```sql
SELECT name FROM sys.indexes
WHERE object_id = OBJECT_ID('dbo.your_view_name');
```

No rows → not materialized → schemabinding has zero performance impact.

## Indexed Views (Materialized Views)

The path to real perf gain via views:

```sql
CREATE VIEW dbo.vw_thing WITH SCHEMABINDING AS
    SELECT col1, col2, COUNT_BIG(*) AS cnt
    FROM dbo.base
    GROUP BY col1, col2;

CREATE UNIQUE CLUSTERED INDEX IX_vw_thing ON dbo.vw_thing(col1, col2);
```

What you get:
- Results stored physically; reads serve from the index
- Optimizer can substitute the indexed view even when you query base tables directly (Enterprise edition; "view matching")
- Big gain on pre-aggregated data queried repeatedly

What it costs:
- Every INSERT/UPDATE/DELETE on base tables must update the indexed view
- Restrictions: deterministic functions only, `COUNT_BIG(*)` required, no `OUTER JOIN`, no subqueries, no `UNION`, etc.
- Schema changes get harder (schemabinding traps)

**Reach for it when:** repeated heavy aggregation against tables with much heavier reads than writes.
**Don't reach for it when:** "I want my view to be faster" — usually the answer is fix the view's logic or query base tables directly.

## Lookup Tables for UI Dropdowns

Recurring pattern: a UI dropdown needs a distinct list of values from a large transactional table.

**Anti-pattern:** Stored proc on dropdown render queries the large table for `SELECT DISTINCT col FROM big_table ...`. Degrades as `big_table` grows.

**Pattern:** Nightly scheduled job loads a lookup table with the distinct list. Dropdown query is `SELECT * FROM lkp_thing ORDER BY sort_order, name` against a tiny indexed lookup.

Trade-off: dropdown isn't real-time fresh. For most search-UI dropdowns this doesn't matter — values change rarely and forced-refresh is a one-line manual run.

When dropdown contents need to match downstream filtering (e.g., search backend filters by confidence score), apply the same filter in the loader proc. The lookup becomes "values currently in scope for search," and filter changes are single-place edits.

Use a `SortOrder` column for "English/Spanish at top, then alphabetical" — beats `UNION`-with-hardcoded-literals patterns that drift when requirements add a third privileged language.

## Stored Procedure Wrapper Doesn't Add View Overhead

Common confusion: "is calling a view from a stored proc slower than a direct query in the proc?" No — the proc layer is identical in both cases. The performance comparison is between the view's logic and the direct query's logic, not the wrapper.

**Separate concern:** stored proc *parameter sniffing* can produce a bad cached plan if the proc compiles for an atypical parameter. Mitigations: `OPTION (RECOMPILE)`, local variable copies, `OPTIMIZE FOR` hints. Proc-level issue, applies whether proc queries a view or a base table.

## Sources

- [SQL Server views](https://learn.microsoft.com/en-us/sql/relational-databases/views/views)
- [Indexed views](https://learn.microsoft.com/en-us/sql/relational-databases/views/create-indexed-views)
- [CREATE VIEW (Transact-SQL) — SCHEMABINDING](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-view-transact-sql)
