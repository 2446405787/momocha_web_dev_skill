---
name: momocha_web_dev_skill
description: >
  Coding conventions and patterns for the Momocha web project (a game/showcase platform).
  Use this skill whenever writing, editing, or reviewing any code in this project — backend
  Node.js/Express modules, frontend Vue 3 components, API services, types, or config files.
  Always apply these conventions; do not invent new patterns when established ones exist.
---

# Momocha Web Project — Coding Conventions

This project is a game/showcase platform with a Node.js/Express + SQLite backend and a Vue 3 + TypeScript frontend.

---

## Backend (Node.js / Express / SQLite)

### Module Structure

Every feature module lives at `backend/src/modules/{name}/` with exactly 4 files — no more, no less:

```
{name}.controller.js   # HTTP handlers + error code mapping to response
{name}.service.js      # Business logic, validation, event emission
{name}.repository.js   # All SQL queries (Data Access Layer)
{name}.router.js       # Express routes + authGuard middleware mounting
```

The service throws errors; the controller catches them and maps to error codes. The repository only runs SQL — no business logic there.

### Response Format

Every endpoint returns this exact shape:

```javascript
// Success
res.json({ success: true, data: result, error: null })

// Failure
res.json({ success: false, data: null, error: 'ERROR_CODE' })
```

Never return raw error messages to the client — always use an error code string.

### Error Codes

- SCREAMING_SNAKE_CASE: `MAP_NOT_FOUND`, `INSUFFICIENT_BALANCE`, `GARDEN_TILE_OCCUPIED`
- The controller lists expected error codes explicitly; anything else → `SYSTEM_ERROR` + `console.log`
- Errors are thrown in the service layer as `throw new Error('ERROR_CODE')`

### Database (better-sqlite3)

```javascript
db.prepare(sql).get(params)    // single row → object | undefined
db.prepare(sql).all(params)    // multiple rows → array
db.prepare(sql).run(params)    // INSERT / UPDATE / DELETE

// Transactions — wrap with db.transaction, validate BEFORE entering
const tx = db.transaction((uid, data) => {
  db.prepare('UPDATE wallet SET balance = balance - ? WHERE uid = ?').run(data.cost, uid)
  db.prepare('INSERT INTO wallet_transactions ...').run(...)
  return db.prepare('SELECT * FROM ...').get(...)
})
return tx(uid, data)

// Idempotent inserts
db.prepare('INSERT OR IGNORE INTO wallet (uid, balance) VALUES (?, 0)').run(uid)
```

**Rule:** Validate all preconditions (balance checks, ownership checks, existence checks) in the service before calling the transaction. Never validate inside a transaction.

### Global Config

Runtime-configurable key-value store. Read values like:

```javascript
const options = JSON.parse(globalConfigRepository.get('garden.focus.options') ?? '[25,50,90]')
const cost    = parseInt(globalConfigRepository.get('character.generate.coin_cost') ?? '0')
```

Key naming convention: `module.subsection.field` (e.g., `garden.rewards.LIGHT`, `character.breed.inherit.RED`)

### Naming Conventions

| Context | Convention |
|---------|-----------|
| JS variables / functions | camelCase |
| Database columns | snake_case |
| Constants / enums | SCREAMING_SNAKE_CASE |
| Error codes | SCREAMING_SNAKE_CASE |
| Database table names | plural snake_case |
| Config key names | dot.separated.lowercase |

### Utilities

```javascript
// Weighted random — pass an object of {KEY: weight}
const weightedRandom = require('../../utils/weightedRandom')
const result = weightedRandom({ WHITE: 60, GREEN: 25, BLUE: 10, PURPLE: 4, GOLD: 1 })

// Cross-module events
const { eventBus, EVENTS } = require('../../core/eventBus')
eventBus.emit(EVENTS.ITEM_DROPPED, { uid, items, mapId: session.map_id })
```

- Static config (item/map definitions): `backend/src/config/gameConfig.js`
- Enums: `backend/src/constants/game.js`
- Auth guard: imported and mounted per-route in `{name}.router.js`

### Database Migrations

**Never modify `sqlite.js` to add new tables or config.** All incremental schema changes go into `backend/src/core/database/update.js`.

Pattern — append inside the existing `db.exec(...)` block and add `INSERT OR IGNORE` lines for any new global_config defaults:

```javascript
// In update.js — append new tables
db.exec(`
  CREATE TABLE IF NOT EXISTS garden_tiles (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    uid        TEXT NOT NULL,
    tree_def_id TEXT NOT NULL,
    grid_x     INTEGER NOT NULL,
    grid_y     INTEGER NOT NULL,
    planted_at TEXT NOT NULL,
    UNIQUE(uid, grid_x, grid_y)
  );
`);

// In update.js — append new global_config defaults
db.prepare(`INSERT OR IGNORE INTO global_config (key, value) VALUES (?, ?)`).run('garden.focus.options', '[25,50,90]');
```

After editing, run `node backend/src/core/database/update.js` once to apply. Add a `console.log("  OK ...")` line per change following the existing style.

### Pagination Pattern

```javascript
const pageNum  = Math.max(1, parseInt(page)  || 1)
const limitNum = Math.min(100, Math.max(1, parseInt(limit) || 20))
```

---

## Frontend (Vue 3 / TypeScript / Tailwind / Nuxt UI)

### File Organization

```
views/Games/{Name}/{Name}View.vue          # Main game view
views/Games/{Name}/components/*.vue        # Sub-components
src/types/{name}/{name}.ts                 # TypeScript type definitions
src/api/{name}/index.ts                    # API service singleton
```

### TypeScript Response Types

All API responses follow the same shape:

```typescript
interface XxxResponse {
  success: boolean
  data: T | null
  error: string | null
}
```

### API Service Pattern

Export a singleton object (not a class) with typed request methods:

```typescript
import request from '@/utils/request'
import type { ConfigResponse, StartFocusResponse } from '@/types/garden/garden'

export const gardenService = {
  getConfig() {
    return request.get<ConfigResponse>('/garden/config')
  },
  startFocus(durationMinutes: number) {
    return request.post<StartFocusResponse>('/garden/focus/start', { durationMinutes }, { silent: true })
  },
}
// { silent: true } suppresses the automatic error toast notification
```

### Composition API Conventions

```typescript
// State naming
const isLoading   = ref(true)          // booleans: is* prefix
const isSaving    = ref(false)
const maps        = ref<MapInfo[]>([])
const activeTab   = ref('focus')

// Event handlers
async function handleEnter(mapId: number) { ... }   // handle* or on* prefix
async function onTabChange(tab: string) { ... }

// Parallel loading in onMounted
onMounted(async () => {
  try {
    await Promise.all([loadConfig(), loadData()])
  } finally {
    isLoading.value = false              // always close loading in finally
  }
})

// Polling
let pollTimer: ReturnType<typeof setInterval> | null = null
onMounted(() => { pollTimer = setInterval(loadData, 5000) })
onUnmounted(() => { if (pollTimer) clearInterval(pollTimer) })

// Computed for all derived / display values
const totalTrees = computed(() => tiles.value.length)
const dropConfig = computed(() => JSON.parse(props.item.drop_config))
```

### View Structure Template

Every game view follows this layout:

```vue
<template>
  <UContainer class="py-12">
    <!-- 1. Hero section -->
    <section class="mb-8 overflow-hidden rounded-3xl border border-border/60
                    bg-gradient-to-br from-primary/10 via-background to-emerald-500/8 shadow-sm">
      <div class="space-y-5 px-5 py-6 md:px-7 md:py-7">
        <!-- Back button, title, subtitle, stats grid -->
      </div>
    </section>

    <!-- 2. Loading state -->
    <div v-if="isLoading" class="flex justify-center py-24">
      <UIcon name="i-lucide-loader-circle" class="w-8 h-8 text-primary animate-spin" />
    </div>

    <!-- 3. Content -->
    <template v-else>
      <!-- Tab buttons + tab content -->
    </template>
  </UContainer>
</template>
```

### Component Conventions

```typescript
// Props — always typed with generic syntax, never options API
const props = defineProps<{
  item: MapInfo
  isActive: boolean
  backgroundImage?: string
}>()

// Emits — typed payloads
defineEmits<{
  enter:     [mapId: number]
  viewDrops: [mapId: number]
}>()

// Never mutate props directly — emit events instead
```

### UI Library (Nuxt UI)

Use these components — don't build equivalents from scratch:

| Component | Usage |
|-----------|-------|
| `UButton` | All buttons. Props: `size`, `color`, `variant`, `:loading`, `:disabled` |
| `UIcon` | Icons. Always `i-lucide-*` namespace |
| `UModal` | Dialogs. Controlled via `v-model:open` |
| `UContainer` | Page wrapper |
| `UBadge` | Small labels/tags |

**Modal pattern** — always include VisuallyHidden for accessibility:

```vue
<UModal v-model:open="showModal">
  <template #content>
    <VisuallyHidden>
      <DialogTitle>{{ title }}</DialogTitle>
      <DialogDescription>{{ description }}</DialogDescription>
    </VisuallyHidden>
    <div class="flex items-center justify-between px-6 pt-6 pb-4 shrink-0">
      <h3 class="text-lg font-bold text-foreground">{{ title }}</h3>
      <UButton variant="ghost" size="sm" icon="i-lucide-x" @click="showModal = false" />
    </div>
    <div class="overflow-y-auto px-6 pb-6">
      <!-- content -->
    </div>
  </template>
</UModal>
```

### Tailwind Class Conventions

```
rounded-3xl       → major sections / page-level cards
rounded-2xl       → item cards
rounded-xl        → buttons, badges, small elements

border-border/60  → standard border with transparency
text-foreground   → primary text
text-muted-foreground → secondary / label text

bg-card/90        → card overlays
bg-background/80  → lighter overlays

shadow-sm         → subtle depth
shadow-lg         → elevated elements

Gradients: from-primary/10 via-background to-{color}-500/8
```

### Rarity Styling

**Always** use the shared composable — never write custom rarity colors:

```typescript
import { useRarityStyle } from '@/modules/showcase/composables/useRarityStyle'

const { rarityCardClass, rarityTextClass, rarityFilterActiveClass } = useRarityStyle()
```

```vue
<!-- Card border + hover color -->
<div :class="['rounded-2xl border', rarityCardClass(item.rarity)]">

<!-- Text color -->
<span :class="rarityTextClass(item.rarity)">{{ item.nameKey }}</span>

<!-- Filter button active state -->
<button :class="rarityFilterActiveClass(rarity)">{{ rarity }}</button>
```

Rarity tiers (ascending): `WHITE → GREEN → BLUE → PURPLE → GOLD → RED → RED_SPECIAL`

`RED_SPECIAL` automatically gets the rotating golden conic-gradient glow animation (`.rarity-glow-special` defined in `base.css`).

### Asset Config Maps

Centralize asset URLs in a `Record<string, string>` constant at the top of the view, then bind via `name_key`:

```typescript
const TREE_ASSETS_BY_NAME_KEY: Record<string, string> = {
  pine_common:  '/garden/trees/pine_common.png',
  cherry_green: '/garden/trees/cherry_green.png',
}
```

```vue
<img :src="TREE_ASSETS_BY_NAME_KEY[item.tree_def_id]" />
```

### i18n

All user-visible strings go through `$t()`. Never hardcode display text.

```typescript
const { t } = useI18n()

// In template
{{ $t('gardenView.title') }}
{{ $t('gardenView.treeNames.pine_common') }}
{{ $t('gardenView.focus.duration', { minutes: 25 }) }}

// In script
toast.add({ title: t('gardenView.reward.title'), color: 'success' })
```

Key convention: `{viewName}.{section}.{field}` — e.g., `gardenView.focus.start`, `mapView.errors.MAP_NOT_FOUND`
