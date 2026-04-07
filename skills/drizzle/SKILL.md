---
name: drizzle
description: Use when writing, reviewing, or debugging Drizzle ORM code — schema definitions, queries, mutations, relations, migrations, transactions, or any project using drizzle-orm with PostgreSQL, MySQL, SQLite, Turso, Neon, D1, or other supported databases
---

# Drizzle ORM

Lightweight, type-safe TypeScript ORM with SQL-like and relational query APIs. Zero dependencies, serverless-ready.

> **Reference versions:** drizzle-orm **0.45.1**, drizzle-kit **0.31.10** (stable, March 2026). v1.0.0 is in beta (`1.0.0-beta.19`).
> Relations v2 (`defineRelations`) landed in 0.38.x. If working with an older codebase, check which API is in use.
> Run `npm ls drizzle-orm` to verify the project's version before applying patterns from this skill.

## When to Use

- Defining database schemas with `pgTable`, `mysqlTable`, `sqliteTable`
- Writing select/insert/update/delete queries
- Setting up relational queries with `db.query`
- Configuring `drizzle-kit` for migrations
- Debugging type errors or query issues in Drizzle code

**Not for:** General SQL questions unrelated to Drizzle, other ORMs (Prisma, TypeORM, Sequelize).

## Connection Setup

```typescript
// PostgreSQL (node-postgres) — simplest form
import { drizzle } from "drizzle-orm/node-postgres";
const db = drizzle(process.env.DATABASE_URL!);

// PostgreSQL with pool
import { Pool } from "pg";
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle({ client: pool });

// PostgreSQL (postgres.js)
import { drizzle } from "drizzle-orm/postgres-js";
import postgres from "postgres";
const db = drizzle({ client: postgres(process.env.DATABASE_URL!) });

// Neon (serverless)
import { drizzle } from "drizzle-orm/neon-http";
const db = drizzle(process.env.DATABASE_URL!);

// SQLite (libsql / Turso)
import { drizzle } from "drizzle-orm/libsql";
const db = drizzle(process.env.DATABASE_URL!);
// or with auth token:
const db = drizzle({ connection: { url: "...", authToken: "..." } });

// Cloudflare D1
import { drizzle } from "drizzle-orm/d1";
const db = drizzle({ connection: env.DB });

// MySQL (mysql2)
import { drizzle } from "drizzle-orm/mysql2";
const db = drizzle(process.env.DATABASE_URL!);
```

**With relations (required for `db.query`):**
```typescript
import { relations } from "./relations";
const db = drizzle(process.env.DATABASE_URL!, { relations });
```

## Schema Definition

See @api-reference.md for full column types per dialect.

```typescript
import { pgTable, serial, text, integer, varchar, timestamp, boolean, uuid, check } from "drizzle-orm/pg-core";
import { sql } from "drizzle-orm";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  email: varchar("email", { length: 256 }).notNull().unique(),
  age: integer("age"),
  verified: boolean("verified").default(false),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// MySQL: import from "drizzle-orm/mysql-core", use mysqlTable, int (not integer)
// SQLite: import from "drizzle-orm/sqlite-core", use sqliteTable, integer, text only
```

**Foreign keys:**
```typescript
export const posts = pgTable("posts", {
  id: serial("id").primaryKey(),
  title: varchar("title", { length: 256 }),
  content: text("content").notNull(),
  authorId: integer("author_id").references(() => users.id),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

**Indexes, constraints, checks** (third arg):
```typescript
import { index, unique, check } from "drizzle-orm/pg-core";

export const example = pgTable("example", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  age: integer("age"),
}, (table) => [
  index("name_idx").on(table.name),
  unique().on(table.name),
  check("age_check", sql`${table.age} > 0`),
]);
```

**Type inference:**
```typescript
type User = typeof users.$inferSelect;    // select result type
type NewUser = typeof users.$inferInsert;  // insert input type
// also: InferSelectModel<typeof users>, InferInsertModel<typeof users>
```

## Relations

### v2 (current — `defineRelations`)
```typescript
import { defineRelations } from "drizzle-orm";
import * as schema from "./schema";

export const relations = defineRelations(schema, (r) => ({
  users: {
    posts: r.many.posts(),
    invitee: r.one.users({
      from: r.users.invitedBy,
      to: r.users.id,
    }),
  },
  posts: {
    author: r.one.users({
      from: r.posts.authorId,
      to: r.users.id,
    }),
    comments: r.many.comments(),
  },
}));

// Many-to-many via junction table:
// groups: r.many.groups({
//   from: r.users.id.through(r.usersToGroups.userId),
//   to: r.groups.id.through(r.usersToGroups.groupId),
// }),
```

### v1 (legacy — `relations()`)
```typescript
import { relations } from "drizzle-orm";
export const usersRelations = relations(users, ({ one, many }) => ({
  posts: many(posts),
}));
export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));
```

## Queries (SQL-like API)

```typescript
import { eq, and, or, gt, gte, lt, lte, ne, like, ilike, inArray, between, isNull, sql, asc, desc } from "drizzle-orm";

// Select all
const allUsers = await db.select().from(users);

// Partial select
const names = await db.select({ id: users.id, name: users.name }).from(users);

// Filters
await db.select().from(users).where(eq(users.id, 1));
await db.select().from(users).where(and(gte(users.age, 18), eq(users.verified, true)));
await db.select().from(users).where(or(eq(users.name, "Alice"), eq(users.name, "Bob")));

// Pagination & sorting
await db.select().from(users).orderBy(desc(users.createdAt)).limit(10).offset(20);

// Distinct
await db.selectDistinct({ name: users.name }).from(users);

// Aggregations
await db.select({
  age: users.age,
  count: sql<number>`cast(count(${users.id}) as int)`,
}).from(users).groupBy(users.age);

// Custom SQL in select
await db.select({
  id: users.id,
  lower: sql<string>`lower(${users.name})`,
}).from(users);
```

## Relational Queries (`db.query`)

Requires relations passed to `drizzle()`.

```typescript
// Find many with nested relations
const result = await db.query.users.findMany({
  columns: { id: true, name: true },
  with: {
    posts: {
      columns: { title: true },
      limit: 5,
      with: { comments: { limit: 3 } },
    },
  },
  where: { verified: { eq: true } },
  orderBy: { createdAt: "desc" },
  limit: 10,
});

// Find first
const user = await db.query.users.findFirst({
  where: { id: { eq: 1 } },
  with: { posts: true },
});

// v2 where clause operators
where: {
  OR: [{ name: { like: "A%" } }, { name: { like: "B%" } }],
  AND: [{ age: { gte: 18 } }],
  NOT: { verified: { eq: false } },
  RAW: (table) => sql`${table.age} BETWEEN 25 AND 35`,
  // filter by column: eq, ne, gt, gte, lt, lte, in, notIn, like, ilike, isNull, isNotNull
}
```

## Joins

```typescript
// Inner join
await db.select().from(users)
  .innerJoin(posts, eq(users.id, posts.authorId));

// Left join
await db.select().from(users)
  .leftJoin(posts, eq(users.id, posts.authorId));

// Right join, full join also available
// .rightJoin(...), .fullJoin(...)

// Left join lateral (correlated subquery — PG only)
const sub = db.select().from(pets).where(gte(users.age, 16)).as("userPets");
await db.select().from(users).leftJoinLateral(sub, sql`true`);

// Subquery in join
const sq = db.select().from(users).where(eq(users.id, 42)).as("sq");
await db.select().from(users).leftJoin(sq, eq(users.id, sq.id));

// CTE (WITH clause)
const sq = db.$with("sq").as(db.select().from(users).where(eq(users.id, 42)));
await db.with(sq).select().from(sq);
```

## Mutations

```typescript
// Insert
await db.insert(users).values({ name: "Alice", email: "alice@example.com" });
await db.insert(users).values([{ name: "A", email: "a@a.com" }, { name: "B", email: "b@b.com" }]);

// Insert with returning (PG/SQLite)
const [newUser] = await db.insert(users).values({ name: "Alice", email: "a@a.com" }).returning();

// Update
await db.update(users).set({ name: "Bob" }).where(eq(users.id, 1));
await db.update(users).set({ name: "Bob" }).where(eq(users.id, 1)).returning();

// Delete
await db.delete(users).where(eq(users.id, 1));
await db.delete(users).where(eq(users.id, 1)).returning();

// Upsert — PG/SQLite
await db.insert(users)
  .values({ id: 1, name: "Alice", email: "alice@a.com" })
  .onConflictDoNothing({ target: users.email });

await db.insert(users)
  .values({ id: 1, name: "Alice", email: "alice@a.com" })
  .onConflictDoUpdate({
    target: users.email,
    set: { name: "Alice Updated" },
  });

// Upsert — MySQL
await db.insert(users)
  .values({ id: 1, name: "Alice", email: "alice@a.com" })
  .onDuplicateKeyUpdate({ set: { name: "Alice Updated" } });
```

## Transactions

```typescript
// Basic
await db.transaction(async (tx) => {
  await tx.update(accounts).set({ balance: sql`${accounts.balance} - 100` }).where(eq(accounts.id, 1));
  await tx.update(accounts).set({ balance: sql`${accounts.balance} + 100` }).where(eq(accounts.id, 2));
});

// With rollback
await db.transaction(async (tx) => {
  const [account] = await tx.select().from(accounts).where(eq(accounts.id, 1));
  if (account.balance < 100) {
    tx.rollback();
    return;
  }
  await tx.update(accounts).set({ balance: sql`${accounts.balance} - 100` }).where(eq(accounts.id, 1));
});

// Return value from transaction
const balance = await db.transaction(async (tx) => {
  await tx.update(accounts).set({ balance: sql`${accounts.balance} - 100` }).where(eq(accounts.id, 1));
  const [acc] = await tx.select().from(accounts).where(eq(accounts.id, 1));
  return acc.balance;
});

// Nested (savepoints)
await db.transaction(async (tx) => {
  await tx.insert(users).values({ name: "User 1", email: "u1@test.com" });
  await tx.transaction(async (tx2) => {
    await tx2.insert(users).values({ name: "User 2", email: "u2@test.com" });
  });
});

// Isolation level (PG)
await db.transaction(async (tx) => { /* ... */ }, {
  isolationLevel: "serializable",   // "read committed" | "repeatable read" | "serializable"
  accessMode: "read write",
});
```

## Prepared Statements

```typescript
const getUser = db.select().from(users)
  .where(eq(users.id, sql.placeholder("id")))
  .prepare("getUser");

await getUser.execute({ id: 10 });
await getUser.execute({ id: 42 });
```

## Batch API (LibSQL, Neon, D1)

```typescript
const results = await db.batch([
  db.insert(users).values({ id: 1, name: "Alice" }).returning({ id: users.id }),
  db.update(users).set({ name: "Bob" }).where(eq(users.id, 1)),
  db.select().from(users),
]);
```

## Dynamic Queries

```typescript
import { SQL } from "drizzle-orm";

const filters: SQL[] = [];
if (search) filters.push(ilike(posts.title, `%${search}%`));
if (category) filters.push(inArray(posts.category, category));
if (minViews) filters.push(gt(posts.views, minViews));

await db.select().from(posts).where(and(...filters));
```

## Drizzle Kit (Migrations & Tooling)

**`drizzle.config.ts`:**
```typescript
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "postgresql",       // "mysql" | "sqlite" | "turso"
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  // optional:
  // migrations: { table: "journal", schema: "drizzle" },
});
```

**CLI commands:**
```bash
drizzle-kit generate   # Generate migration SQL from schema changes
drizzle-kit migrate    # Apply pending migrations
drizzle-kit push       # Push schema directly (no migration files — dev only)
drizzle-kit pull       # Introspect existing DB → generate schema
drizzle-kit studio     # Open visual DB browser
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `db.query.users` returns undefined | Pass `{ relations }` to `drizzle()` |
| `.returning()` not working on MySQL | MySQL doesn't support RETURNING — use separate select |
| Forgetting `.notNull()` | Columns are nullable by default in Drizzle |
| Using `relations()` v1 with v2 `where` syntax | v2 uses `defineRelations`, v1 uses `relations()` from `drizzle-orm` — don't mix |
| `onConflictDoUpdate` on MySQL | MySQL uses `onDuplicateKeyUpdate` instead |
| Missing `as("alias")` on subqueries | Subqueries in joins/CTEs require `.as("name")` |
| `serial` vs `integer` for auto-increment | PG: `serial()`. MySQL: `int().primaryKey().autoincrement()`. SQLite: `integer().primaryKey({ autoIncrement: true })` |
| Using v2 relations on older drizzle-orm | `defineRelations` requires 0.38.x+. Older projects use `relations()` from `drizzle-orm` (v1 API) |
| v2 `where` object syntax in `db.query` | Object-style `where: { col: { eq: val } }` is v2 only (0.38.x+). v1 uses callback: `where: (t, { eq }) => eq(t.col, val)` |
| Batch API unavailable | `db.batch()` only works with LibSQL, Neon HTTP, and D1 drivers — not node-postgres or mysql2 |
