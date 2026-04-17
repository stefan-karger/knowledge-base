# SaaS Billing & Access Control Architecture

> Stack: `TanStack Start` + `SolidJS` + `Drizzle` (`Postgres`) + `better-auth` (`Admin` / `Organization` / `Polar`)

## Table of Contents

- [Overview & Goals](#overview--goals)
- [Prerequisites & Assumptions](#prerequisites--assumptions)
- [Core Architecture Decisions](#core-architecture-decisions)
- [Database Schema (Drizzle)](#database-schema-drizzle)
- [better-auth Integration](#better-auth-integration)
- [Server Layer (TanStack Start)](#server-layer-tanstack-start)
- [Plan Resolution Logic](#plan-resolution-logic)
- [Overrides (Admin Control)](#overrides-admin-control)
- [Polar Billing Integration](#polar-billing-integration)
- [Feature Gating & Permissions](#feature-gating--permissions)
- [Limits & Quotas (Locations Example)](#limits--quotas-locations-example)
- [TanStack Query Integration](#tanstack-query-integration)
- [Implementation Roadmap](#implementation-roadmap)
- [Final Result](#final-result)

## Overview & Goals

This system enables:

- User-based subscription tiers: `Free`, `Pro`, `Max`
- Admin overrides, including temporary or permanent changes
- `Polar` as the billing source of truth
- Organizations as access control only in v1, not billing owners
- Plan-based feature gating and limits such as `locations`

### Key Principle

```text
Admin > Override > PlanAssignment > Default (Free)
```

## Prerequisites & Assumptions

### Required Stack

- `TanStack Start`
- `SolidJS`
- `Drizzle ORM`
- `better-auth`
- `PostgreSQL`

### Required better-auth Plugins

- `Admin` plugin for roles such as `admin`
- `Organization` plugin for orgs and membership
- `Polar` plugin for billing and webhooks

### Assumptions

- `user.id` is a string, using the default `better-auth` user model
- Organization membership is handled by `better-auth`
- All business logic runs through `TanStack Start` server functions

## Core Architecture Decisions

### Separation of Concerns

| Concern         | Table / System             | Managed By       |
| --------------- | -------------------------- | ---------------- |
| Billing state   | `plan_assignment`          | `Polar` / system |
| Manual override | `plan_override`            | Admin            |
| Access control  | `better-auth` organization | `better-auth`    |
| Permissions     | App logic                  | App              |

### Plan Ownership

- A plan belongs to the user
- An organization is only for grouping and access control
- Mixed plans inside one organization are allowed

### Why Not Merge Override Into Plan?

Because:

- Webhooks would overwrite manual changes
- Temporary upgrades become messy
- Auditability gets worse

## Database Schema (Drizzle)

### Concept

Use two layers:

- `plan_assignment` for canonical billing state
- `plan_override` for manual override state

### Full Schema

```ts
import {
  pgTable,
  pgEnum,
  uuid,
  text,
  timestamp,
  index,
  uniqueIndex,
  integer,
} from "drizzle-orm/pg-core";
import { sql } from "drizzle-orm";

export const subjectTypeEnum = pgEnum("subject_type", ["user", "organization"]);
export const planTierEnum = pgEnum("plan_tier", ["free", "pro", "max"]);
export const planSourceEnum = pgEnum("plan_source", ["system", "polar"]);

export const planAssignment = pgTable(
  "plan_assignment",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    subjectType: subjectTypeEnum("subject_type").notNull(),
    subjectId: text("subject_id").notNull(),
    tier: planTierEnum("tier").notNull().default("free"),
    source: planSourceEnum("source").notNull().default("system"),
    polarProductId: text("polar_product_id"),
    polarSubscriptionId: text("polar_subscription_id"),
    status: text("status"),
    effectiveFrom: timestamp("effective_from", { withTimezone: true })
      .notNull()
      .defaultNow(),
    effectiveTo: timestamp("effective_to", { withTimezone: true }),
    reason: text("reason"),
    createdAt: timestamp("created_at", { withTimezone: true })
      .notNull()
      .defaultNow(),
  },
  (t) => ({
    currentOnlyUx: uniqueIndex("plan_assignment_current_only_ux")
      .on(t.subjectType, t.subjectId)
      .where(sql`${t.effectiveTo} is null`),
  }),
);

export const planOverride = pgTable("plan_override", {
  id: uuid("id").primaryKey().defaultRandom(),
  subjectType: subjectTypeEnum("subject_type").notNull(),
  subjectId: text("subject_id").notNull(),
  tier: planTierEnum("tier").notNull(),
  startsAt: timestamp("starts_at", { withTimezone: true })
    .notNull()
    .defaultNow(),
  endsAt: timestamp("ends_at", { withTimezone: true }),
  revokedAt: timestamp("revoked_at", { withTimezone: true }),
  createdByUserId: text("created_by_user_id").notNull(),
});
```

## better-auth Integration

### Goal

Ensure every new user starts on the `free` plan.

### Hook: User Creation

```ts
databaseHooks: {
  user: {
    create: {
      after: async (user) => {
        await db
          .insert(planAssignment)
          .values({
            subjectType: "user",
            subjectId: user.id,
            tier: "free",
            source: "system",
            reason: "initial_free",
          })
          .onConflictDoNothing();
      },
    },
  },
}
```

## Server Layer (TanStack Start)

### Core Idea

All business logic lives in server functions.

### Example: Set Current Plan

```ts
export async function setCurrentPlan(input) {
  await db
    .update(planAssignment)
    .set({ effectiveTo: sql`now()` })
    .where(/* current */);

  await db.insert(planAssignment).values(input);
}
```

### Full Implementation

```ts
export async function setCurrentPlan(input: {
  subjectType: "user";
  subjectId: string;
  tier: "free" | "pro" | "max";
  source: "system" | "polar";
}) {
  await db
    .update(planAssignment)
    .set({ effectiveTo: sql`now()` })
    .where(
      and(
        eq(planAssignment.subjectType, input.subjectType),
        eq(planAssignment.subjectId, input.subjectId),
        isNull(planAssignment.effectiveTo),
      ),
    );

  await db.insert(planAssignment).values(input);
}
```

## Plan Resolution Logic

### Core Logic

```text
Admin -> Override -> Plan -> Free
```

### Implementation

```ts
export async function getEffectiveTierForUser(userId: string) {
  if (await isAdmin(userId)) return "max";

  const override = await getOverride(userId);
  if (override) return override.tier;

  const plan = await getPlan(userId);
  return plan?.tier ?? "free";
}
```

## Overrides (Admin Control)

### Purpose

- Temporary upgrade
- Compensation
- Internal users

### Example

```ts
await createOverride({
  subjectId: userId,
  tier: "max",
  endsAt: new Date(Date.now() + 7 * 86400000),
});
```

### Full Implementation

```ts
export async function createOverride(input) {
  await db.insert(planOverride).values(input);
}
```

## Polar Billing Integration

### Flow

1. User buys a plan
2. `Polar` webhook fires
3. App updates `plan_assignment`

### Hook Example

```ts
onSubscriptionActive: async (payload) => {
  await syncTier(payload);
};
```

### Sync Logic (Simplified)

```ts
export async function syncTier(payload) {
  const tier = mapProductToTier(payload.productId);

  await setCurrentPlan({
    subjectType: "user",
    subjectId: payload.userId,
    tier,
    source: "polar",
  });
}
```

## Feature Gating & Permissions

### Example: Invite Permission

```ts
if (tier === "free") throw new Error("UPGRADE_REQUIRED");
```

### Combined With Organization Role

```ts
if (!isOrgAdmin || tier === "free") deny();
```

## Limits & Quotas (Locations Example)

### Strategy

- UI check for UX
- Server check for enforcement

### Simple Count Approach

```ts
if (count >= limit) throw Error("LIMIT");
```

### Full Implementation

```ts
export const createLocationFn = createServerFn().handler(async ({ data }) => {
  const tier = await getEffectiveTierForUser(userId);
  const limit = tier === "free" ? 10 : tier === "pro" ? 100 : Infinity;

  const count = await countLocations(userId);

  if (count >= limit) throw new Error("LIMIT");

  return db.insert(locations).values(data);
});
```

### Advanced (Race-Safe) Counter

```sql
SELECT used FROM usage_counter FOR UPDATE;
```

This prevents double-insert issues under concurrency.

## TanStack Query Integration

### Pattern

```ts
queryOptions({
  queryKey: [...],
  queryFn: ...,
})
```

### Example

```ts
export const canCreateLocationQuery = (orgId) =>
  queryOptions({
    queryKey: ["locations", "canCreate", orgId],
    queryFn: () => canCreateLocationFn({ data: { orgId } }),
  });
```

## Implementation Roadmap

### Phase 1: Foundation

- DB schema
- Migrations
- `better-auth` hook for default free plan

### Phase 2: Core Logic

- Plan resolution helper
- Override logic
- Server functions

### Phase 3: Billing

- `Polar` setup
- Webhook sync

### Phase 4: Limits

- Count logic
- Server enforcement
- UI integration

### Phase 5: Admin Tools

- Override UI
- Plan history UI

## Final Result

You now have:

- Clean separation of billing vs. overrides
- Scalable architecture
- Production-safe limits
- Extendable org model
- Minimal coupling to `Polar`
