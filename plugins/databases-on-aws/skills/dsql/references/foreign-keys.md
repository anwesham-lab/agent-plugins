# Native Foreign Keys

Aurora DSQL supports foreign key constraints. Preserve supported PostgreSQL foreign keys during
migration.

## New Tables

Create the referenced table first. The referenced columns **MUST** have a matching
non-deferrable `PRIMARY KEY` or `UNIQUE` constraint.

Run `dsql_lint(fix=True)` before each DDL statement. Handle every diagnostic as described in
[dsql-lint.md](dsql-lint.md). Proceed only when no diagnostic is unfixable, and execute the
returned `fixed_sql`, not the pre-lint input. Run each statement in a separate `transact` call:

```python
customers_ddl = """CREATE TABLE customers (
  tenant_id UUID NOT NULL,
  customer_id UUID NOT NULL,
  name TEXT NOT NULL,
  PRIMARY KEY (tenant_id, customer_id)
)"""
customers_lint = dsql_lint(sql=customers_ddl, fix=True)
if customers_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([customers_lint["fixed_sql"]])

orders_ddl = """CREATE TABLE orders (
  tenant_id UUID NOT NULL,
  order_id UUID NOT NULL,
  customer_id UUID NOT NULL,
  PRIMARY KEY (tenant_id, order_id),
  CONSTRAINT orders_customer_fkey
    FOREIGN KEY (tenant_id, customer_id)
    REFERENCES customers (tenant_id, customer_id)
    NOT DEFERRABLE
)"""
orders_lint = dsql_lint(sql=orders_ddl, fix=True)
if orders_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([orders_lint["fixed_sql"]])
```

For multi-tenant data, the foreign key **MUST** include `tenant_id` on both sides. Referencing
`customer_id` alone can permit a referencing row to use another tenant's referenced row.

## Supported Options

| Feature                    | Supported behavior                                                                                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Referential actions        | `NO ACTION`, `RESTRICT`, `CASCADE`, `SET NULL`, `SET DEFAULT`                                                                                                      |
| Referencing-column subsets | `ON DELETE SET NULL` and `ON DELETE SET DEFAULT` may name a subset of referencing columns. `ON UPDATE` does not support this subset form.                          |
| Match types                | `MATCH SIMPLE` (default) permits any null key component. `MATCH FULL` requires all key components to be null or all to be non-null.                                |
| Deferrability              | `NOT DEFERRABLE` checks after each statement. `DEFERRABLE` defaults to `INITIALLY IMMEDIATE`; use `INITIALLY DEFERRED` to check at commit.                         |
| Transaction mode changes   | Run `SET CONSTRAINTS ... IMMEDIATE\|DEFERRED` inside an explicit transaction. It changes only deferrable foreign keys. `RESTRICT` actions always remain immediate. |
| Retroactive checks         | Changing a deferrable foreign key to `IMMEDIATE` checks outstanding changes immediately.                                                                           |

Use `MATCH SIMPLE` or `MATCH FULL`. PostgreSQL does not implement `MATCH PARTIAL`; Aurora DSQL
inherits this behavior.

For `SET NULL`, each referencing column selected by the action **MUST** be nullable. An `ON DELETE
SET NULL (column_name)` subset can leave other key columns, such as `tenant_id`, as `NOT NULL`.
With `SET DEFAULT`, each resulting non-null default **MUST** match a referenced row.

Run `SET CONSTRAINTS` and the affected DML in the same `transact` call. A standalone call commits
immediately and cannot change the mode of a later transaction. Keep constraints `NOT DEFERRABLE`
unless a transaction must write a referencing row before its referenced row. For that case, make
only the required constraint deferrable:

```python
deferrable_ddl = """ALTER TABLE orders
  ALTER CONSTRAINT orders_customer_fkey
  DEFERRABLE INITIALLY IMMEDIATE"""
deferrable_lint = dsql_lint(sql=deferrable_ddl, fix=True)
if deferrable_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([deferrable_lint["fixed_sql"]])
```

Then defer it only for the transaction that needs the reversed write order:

```python
transact([
    "SET CONSTRAINTS orders_customer_fkey DEFERRED",
    """INSERT INTO orders (tenant_id, order_id, customer_id)
       VALUES (
         '00000000-0000-0000-0000-000000000001',
         '00000000-0000-0000-0000-000000000002',
         '00000000-0000-0000-0000-000000000003'
       )""",
    """INSERT INTO customers (tenant_id, customer_id, name)
       VALUES (
         '00000000-0000-0000-0000-000000000001',
         '00000000-0000-0000-0000-000000000003',
         'Example customer'
       )"""
])
```

**SHOULD** default to `NO ACTION` and `NOT DEFERRABLE`. Use `CASCADE`, `SET NULL`, `SET DEFAULT`,
or deferral only when the user explicitly intends the behavior and confirms its impact.

## Existing Tables

Add a foreign key with `NOT VALID` so it applies to new writes without scanning existing
referencing rows:

```python
add_fk_ddl = """ALTER TABLE orders
  ADD CONSTRAINT orders_customer_fkey
  FOREIGN KEY (tenant_id, customer_id)
  REFERENCES customers (tenant_id, customer_id)
  NOT VALID"""
add_fk_lint = dsql_lint(sql=add_fk_ddl, fix=True)
if add_fk_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([add_fk_lint["fixed_sql"]])
```

Validate existing rows in a separate asynchronous operation:

```python
validate_fk_ddl = """ALTER TABLE ASYNC orders
  VALIDATE CONSTRAINT orders_customer_fkey"""
validate_fk_lint = dsql_lint(sql=validate_fk_ddl, fix=True)
if validate_fk_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([validate_fk_lint["fixed_sql"]])
```

Poll `sys.jobs` until validation reaches a terminal state and treat a failed job as a failed
migration. Alternatively, run `CALL sys.wait_for_job('<job_id>')` through a database client with
autocommit enabled, outside an explicit transaction.

Change an existing foreign key's deferrability directly with `ALTER TABLE ... ALTER CONSTRAINT
... [ NOT ] DEFERRABLE [ INITIALLY IMMEDIATE | INITIALLY DEFERRED ]`.

Before dropping a foreign key, verify the named constraint exists with `contype = 'f'`, explain
that the drop removes database-enforced referential integrity, and obtain confirmation. Then lint
and execute the direct drop:

```python
drop_fk_ddl = """ALTER TABLE orders
  DROP CONSTRAINT orders_customer_fkey RESTRICT"""
drop_fk_lint = dsql_lint(sql=drop_fk_ddl, fix=True)
if drop_fk_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([drop_fk_lint["fixed_sql"]])
```

Verify that the constraint no longer exists. Use `CASCADE` only after confirming the dependent
objects that DSQL will remove.

## Indexes

Create a referencing-side index separately when joins or referenced-row deletes and updates need
efficient lookup. Lint the index DDL before executing it:

```python
index_ddl = """CREATE INDEX ASYNC orders_customer_idx
  ON orders (tenant_id, customer_id)"""
index_lint = dsql_lint(sql=index_ddl, fix=True)
if index_lint["summary"]["errors"]:
    raise ValueError("Resolve unfixable diagnostics and re-lint before execution")
transact([index_lint["fixed_sql"]])
```

## Transactions and Concurrency

- Foreign key checks add reads to writes on referenced and referencing tables. Benchmark the
  workload before adding a constraint and confirm that its performance meets requirements.
- Rows changed by `CASCADE`, `SET NULL`, or `SET DEFAULT` count toward DSQL transaction row and
  data-size limits. Batch referenced-row changes so the total direct and cascaded writes remain
  within current limits.
- Concurrent referenced-row and referencing-row changes can produce retryable serialization
  failures. Retry the complete transaction on SQLSTATE `40001` using the patterns in
  [occ-retry-patterns.md](occ-retry-patterns.md).
- Keep heavily referenced key columns stable. Updating non-key columns does not conflict with a
  referencing insert, but concurrent changes to referenced keys can cause serialization failures.
- Application authorization and business validation can complement foreign keys, but do not
  replace database-enforced referential integrity.
