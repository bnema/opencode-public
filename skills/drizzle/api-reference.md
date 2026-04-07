# Drizzle ORM API Reference

> **Documented against:** drizzle-orm **0.45.1**, drizzle-kit **0.31.10** (stable, March 2026).
> APIs may change in v1.0.0 (currently beta). Always verify against the project's installed version.

## Column Types by Dialect

### PostgreSQL (`drizzle-orm/pg-core`)

| Type | Usage | Notes |
|------|-------|-------|
| `serial` | `serial("id")` | Auto-increment int4 |
| `bigserial` | `bigserial("id", { mode: "number" })` | Auto-increment int8 |
| `integer` | `integer("age")` | int4 |
| `bigint` | `bigint("n", { mode: "number" })` | int8 |
| `smallint` | `smallint("n")` | int2 |
| `real` | `real("n")` | float4 |
| `doublePrecision` | `doublePrecision("n")` | float8 |
| `numeric` | `numeric("price", { precision: 10, scale: 2 })` | Exact decimal |
| `text` | `text("bio")` | Unlimited text |
| `varchar` | `varchar("name", { length: 256 })` | Variable length |
| `char` | `char("code", { length: 3 })` | Fixed length |
| `boolean` | `boolean("active")` | true/false |
| `timestamp` | `timestamp("ts")` | Without timezone |
| `timestamp` | `timestamp("ts", { withTimezone: true })` | With timezone |
| `date` | `date("d")` | Date only |
| `time` | `time("t")` | Time only |
| `interval` | `interval("dur")` | Time interval |
| `uuid` | `uuid("id").defaultRandom()` | UUID v4 |
| `json` | `json("data")` | JSON text |
| `jsonb` | `jsonb("data")` | Binary JSON |
| `pgEnum` | `pgEnum("status", ["active", "inactive"])` | Enum type |

### MySQL (`drizzle-orm/mysql-core`)

| Type | Usage | Notes |
|------|-------|-------|
| `int` | `int("id")` | 4-byte int |
| `bigint` | `bigint("n", { mode: "number" })` | 8-byte int |
| `tinyint` | `tinyint("flag")` | 1-byte int |
| `smallint` | `smallint("n")` | 2-byte int |
| `mediumint` | `mediumint("n")` | 3-byte int |
| `float` | `float("n")` | Single precision |
| `double` | `double("n")` | Double precision |
| `decimal` | `decimal("price", { precision: 10, scale: 2 })` | Exact decimal |
| `text` | `text("bio")` | Text |
| `varchar` | `varchar("name", { length: 256 })` | Variable length |
| `char` | `char("code", { length: 3 })` | Fixed length |
| `boolean` | `boolean("active")` | tinyint(1) |
| `timestamp` | `timestamp("ts")` | Timestamp |
| `datetime` | `datetime("dt")` | Datetime |
| `date` | `date("d")` | Date |
| `time` | `time("t")` | Time |
| `json` | `json("data")` | JSON |
| `mysqlEnum` | `mysqlEnum("role", ["admin", "user"])` | Enum |

### SQLite (`drizzle-orm/sqlite-core`)

| Type | Usage | Notes |
|------|-------|-------|
| `integer` | `integer("id")` | Integer (any size) |
| `real` | `real("n")` | Floating point |
| `text` | `text("name")` | Text |
| `blob` | `blob("data")` | Binary data |
| `numeric` | `numeric("n")` | Numeric affinity |

**SQLite enum pattern** (no native enum):
```typescript
text("role", { enum: ["admin", "user", "guest"] })
```

## Column Modifiers

All dialects support these chainable modifiers:

```typescript
.primaryKey()           // PRIMARY KEY
.notNull()              // NOT NULL
.default(value)         // DEFAULT value
.defaultNow()           // DEFAULT now() / CURRENT_TIMESTAMP
.unique()               // UNIQUE constraint
.unique("custom_name")  // Named unique
.references(() => t.id) // FOREIGN KEY
.$defaultFn(() => ...)  // JS-side default (runs on insert)
.$onUpdateFn(() => ...) // JS-side update default
```

**PG-specific:**
```typescript
.defaultRandom()        // gen_random_uuid() — for uuid columns
```

**MySQL-specific:**
```typescript
.autoincrement()        // AUTO_INCREMENT
```

**SQLite-specific:**
```typescript
.primaryKey({ autoIncrement: true })  // AUTOINCREMENT
```

## Filter Operators

All imported from `"drizzle-orm"`:

| Operator | Usage | SQL |
|----------|-------|-----|
| `eq` | `eq(col, val)` | `col = val` |
| `ne` | `ne(col, val)` | `col <> val` |
| `gt` | `gt(col, val)` | `col > val` |
| `gte` | `gte(col, val)` | `col >= val` |
| `lt` | `lt(col, val)` | `col < val` |
| `lte` | `lte(col, val)` | `col <= val` |
| `like` | `like(col, "%pat%")` | `col LIKE '%pat%'` |
| `ilike` | `ilike(col, "%pat%")` | `col ILIKE '%pat%'` (PG) |
| `notLike` | `notLike(col, pat)` | `col NOT LIKE pat` |
| `inArray` | `inArray(col, [1,2])` | `col IN (1, 2)` |
| `notInArray` | `notInArray(col, [1,2])` | `col NOT IN (1, 2)` |
| `between` | `between(col, 1, 10)` | `col BETWEEN 1 AND 10` |
| `isNull` | `isNull(col)` | `col IS NULL` |
| `isNotNull` | `isNotNull(col)` | `col IS NOT NULL` |
| `exists` | `exists(subquery)` | `EXISTS (subquery)` |
| `and` | `and(cond1, cond2)` | `cond1 AND cond2` |
| `or` | `or(cond1, cond2)` | `cond1 OR cond2` |
| `not` | `not(cond)` | `NOT cond` |
| `sql` | `` sql`custom` `` | Raw SQL expression |

## Relational Query v2 `where` Operators

Used inside `db.query.*.findMany({ where: { ... } })`:

```typescript
where: {
  // Logical
  OR: [condition1, condition2],
  AND: [condition1, condition2],
  NOT: { ... },
  RAW: (table) => sql`...`,

  // Column filters
  columnName: {
    eq: value, ne: value,
    gt: value, gte: value, lt: value, lte: value,
    in: [values], notIn: [values],
    like: "pattern", ilike: "pattern",
    notLike: "pattern", notIlike: "pattern",
    isNull: true, isNotNull: true,
    // PG array operators:
    arrayOverlaps: [values],
    arrayContains: [values],
    arrayContained: [values],
  },

  // Filter by relation existence
  relationName: { /* nested where */ },
}
```

## Set Operations

```typescript
import { union, unionAll, intersect, intersectAll, except, exceptAll } from "drizzle-orm/pg-core";

union(db.select().from(users), db.select().from(customers));
unionAll(...);
intersect(...);
except(...);
```

## Raw SQL (`sql` template tag)

```typescript
import { sql } from "drizzle-orm";

// In select
db.select({ lower: sql<string>`lower(${users.name})` }).from(users);

// In where
db.select().from(users).where(sql`${users.age} > 18`);

// Placeholder for prepared statements
sql.placeholder("id")

// Raw string (CAUTION: no escaping)
sql.raw("NOW()")

// Empty SQL
sql.empty()
```

## Views

```typescript
// PostgreSQL
import { pgView } from "drizzle-orm/pg-core";

export const activeUsers = pgView("active_users").as((qb) =>
  qb.select().from(users).where(eq(users.verified, true))
);

// Then query:
await db.select().from(activeUsers);

// Materialized view (PG only)
import { pgMaterializedView } from "drizzle-orm/pg-core";
export const stats = pgMaterializedView("stats").as((qb) =>
  qb.select({ count: sql`count(*)` }).from(users)
);
```
