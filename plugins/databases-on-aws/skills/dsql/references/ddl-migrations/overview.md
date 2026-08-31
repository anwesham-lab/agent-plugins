# DSQL DDL Migration Guide - Overview

This guide provides the **Table Recreation Pattern** for schema modifications that require rebuilding tables.

## Table of Contents

1. [Destructive Operations Warning](#critical-destructive-operations-warning)
2. [Table Recreation Operations](#table-recreation-operations)
3. [Table Recreation Pattern Overview](#table-recreation-pattern-overview)
4. [Pre-Create Relationship and Dependency Gate](#pre-create-relationship-and-dependency-gate)
5. [Common Verify & Swap Pattern](#common-verify--swap-pattern)
6. [Recovery — Row Counts Do Not Match](#recovery--row-counts-do-not-match)
7. [Recovery — Relationship Restoration Fails](#recovery--relationship-restoration-fails)
8. [Best Practices Summary](#best-practices-summary)

For column-level operations, see [column-operations.md](column-operations.md).
For constraint and structural operations, see [constraint-operations.md](constraint-operations.md).
For batched migration patterns, see [batched-migration.md](batched-migration.md).

---

## CRITICAL: Destructive Operations Warning

**The Table Recreation Pattern involves DESTRUCTIVE operations that can result in DATA LOSS.**

Table recreation requires dropping the original table, which is **irreversible**. If any step fails after the original table is dropped, data may be permanently lost.

### Mandatory User Verification Requirements

Agents MUST obtain explicit user approval before executing migrations on live tables:

1. **MUST present the complete migration plan** to the user before any execution
2. **MUST clearly state** that this operation will DROP the original table
3. **MUST confirm** the user has a current backup or accepts the risk of data loss
4. **MUST verify with the user** at each checkpoint before proceeding:
   - Before creating the new table structure
   - Before beginning data migration
   - Before dropping the original table (CRITICAL CHECKPOINT)
   - Before renaming the new table
5. **MUST NOT proceed** with any destructive action without explicit user confirmation
6. **MUST recommend** performing migrations on non-production environments first

### Risk Acknowledgment

Before proceeding, the user MUST confirm:

- [ ] They understand this is a destructive operation
- [ ] They have a backup of the table data (or accept the risk)
- [ ] They approve the agent to execute each step with verification
- [ ] They understand the migration cannot be automatically rolled back after DROP TABLE

---

## Table Recreation Operations

The following ALTER TABLE operations MUST use the **Table Recreation Pattern**:

| Operation                      | Key Approach                                   |
| ------------------------------ | ---------------------------------------------- |
| DROP COLUMN                    | Exclude column from new table                  |
| ALTER COLUMN TYPE              | Cast data type in SELECT                       |
| ALTER COLUMN SET/DROP NOT NULL | Change constraint in new table definition      |
| ALTER COLUMN SET/DROP DEFAULT  | Define default in new table definition         |
| ADD PRIMARY KEY                | Include constraint in new table definition     |
| DROP non-FK constraint         | Remove constraint from new table definition    |
| MODIFY PRIMARY KEY             | Define new PK, validate uniqueness first       |
| Split/Merge Columns            | Use SPLIT_PART, SUBSTRING, or CONCAT in SELECT |

The following operations are supported directly:

- `ALTER TABLE ... ADD CONSTRAINT ... FOREIGN KEY ... NOT VALID`
- `ALTER TABLE ... ADD CONSTRAINT ... UNIQUE USING INDEX`
- `ALTER TABLE ASYNC ... VALIDATE CONSTRAINT`
- `ALTER TABLE ... DROP CONSTRAINT` for a foreign key
- `ALTER TABLE ... RENAME COLUMN` - Rename a column
- `ALTER TABLE ... RENAME TO` - Rename a table
- `ALTER TABLE ... ADD COLUMN` - Add a new column

---

## Table Recreation Pattern Overview

**MUST** follow these phases with user verification at each checkpoint:

1. **Plan and confirm** — present the migration plan and obtain approval.
2. **Complete the pre-create gate** — inventory relationships and dependent views, prove the
   replacement remains compatible, and stop before the write fence when it does not.
3. **Create and migrate** — derive the complete replacement definition from the source, change only
   the requested property, and copy data in bounded batches.
4. **Verify and swap** — synchronize under a write fence, obtain irreversible-action confirmation,
   drop recorded inbound FKs directly, drop the original table, and rename the replacement.
5. **Restore and re-index** — restore every relationship to its recorded validation state, recreate
   indexes with `ASYNC`, verify completion, and release the write fence.

### Transaction Rules

Verify current limits via `awsknowledge`: `aurora dsql transaction limits`

- **MUST batch** migrations exceeding 3,000 row mutations
- **PREFER batches of 500-1,000 rows** for optimal throughput
- **MUST respect** 10 MiB data size per transaction
- **MUST respect** 5-minute transaction duration

---

## Pre-Create Relationship and Dependency Gate

Complete this gate before creating `target_table_new`. Every operation-specific recreation example
inherits this requirement.

1. **Resolve the actual target and persist exact FK definitions.** Use the schema-qualified table
   supplied by the user; do not assume `public`.

   ```python
   from safe_query import build, literal

   target_table = "app.target_table"
   target_oid = readonly_query(build(
       "SELECT {target_table}::regclass::oid AS oid",
       target_table=literal(target_table),
   ))[0]["oid"]

   relationships = readonly_query(
       build(
           """SELECT c.conname,
                     c.conrelid AS referencing_table_oid,
                     c.confrelid AS referenced_table_oid,
                     format('%I.%I', rn.nspname, r.relname) AS referencing_table,
                     format('%I.%I', pn.nspname, p.relname) AS referenced_table,
                     format('ALTER TABLE %I.%I DROP CONSTRAINT %I',
                            rn.nspname, r.relname, c.conname) AS drop_constraint_ddl,
                     c.convalidated,
                     pg_get_constraintdef(c.oid) AS definition
              FROM pg_constraint c
              JOIN pg_class r ON r.oid = c.conrelid
              JOIN pg_namespace rn ON rn.oid = r.relnamespace
              JOIN pg_class p ON p.oid = c.confrelid
              JOIN pg_namespace pn ON pn.oid = p.relnamespace
              WHERE c.contype = 'f'
                AND (c.conrelid = {target_table}::regclass
                     OR c.confrelid = {target_table}::regclass)""",
           target_table=literal(target_table),
       )
   )

   self_relationships = [
       rel for rel in relationships
       if rel["referencing_table_oid"] == rel["referenced_table_oid"] == target_oid
   ]
   inbound_relationships = [
       rel for rel in relationships
       if rel["referenced_table_oid"] == target_oid
       and rel["referencing_table_oid"] != target_oid
   ]
   outbound_relationships = [
       rel for rel in relationships
       if rel["referencing_table_oid"] == target_oid
       and rel["referenced_table_oid"] != target_oid
   ]
   ```

   Persist `relationships` with the migration record before any destructive step. Generated
   restore DDL **MUST** replay each recorded definition exactly, preserving composite columns,
   `MATCH`, referential actions, and deferrability. Normalize any recorded `NOT VALID` suffix,
   then append `NOT VALID` exactly once.

2. **Detect dependent views before the write fence.** Query `pg_depend` and `pg_rewrite` for views
   that reference the target:

   ```python
   dependent_views = readonly_query(build(
       """SELECT DISTINCT format('%I.%I', n.nspname, v.relname) AS dependent_view,
                         pg_get_viewdef(v.oid, true) AS definition
          FROM pg_depend d
          JOIN pg_rewrite rw ON rw.oid = d.objid
          JOIN pg_class v ON v.oid = rw.ev_class
          JOIN pg_namespace n ON n.oid = v.relnamespace
          WHERE d.refobjid = {target_table}::regclass
            AND d.classid = 'pg_rewrite'::regclass
            AND v.oid <> {target_table}::regclass""",
       target_table=literal(target_table),
   ))
   ```

   When `dependent_views` is non-empty, **MUST** stop before creating the replacement table and
   present a separate dependency-ordered view migration plan for approval. **MUST NOT** enter the
   write fence or use `CASCADE` to bypass this gate.

3. **Prove the replacement remains restorable.** Derive the complete replacement definition from
   the source table and preserve every unchanged column, key, constraint, and default. Change only
   the requested property. Compare each inbound FK with the proposed replacement: the referenced
   columns, types, unique key, and action **MUST** remain compatible.

   Declare originally validated outbound FKs on `target_table_new`. Add originally unvalidated
   outbound FKs after the copy with `NOT VALID`. Handle each self-referential FK exactly once as an
   outbound relationship: rewrite its referenced table to `target_table_new`, declare it during
   creation when originally validated or add it after the copy when originally `NOT VALID`, then
   exclude it from inbound drop and restore loops. Do not attach a test FK from a live referencing
   table to `target_table_new`; a `NOT VALID` constraint still applies to new writes.

---

## Common Verify & Swap Pattern

All migrations end with this pattern after completing the
[Pre-Create Relationship and Dependency Gate](#pre-create-relationship-and-dependency-gate),
creating the replacement table, and copying the data.

**CRITICAL: MUST obtain explicit user confirmation before DROP TABLE and MUST hold a maintenance
window or application write fence from the first inbound-FK drop until every relationship is
restored to its recorded validation state.**

1. **Start the write fence, synchronize, and verify.** Stop writes to `target_table` and every
   referencing table. Apply the final data catch-up, then compare primary-key sets, transformed
   values, row counts, and relationship anti-joins.

   ```python
   readonly_query("SELECT COUNT(*) FROM target_table")
   readonly_query("SELECT COUNT(*) FROM target_table_new")
   ```

   Present the final comparison and obtain confirmation. Keep the fence active until every
   relationship is restored to its recorded validation state.

   **MUST** display: "The original and replacement tables have been verified. The next step
   permanently drops the original table and cannot be rolled back. Proceed? (yes/no)"
   **MUST NOT** continue without an explicit `yes`.

2. **Drop recorded inbound FKs and swap.** **MUST NOT** use `DROP TABLE ... CASCADE` in this
   workflow. `CASCADE` silently removes inbound constraints and bypasses exact
   relationship-by-relationship restoration and verification.

   Iterate over the complete persisted inbound relationship set. Lint and execute one direct
   `ALTER TABLE ... DROP CONSTRAINT` per transaction, and append a relationship to
   `dropped_inbound` only after its drop succeeds:

   ```python
   dropped_inbound = []
   for relationship in inbound_relationships:
       lint_result = dsql_lint(sql=relationship["drop_constraint_ddl"], fix=True)
       # Handle every diagnostic per dsql-lint.md; execute only clean fixed_sql.
       transact([lint_result["fixed_sql"]])
       dropped_inbound.append(relationship)
   ```

   Apply these failure branches before continuing:

   - **Any inbound-FK drop fails:** restore every entry in `dropped_inbound` from its persisted
     definition in reverse drop order, return each to its recorded validation state, verify full
     relationship coverage, and only then release the write fence. If restoration fails, keep the
     fence active and continue the persisted recovery plan.
   - **`DROP TABLE target_table` fails:** the original table still exists. Restore and verify every
     entry in `dropped_inbound` before releasing the fence or exiting the workflow.
   - **The rename fails after the original table was dropped:** the destructive boundary has been
     crossed. **MUST** keep the fence active, diagnose and retry
     `ALTER TABLE target_table_new RENAME TO target_table`, then continue with relationship
     restoration. **MUST NOT** return or release the fence while the canonical table name or any
     recorded relationship is missing.

   After every inbound FK has been dropped successfully, execute the swap as separate DDL
   transactions:

   ```python
   transact(["DROP TABLE target_table"])
   transact(["ALTER TABLE target_table_new RENAME TO target_table"])
   ```

3. **Restore exact definitions and verify completion.** For each recorded inbound relationship,
   lint and execute:

   ```text
   ALTER TABLE <recorded_referencing_table>
     ADD CONSTRAINT <recorded_constraint_name>
     <recorded_pg_get_constraintdef>
     NOT VALID
   ```

   If the recorded `convalidated` value was true, validate the restored constraint with
   `ALTER TABLE ASYNC ... VALIDATE CONSTRAINT`, capture its `job_id`, poll `sys.jobs` to
   `completed` or `failed`, inspect `details` on failure, and verify
   `pg_constraint.convalidated = true`. Preserve an originally unvalidated constraint as
   `NOT VALID` unless the user explicitly approves strengthening that invariant. Verify the
   recorded outbound and self-referential constraints survived the rename, recreate indexes with
   `CREATE INDEX ASYNC`, and only then resume writes.

### Recovery — Row Counts Do Not Match

When `target_table_new` has fewer rows than `target_table`, treat the migration as incomplete.
The original table still holds the authoritative data, so recovery is always possible — **MUST NOT**
proceed with `DROP TABLE` until the counts agree.

1. **Diagnose** — find the missing rows by comparing ranges (for cursor-based migrations, query
   `target_table` for IDs greater than `MAX(id)` in `target_table_new`; for OFFSET-based, check
   which batch dropped rows by re-running the SELECT portion of each batch and comparing counts).
2. **Retry the missing batches** — insert the gap rows into `target_table_new` using the same
   batch pattern from [batched-migration.md](batched-migration.md). Because each `INSERT … SELECT`
   is idempotent on primary key, re-running completed batches is safe; they will collide on PK
   and error without writing duplicate data.
3. **If a type cast or constraint rejected rows** — migration cannot complete until the data is
   reconciled. Fix the source data in `target_table` (or adjust the new table's constraint),
   then re-run the missing batches.
4. **Escape hatch** — if diagnosis stalls, drop `target_table_new` and restart the migration
   from a clean slate. The original table is untouched, so no data is at risk.

Re-run the count comparison after each retry. Only proceed to `DROP TABLE` once
`COUNT(*)` matches exactly.

### Recovery — Relationship Restoration Fails

- **Before the original table is dropped:** restore any removed inbound FKs from the persisted
  definitions, verify them, then release the write fence.
- **After the swap:** keep the write fence active. Resume the persisted restore plan from the
  first missing constraint and complete relationship coverage before continuing.
- **Validation failure:** inspect `sys.jobs.details`, fix the reported data or schema issue, and
  rerun validation when the recorded `convalidated` state was true. Resume writes only after
  every FK is restored to its recorded validation state.

---

## Best Practices Summary

### User Verification (CRITICAL)

- **MUST present** complete migration plan to user before any execution
- **MUST obtain** explicit user confirmation before DROP TABLE operations
- **MUST verify** with user at each checkpoint during migration
- **MUST NOT** proceed with destructive actions without explicit user approval
- **MUST recommend** testing migrations on non-production data first
- **MUST confirm** user has backup or accepts data loss risk

### Technical Requirements

- **MUST validate** data compatibility before type changes
- **MUST batch** tables exceeding 3,000 rows
- **MUST verify** row counts before and after migration
- **MUST complete** the pre-create relationship and dependent-view gate
- **MUST inventory and restore** inbound, outbound, and self-referential foreign keys
- **MUST drop directly** every recorded inbound foreign key and preserve its restore definition
- **MUST NOT use** `DROP TABLE ... CASCADE` during relationship-preserving table recreation
- **MUST recreate** indexes after table swap using ASYNC
- **MUST NOT** drop original table until new table is verified
- **PREFER** cursor-based batching for very large tables
- **PREFER** batches of 500-1,000 rows for optimal throughput
