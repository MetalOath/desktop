# Zen Browser — Sidebar Order API Specification

**Status:** Proposed
**Affects:** `browser.tabs`, `browser.tabGroups`, `browser.zenWorkspaces`
**Required permission:** `zenWorkspaces` (existing)

---

## Problem

Firefox enforces that pinned tabs always occupy the lowest indices in the tab list. `browser.tabs.move()` respects this constraint and silently clamps a pinned tab's position to the pinned-tab region. As a result, extensions cannot produce a sidebar layout where pinned tabs and Zen Folders are arbitrarily interleaved — any call to `tabs.update({ pinned: true })` or `tabs.move()` on a pinned tab hoists it above all non-pinned content regardless of the requested index.

Zen Browser's sidebar already renders workspaces independently of Firefox's internal tab ordering, so it is well-positioned to expose a richer positioning model to extensions.

---

## Goals

- Allow any tab (pinned, essential, or normal) to be placed at any sidebar position within its workspace.
- Allow Zen Folders to be placed at any sidebar position within their workspace.
- Support atomic batch reordering so extensions can reconstruct an exact saved layout in a single operation without intermediate visual flicker or event cascades.
- Read back the current sidebar order so extensions can persist it.
- Remain backwards-compatible: existing calls to `browser.tabs.move()` and `browser.tabGroups` continue to work unchanged.

---

## New permission

No new permission is required. The APIs below are gated behind the existing `zenWorkspaces` permission, since sidebar ordering is inherently workspace-scoped.

---

## API 1 — `browser.tabs.move()` extension: `zenIndex`

### Motivation

`browser.tabs.move()` today accepts `{ windowId?, index }`. Adding an optional `zenIndex` property instructs Zen to place the tab at that position in the workspace sidebar, **without hoisting it to the pinned region**. If `zenIndex` is supplied, `index` is ignored for the purposes of sidebar display.

### Signature

```js
browser.tabs.move(tabIds, { zenIndex: number, zenWorkspaceId?: string }) → Tab[]
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `tabIds` | `number \| number[]` | Tab ID(s) to move. |
| `zenIndex` | `number` | Zero-based target position in the workspace sidebar. 0 = top. Out-of-range values are clamped. |
| `zenWorkspaceId` | `string?` | If supplied, reassign the tab(s) to this workspace and place at `zenIndex` within it. If omitted, tabs stay in their current workspace. |

### Behaviour

- Pinned tabs, normal tabs, and essential tabs can all be moved with `zenIndex`. The pinned/essential state is not changed.
- Moving a tab with `zenIndex` does **not** fire `tabs.onMoved` (which carries Firefox's internal index semantics). Instead it fires `tabs.onZenMoved` (see **Events** below).
- If `zenWorkspaceId` is supplied and the tab is essential (`tab.essential === true`), the call throws `ExtensionError` — essential tabs are workspace-independent and have no workspace position.
- Moving a tab into a position occupied by a Zen Folder inserts it immediately before that folder (the folder shifts down one position).

### Example

```js
// Move a pinned tab to position 3 in the active workspace,
// after two Zen Folders that occupy positions 0 and 1-2.
const ws = await browser.zenWorkspaces.getActive();
await browser.tabs.move(pinnedTab.id, {
  zenIndex: 3,
  zenWorkspaceId: ws.id,
});
```

---

## API 2 — `browser.tabGroups.move()`

### Motivation

There is currently no API to reposition a Zen Folder or tab group within the sidebar. `browser.tabs.move()` operates on individual tabs, not on groups as a unit.

### Signature

```js
browser.tabGroups.move(groupId, { zenIndex: number }) → TabGroup
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `groupId` | `number` | ID of the tab group or Zen Folder to move. |
| `zenIndex` | `number` | Zero-based target position in the workspace sidebar for the **start** of the group. All member tabs move together as a unit. Out-of-range values are clamped. |

### Behaviour

- All member tabs of the group move together atomically. The relative order of tabs within the group is preserved.
- Fires `tabGroups.onMoved` (new event, see **Events** below) once for the group.
- Does not fire `tabs.onMoved` for individual member tabs.
- If the target position falls inside another group, the group is inserted immediately before that group.

### Example

```js
// Move a Zen Folder to the top of the workspace sidebar.
await browser.tabGroups.move(folderId, { zenIndex: 0 });
```

---

## API 3 — `browser.zenWorkspaces.reorder()`

### Motivation

Restoring a saved tab layout requires multiple individual `move()` calls, each of which fires events and may trigger intermediate renders. A batch reorder API allows extensions to describe the complete desired order in one call, applied atomically.

### Signature

```js
browser.zenWorkspaces.reorder(workspaceId, items) → void
```

```ts
type OrderItem =
  | { type: 'tab',   id: number }
  | { type: 'group', id: number }
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `workspaceId` | `string` | UUID of the workspace to reorder. |
| `items` | `OrderItem[]` | Complete desired order of tabs and groups in the workspace sidebar, from top (index 0) to bottom. |

### Behaviour

- The `items` array must contain every non-essential tab and every group currently belonging to `workspaceId`. If any item is missing, the call throws `ExtensionError: incomplete order — missing N items`.
- Essential tabs are workspace-independent and must **not** appear in `items`. If an essential tab ID is included, the call throws `ExtensionError`.
- Tabs or groups belonging to a different workspace must not appear in `items`. Throws `ExtensionError` if they do.
- The reorder is applied atomically — no intermediate `onMoved` events fire during the operation. A single `zenWorkspaces.onReordered` event fires when complete (see **Events** below).
- If the call fails partway through (e.g. a tab was closed between the call being issued and it being applied), the workspace order is left unchanged and an error is thrown.

### Example

```js
// Reconstruct a saved layout: pinned tab, then a folder, then another
// pinned tab, then normal tabs — in one atomic call.
const ws = await browser.zenWorkspaces.getActive();

await browser.zenWorkspaces.reorder(ws.id, [
  { type: 'tab',   id: pinnedTabA.id },
  { type: 'group', id: aiFolder.id   },
  { type: 'tab',   id: pinnedTabB.id },
  { type: 'tab',   id: normalTab1.id },
  { type: 'tab',   id: normalTab2.id },
]);
```

---

## API 4 — `browser.zenWorkspaces.getSidebarOrder()`

### Motivation

Extensions need to read the current sidebar order to persist it. `browser.tabs.query()` returns Firefox's internal tab order, which does not reflect Zen's sidebar order when pinned tabs have been moved via `zenIndex`.

### Signature

```js
browser.zenWorkspaces.getSidebarOrder(workspaceId) → OrderItem[]
```

Returns the current sidebar order of the given workspace as an array of `OrderItem` objects (same type as `reorder()`), from top to bottom. Essential tabs are not included.

### Example

```js
const ws = await browser.zenWorkspaces.getActive();
const order = await browser.zenWorkspaces.getSidebarOrder(ws.id);

// Persist the order
for (let i = 0; i < order.length; i++) {
  console.log(i, order[i].type, order[i].id);
}
// 0  tab    101   ← pinned tab
// 1  group  5     ← Zen Folder "AI"
// 2  tab    102   ← another pinned tab
// 3  tab    103   ← normal tab
```

---

## New tab property: `zenIndex`

All `tabs.Tab` objects returned by `browser.tabs.query()`, `browser.tabs.get()`, and event callbacks gain a new property:

| Property | Type | Description |
|----------|------|-------------|
| `zenIndex` | `number \| null` | Zero-based position of this tab in its workspace sidebar, or `null` for essential tabs (which are workspace-independent). |

This allows extensions to read positions without calling `getSidebarOrder()` per-workspace.

### Example

```js
const tabs = await browser.tabs.query({ currentWindow: true });
for (const tab of tabs) {
  console.log(tab.id, tab.zenIndex, tab.zenWorkspaceId);
}
```

---

## New group property: `zenIndex`

`tabGroups.TabGroup` objects gain the same property:

| Property | Type | Description |
|----------|------|-------------|
| `zenIndex` | `number` | Zero-based position of the **start** of this group in its workspace sidebar. |

---

## Events

### `browser.tabs.onZenMoved`

Fired when a tab's sidebar position changes due to `browser.tabs.move({ zenIndex })` or `browser.zenWorkspaces.reorder()`.

**Callback:** `(tabId: number, moveInfo: { workspaceId: string, fromIndex: number, toIndex: number }) → void`

Not fired for Firefox-internal tab moves (e.g. drag-and-drop that only changes the internal `index`). Not fired during a `reorder()` call — `zenWorkspaces.onReordered` fires instead.

### `browser.tabGroups.onMoved`

Fired when a group's sidebar position changes due to `browser.tabGroups.move()` or `browser.zenWorkspaces.reorder()`.

**Callback:** `(groupId: number, moveInfo: { workspaceId: string, fromIndex: number, toIndex: number }) → void`

### `browser.zenWorkspaces.onReordered`

Fired after a successful `browser.zenWorkspaces.reorder()` call.

**Callback:** `(workspaceId: string) → void`

Call `getSidebarOrder(workspaceId)` in the handler to get the new order.

---

## `tabs.create()` extension: `zenIndex`

`browser.tabs.create()` gains an optional `zenIndex` property alongside the existing `index`:

```js
browser.tabs.create({
  url: 'https://example.com',
  pinned: true,
  zenWorkspaceId: ws.id,
  zenIndex: 2,        // place at sidebar position 2, regardless of pin status
})
```

When `zenIndex` is supplied, the tab is placed at that sidebar position after creation. The existing `index` property continues to control Firefox's internal tab list order (used for non-Zen rendering contexts). If both are supplied and conflict, `zenIndex` wins for the sidebar display.

---

## ZenSync integration example

With these APIs, ZenSync's `reconcile()` can restore an exact saved layout in two steps:

```js
// Step 1: Create all tabs and groups (order doesn't matter yet).
const restoredTabs = await createAllTabs(bookmarkEntries);
const restoredFolders = await createAllFolders(groupEntries);

// Step 2: Apply exact saved order in one atomic call.
const order = buildOrderFromBookmarks(bookmarkEntries, groupEntries);
//  order = [
//    { type: 'tab',   id: restoredTabs['https://...'] },
//    { type: 'group', id: restoredFolders['AI']       },
//    { type: 'tab',   id: restoredTabs['https://...'] },
//    ...
//  ]

await browser.zenWorkspaces.reorder(workspaceId, order);
```

No deferred pinning, no per-item `move()` calls, no event suppression required.

---

## Summary table

| API | Purpose |
|-----|---------|
| `browser.tabs.move({ zenIndex })` | Move one or more tabs to an exact sidebar position, bypassing pin hoisting |
| `browser.tabGroups.move({ zenIndex })` | Move a group/folder to an exact sidebar position as a unit |
| `browser.zenWorkspaces.reorder(id, items)` | Atomically set the complete sidebar order for a workspace |
| `browser.zenWorkspaces.getSidebarOrder(id)` | Read current sidebar order as an ordered array |
| `tab.zenIndex` | Per-tab sidebar position (readable from `tabs.query()`) |
| `tabGroup.zenIndex` | Per-group sidebar position (readable from `tabGroups.query()`) |
| `tabs.create({ zenIndex })` | Create a tab at a specific sidebar position |
| `tabs.onZenMoved` | Event: a tab's sidebar position changed |
| `tabGroups.onMoved` | Event: a group's sidebar position changed |
| `zenWorkspaces.onReordered` | Event: a workspace's entire order was atomically updated |
