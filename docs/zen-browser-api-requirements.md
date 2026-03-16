# Zen Browser WebExtension API Requirements

**Project:** ZenSync — cross-device tab sync extension for Zen Browser
**Zen source:** https://github.com/zen-browser/desktop
**Date:** 2026-03-16
**Status:** Draft for Zen Browser contributors

---

## Background

ZenSync syncs tabs across devices by encoding tab state into Firefox bookmarks and relying on Firefox's native bookmark sync. It is a standard WebExtension (Manifest V2) that uses only `browser.*` APIs — no native messaging, no `fx-autoconfig`, no `window.plugins`.

Zen Browser introduces three concepts that are not yet fully accessible to WebExtensions:

1. **Zen Folders** — Zen's variant of Firefox tab groups, visually distinct, automatically pinned, workspace-scoped
2. **Essential tabs** — Tabs marked as always-on, distinguished by a XUL attribute not yet surfaced to extensions
3. **Workspaces** — Zen's tab isolation feature, accessible today only via `window.plugins` (not available in the shipped browser)

Each gap below describes the exact symptom ZenSync hits, the minimal API addition needed to fix it, and a pointer into the Zen/Firefox source tree where the change would live.

---

## Gap 1: Zen Folder creation

### Problem

ZenSync's restore path needs to recreate Zen Folders when syncing from one device to another. The only WebExtension API that creates tab groups is `browser.tabs.group()`. Calling it creates a **standard Firefox tab group** — not a Zen Folder. The result:

- Renders with Firefox's default tab group chrome, not Zen's Folder UI
- Is not pinned (Zen Folders are pinned automatically)
- Is not integrated into Zen's workspace system
- Does not receive the Zen-specific visual treatment

Passing `color: "zen-workspace-color"` to `browser.tabGroups.update()` after creation is not a workaround — that value is rejected by the color enum validator even in Zen's fork.

### Proposed API addition

Add a `zenFolder: boolean` property to the `TabGroupUpdateProperties` schema, accepted by `browser.tabGroups.update()`:

```js
// Create a group via existing API, then promote it to a Zen Folder
const groupId = await browser.tabs.group({ tabIds: [tabId] });
await browser.tabGroups.update(groupId, {
  title: 'My Work',
  zenFolder: true,         // ← new property: promotes to Zen Folder
});
```

Setting `zenFolder: true` should cause Zen to apply all Zen Folder semantics to the group: pin it, apply Zen's visual chrome, and associate it with the creating tab's workspace.

Alternatively, a dedicated `browser.zenFolders.create()` method would provide more explicit control:

```js
const folder = await browser.zenFolders.create({
  title: 'My Work',
  workspaceId: 'uuid-of-workspace',  // optional; defaults to active workspace
  tabIds: [tab1.id, tab2.id],
});
// folder.id is the standard tabGroups groupId, usable with existing tabGroups API
```

### Why `tabGroups.update` extension is preferred

The `browser.tabs.group()` + `browser.tabGroups.update()` pattern is already established and tested. Extending `update()` with a `zenFolder` flag is the smallest possible change and keeps the API surface consistent. Extensions that don't know about `zenFolder` continue to create regular Firefox groups; Zen-aware extensions opt in explicitly.

### Implementation hints

In Firefox's tab groups implementation the relevant files are:

- **Schema:** `toolkit/components/extensions/schemas/tab_groups.json` — add `zenFolder` to `TabGroupUpdateProperties`
- **Implementation:** `toolkit/components/extensions/parent/ext-tabGroups.js` — handle `zenFolder` in the `update()` method, call Zen's internal folder-promotion logic
- **Zen folder logic:** Search for `MozTabbrowserTabGroup` subclassing or `zenFolder` attribute in Zen's `src/` tree — the internal promotion code already exists since Zen creates Folders from its own UI

---

## Gap 2: Essential tab detection and management

### Problem

Zen's "essential tabs" are tabs marked as always-on — they persist across workspace switches and are indicated by a special XUL attribute (`zen-essential`). ZenSync needs to:

1. **Detect** which open tabs are essential (so it can encode them as `ESS|` in bookmark titles and restore them as essential)
2. **Restore** a tab as essential (so a synced essential tab comes back essential, not just pinned)

Currently, `zen-essential` is not part of `browser.tabs.Tab` — it is not surfaced to extensions at all. ZenSync's workaround is to treat essential tabs as pinned tabs (`PIN|`), which is lossy: the essential status is lost on restore.

### Proposed API addition

**Detection:** Add an `essential` boolean property to `browser.tabs.Tab`:

```js
// browser.tabs.Tab gains:
{
  essential: boolean  // true if this tab is a Zen essential tab
}

// Usage
const tabs = await browser.tabs.query({ essential: true });
const essentialTabs = tabs.filter(t => t.essential);
```

**Management:** Add `essential` to `browser.tabs.UpdateProperties`:

```js
// Mark a tab as essential
await browser.tabs.update(tabId, { essential: true });

// Remove essential status
await browser.tabs.update(tabId, { essential: false });
```

**Events:** `browser.tabs.onUpdated` should include `essential` in its `changeInfo` when the status changes:

```js
browser.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {
  if ('essential' in changeInfo) {
    // essential status changed
  }
});
```

### Implementation hints

- **Schema:** `toolkit/components/extensions/schemas/tabs.json` — add `essential` to `Tab` type and `UpdateProperties`
- **Implementation:** `toolkit/components/extensions/parent/ext-tabs.js` — read the XUL attribute in the tab-to-API-object conversion, handle `essential` in `update()`, emit `changeInfo` on change
- **XUL attribute:** Search Zen's source for `zen-essential` to find where it is set/cleared on `<tab>` elements — the WebExtension bridge just needs to read/write that same attribute

---

## Gap 3: Workspace API via `browser.*`

### Problem

Zen Workspaces are accessible today through `window.plugins` — an internal Zen API that is **not available in the shipped browser**. ZenSync's `WorkspaceAPI` class falls back to inferring workspace from `tab.cookieStoreId` (the Firefox container ID), which works because Zen assigns one container per workspace — but this is an undocumented implementation detail, not a stable contract.

The specific operations ZenSync needs:

1. **List workspaces** — get all current workspaces with their UUIDs/names
2. **Get a tab's workspace** — given a tab object, return its workspace UUID
3. **Create a tab in a specific workspace** — when restoring, open a tab directly into the target workspace rather than relying on post-creation moves
4. **Workspace change events** — be notified when a tab moves between workspaces

### Proposed API addition

A `browser.zenWorkspaces` namespace:

```js
// List all workspaces
const workspaces = await browser.zenWorkspaces.query();
// → [{ id: 'uuid', name: 'Work', position: 0, active: true }, ...]

// Get active workspace
const ws = await browser.zenWorkspaces.getActive();

// Get a tab's current workspace
const wsId = await browser.zenWorkspaces.getTabWorkspace(tabId);
// → 'uuid-of-workspace'

// Create tab in specific workspace (extends browser.tabs.create)
// Option A: new namespace method
await browser.zenWorkspaces.createTab({ url, workspaceId: 'uuid' });

// Option B: extend browser.tabs.create with zenWorkspaceId
await browser.tabs.create({ url, zenWorkspaceId: 'uuid' });

// Events
browser.zenWorkspaces.onTabMoved.addListener((tabId, { fromWorkspace, toWorkspace }) => {});
browser.zenWorkspaces.onChanged.addListener((workspace, changeInfo) => {});
```

### Minimal viable subset

If a full `browser.zenWorkspaces` namespace is too large for an initial PR, the single highest-value addition is:

```js
// Add zenWorkspaceId to browser.tabs.Tab
{
  zenWorkspaceId: string | null  // null if tab is in the default workspace
}
```

This alone would let ZenSync reliably detect a tab's workspace without relying on the container-inference heuristic. All other workspace operations (list, create-in-workspace, events) would be follow-on additions.

### Implementation hints

- **Schema:** New file `toolkit/components/extensions/schemas/zen_workspaces.json`, or extend `tabs.json` for the `zenWorkspaceId` tab property
- **Implementation:** New file `toolkit/components/extensions/parent/ext-zenWorkspaces.js`, mirroring the pattern of `ext-tabGroups.js`
- **Zen internals:** The workspace system already tracks tab↔workspace assignments. The WebExtension bridge needs to expose read/write access to that existing state

---

## Gap 4: Zen Folder ↔ workspace binding on creation

### Problem

Even if Gap 1 is fixed (Zen Folder creation via WebExtension), a restored Zen Folder needs to land in the **correct workspace** — the same workspace it was in on the source device. Today:

- `browser.tabs.group()` creates a group in whatever workspace is currently active
- There is no parameter to `tabs.group()` or `tabGroups.update()` that specifies a target workspace

### Proposed API addition

Extend `browser.tabs.group()` with a `zenWorkspaceId` option:

```js
const groupId = await browser.tabs.group({
  tabIds: [tab1.id, tab2.id],
  zenWorkspaceId: 'target-workspace-uuid',
});
```

Or, if Gap 1 adds `browser.zenFolders.create()`, include `workspaceId` there (already shown in Gap 1's proposed API).

---

## Summary table

| Gap | Missing API | Impact on ZenSync | Complexity |
|-----|-------------|-------------------|------------|
| 1 | Zen Folder creation (`tabGroups.update({ zenFolder: true })`) | Cannot recreate Zen Folders on restore; creates generic Firefox groups instead | Medium |
| 2a | `tab.essential` property (read) | Cannot distinguish essential tabs; synced as pinned, lose essential status | Low |
| 2b | `tabs.update({ essential })` (write) | Cannot restore a tab as essential | Low |
| 3a | `tab.zenWorkspaceId` property | Workspace detection relies on container inference heuristic | Low |
| 3b | `browser.zenWorkspaces.query()` + events | Cannot reliably list workspaces or respond to workspace changes | Medium |
| 4 | `tabs.group({ zenWorkspaceId })` | Restored Zen Folders land in the wrong workspace | Low (follows from Gap 1) |

**Highest value, lowest effort first:**
Gap 2a (read `tab.essential`) → Gap 3a (read `tab.zenWorkspaceId`) → Gap 1 (`zenFolder` flag on update) → Gap 2b (write essential) → Gap 3b (full workspace namespace) → Gap 4 (workspace-scoped group creation)

---

## How ZenSync would use these APIs

Once all gaps are filled, the restore path would look like:

```js
// Restore a Zen Folder bookmark
const groupId = await browser.tabs.group({
  tabIds: memberTabIds,
  zenWorkspaceId: folderBookmark.workspaceId,  // Gap 4
});
await browser.tabGroups.update(groupId, {
  title: folderBookmark.name,
  color: safeGroupColor(folderBookmark.color),
  collapsed: folderBookmark.collapsed,
  zenFolder: true,  // Gap 1 — promotes to Zen Folder
});

// Restore an essential tab
const tab = await browser.tabs.create({ url });
if (bookmarkEntry.type === 'ESS') {
  await browser.tabs.update(tab.id, { essential: true });  // Gap 2b
}

// Detect essential tabs on outbound sync
browser.tabs.onUpdated.addListener((tabId, changeInfo) => {
  if ('essential' in changeInfo) {
    tabWatcher.handleEssentialChange(tabId, changeInfo.essential);
  }
});
```

And the outbound bookmark title format would gain an `ESS|` type that actually means something:

```
ESS|003|1741824000|Always-on Dashboard  ← currently treated identically to PIN
```

---

## Related files in Zen source

Suggested starting points when reading the Zen Browser source:

| Topic | File(s) to read |
|-------|-----------------|
| Tab Groups WebExtension bridge | `toolkit/components/extensions/parent/ext-tabGroups.js` |
| Tab API | `toolkit/components/extensions/parent/ext-tabs.js` |
| Extension schemas | `toolkit/components/extensions/schemas/tab_groups.json`, `tabs.json` |
| Zen Folder internal implementation | Search `zen-browser/desktop` for `MozTabbrowserTabGroup`, `zenFolder`, `zen-folder` |
| Essential tab attribute | Search `zen-browser/desktop` for `zen-essential` |
| Workspace system | Search `zen-browser/desktop` for `ZenWorkspaces`, `zenWorkspace` |
| Extension manifest permissions | `toolkit/components/extensions/schemas/manifest.json` — add new permissions if needed |

---

## Notes for a pull request

- All proposed additions are **additive** — no existing behavior changes, no breaking changes to Firefox or standard extension APIs
- Zen-specific properties (`zenFolder`, `zenWorkspaceId`, `essential`) should be gated behind a `zen` permission in `manifest.json` so standard Firefox extensions don't accidentally depend on them
- Each gap can be implemented independently; they do not depend on each other
- ZenSync's test suite (`npm test` in the extension repo) provides a ready-made integration harness to validate the changes once implemented — the restore path tests can be re-enabled and extended as each API becomes available
