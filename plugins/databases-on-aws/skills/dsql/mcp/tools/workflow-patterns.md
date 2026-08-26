# MCP Common Workflow Patterns

Part of [Aurora DSQL MCP Tools Reference](../mcp-tools.md).

---

## Pattern 1: Explore Schema

```python
from safe_query import build

# Step 1: List all tables (fully static — build() documents safe-query intent)
readonly_query(build(
    "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'",
))

# Step 2: Get schema for specific table
get_schema("entities")

# Step 3: Query data (fully static — build() documents safe-query intent)
readonly_query(build(
    "SELECT * FROM entities LIMIT 10",
))
```

## Pattern 2: Create Table with Index

```python
# WRONG - Combined DDL and index in single transaction
transact([
  "CREATE TABLE entities (...)",
  "CREATE INDEX ASYNC idx_tenant ON entities(tenant_id)"  # ❌ Will fail
])

# CORRECT - Separate transactions
transact(["CREATE TABLE entities (...)"])
transact(["CREATE INDEX ASYNC idx_tenant ON entities(tenant_id)"])
```

## Pattern 3: Safe Data Migration

```python
from safe_query import build, allow, regex, TENANT_SLUG

STATUSES = {"active", "archived", "pending"}

# Step 1: Add column
transact(["ALTER TABLE entities ADD COLUMN status VARCHAR(50)"])

# Step 2: Populate in batches — separate transactions, under 3,000 rows each
populate = build(
    "UPDATE entities SET status = {s} "
    "WHERE entity_id IN ("
    "    SELECT entity_id FROM entities WHERE status IS NULL LIMIT 1000"
    ")",
    s=allow("active", STATUSES),
)
transact([populate])
transact([populate])

# Step 3: Verify (fully static — build() documents safe-query intent)
readonly_query(build(
    "SELECT COUNT(*) AS total, COUNT(status) AS with_status FROM entities",
))

# Step 4: Create index in a separate transaction
transact(["CREATE INDEX ASYNC idx_status ON entities(tenant_id, status)"])
```

## Pattern 4: Batch Inserts

```python
from safe_query import build, regex, literal, UUID, TENANT_SLUG

inserts = [
    build(
        "INSERT INTO entities (entity_id, tenant_id, name) "
        "VALUES ({eid}, {tid}, {name})",
        eid=regex(row["entity_id"], UUID),
        tid=regex(row["tenant_id"], TENANT_SLUG),
        name=literal(row["name"]),
    )
    for row in rows  # keep each transact call under 3,000 rows
]
transact(inserts)
```

## Pattern 5: Native Foreign Key

```sql
FOREIGN KEY (tenant_id, entity_id)
  REFERENCES entities (tenant_id, entity_id)
```

Use a composite foreign key so the database rejects missing and cross-tenant references. See
[Native Foreign Keys](../../references/foreign-keys.md) for the complete workflow.
