# core-activity -- CLAUDE.md

User-facing activity feed and timeline system. Records events from across the app and exposes them as public, user, org, and entity-scoped feeds.

## Module Boundary

**core-activity owns:** `activity_events` schema, activity router, `recordActivity()` service, DI deps, components, styles, seed.

**Project owns:** `config/deps/activity-deps.ts`, `app/dashboard/(panel)/activity/page.tsx`, dashboard widget integration in `config/dashboard-widgets.tsx`.

## DI (`setActivityDeps()`)

- `resolveOrgId` -- validates user belongs to org (from `@/server/lib/resolve-org`)

## Schema

Single table `activity_events` (source: `schema/`): actor (user/system), dotted action string, polymorphic target (type + id + label), JSONB metadata, org scope, public flag.

## `recordActivity()` API

`recordActivity()` from `@/core-activity/lib/activity-service` is **fire-and-forget** -- never throws, logs errors via logger. Use `actorType: 'system'` (no `actorId`) for system events.

```ts
await recordActivity({
  actorId: userId,
  action: 'comment.created',
  targetType: 'post',
  targetId: postId,
  targetLabel: 'My Blog Post',
  organizationId: orgId,
  isPublic: true,
});
```

## Router Endpoints

Live list: `routers/activity.ts` (JSDoc per procedure → `src/generated/procedure-docs.ts` / MCP `search_tools`). Non-obvious: `orgFeed` validates membership via injected `resolveOrgId`; `adminFeed` is offset-paginated, all other feeds are cursor-paginated.

## Components

Live list: `components/` (JSDoc'd props; map with `bun run indigo map core-activity`). Non-obvious: `DashboardActivityWidget` is self-contained (fetches 8 recent events via `activity.adminFeed`) and is already wired in `src/config/dashboard-widgets.tsx` as the `'activity-feed'` widget; `ActivityItem` maps action prefixes to icons/labels in-code, falling back to the last segment of the dotted action for unknown actions.

## Styles

`styles/activity.css` -- prefixed `.activity-*` classes inside `@layer components`. Dark mode via `:root.dark` overrides on icon backgrounds.

## Seed

`seed/index.ts` -- `seedActivity(db, superadminUserId)` demo events, `hasActivityData(db)` skip check. Deterministic UUIDs (`00000000-0000-4000-c200-*`) + `onConflictDoNothing()` for idempotency.

## Admin Page

`_templates/app/dashboard/(panel)/activity/page.tsx` -- full admin activity log at `/dashboard/activity` (stat cards, category tabs, filter bar, paginated table).

## Wiring

1. Create `config/deps/activity-deps.ts` (scaffolded from `_templates/`)
2. Import in `server.ts` as side-effect
3. Routers auto-registered via `indigo:sync`
4. Call `recordActivity()` from any module to log user-facing events
5. Admin page at `/dashboard/activity` (scaffolded from `_templates/`)
6. Optionally integrate the dashboard widget (see Components)
