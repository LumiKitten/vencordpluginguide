# Vencord / Equicord Plugin Writing Guide (AI Instructions)

> This document is written for an AI assistant. It provides everything needed to write, structure, and ship a plugin for [Vencord](https://vencord.dev) or [Equicord](https://equicord.org) from scratch.
>
> **Official documentation:**
> - Vencord: https://docs.vencord.dev/plugins/
> - Equicord: https://docs.equicord.org/plugins

---

## Table of Contents

1. [What Are Vencord/Equicord Plugins?](#1-what-are-vencordequicord-plugins)
2. [Prerequisites](#2-prerequisites)
3. [Repository Setup](#3-repository-setup)
4. [Folder Structure & Naming Rules](#4-folder-structure--naming-rules)
5. [Special Files](#5-special-files)
6. [Plugin Boilerplate](#6-plugin-boilerplate)
7. [Plugin Metadata Reference](#7-plugin-metadata-reference)
8. [Settings API](#8-settings-api)
9. [Webpack Module Finders](#9-webpack-module-finders)
10. [Patches (Code Modification)](#10-patches-code-modification)
11. [React & UI](#11-react--ui)
12. [Context Menus](#12-context-menus)
13. [Native (Node.js) Code](#13-native-nodejs-code)
14. [Lifecycle Hooks](#14-lifecycle-hooks)
15. [Build & Test Workflow](#15-build--test-workflow)
16. [Submission Checklist](#16-submission-checklist)
17. [Common Mistakes](#17-common-mistakes)
18. [Real-World Examples to Study](#18-real-world-examples-to-study)

---

## 1. What Are Vencord/Equicord Plugins?

Vencord and Equicord are Discord client mods. Plugins extend Discord's functionality by:

- **Patching** Discord's internal JavaScript (regex-based, in-place replacements)
- **Injecting React components** into the UI
- **Using webpack module finders** to access Discord's internal APIs
- **Running native Node.js code** for filesystem/OS access
- **Adding settings UI** natively integrated into Discord's settings

Plugins are written in **TypeScript** (`.ts` / `.tsx`) and are compiled into the client at build time. There is no runtime plugin loading — every change requires a rebuild.

Equicord uses the same plugin system as Vencord. All patterns here apply to both unless noted.

---

## 2. Prerequisites

Before writing any plugin, ensure:

- A **development install** of Vencord or Equicord (cloned from source, not installed via installer)
- **Node.js** (LTS) and **pnpm** installed
- **VSCode** with Vencord's recommended extensions (for snippets and TypeScript support)
- Understanding of **TypeScript** and basic **React**
- Basic **regex** knowledge (for patches)
- Familiarity with **Chrome DevTools** (for inspecting Discord's module system at runtime)
- The ability to read and understand unfamiliar code — Discord's internals are largely undocumented

---

## 3. Repository Setup

```sh
# Vencord
git clone https://github.com/Vendicated/Vencord
cd Vencord
pnpm install

# Equicord
git clone https://github.com/Equicord/Equicord
cd Equicord
pnpm install
```

Run the dev build to verify everything works before writing a plugin:

```sh
pnpm build
```

---

## 4. Folder Structure & Naming Rules

### Which folder to use?

| Folder | Client | Use when |
|---|---|---|
| `src/plugins/` | Vencord | You intend to submit the plugin to official Vencord |
| `src/equicordplugins/` | Equicord | You intend to submit the plugin to official Equicord |
| `src/userplugins/` | Both | Personal use, prototyping, not submitting |

> **When in doubt, use `src/userplugins/`.** It is not tracked by git and is the safest place to experiment.

### Naming rules

- Folder name **must be camelCase**
- No spaces, no PascalCase, no kebab-case

```
✅  src/userplugins/myFirstPlugin/
✅  src/userplugins/betterRoleColors/
❌  src/userplugins/MyFirstPlugin/
❌  src/userplugins/my-first-plugin/
❌  src/userplugins/my first plugin/
```

Each plugin lives in its **own folder**. The entry file must be named `index.ts` or `index.tsx`.

```
✅  src/userplugins/myPlugin/index.ts
✅  src/userplugins/myPlugin/index.tsx
❌  src/userplugins/index.ts            (no folder)
❌  src/userplugins/myPlugin/myPlugin.ts (wrong filename)
```

---

## 5. Special Files

Each plugin folder can contain up to three special files:

### `index.ts` / `index.tsx` (required)

The main entry point. All plugin logic, patches, and settings go here (or are imported from here).

- Runs in the **browser** context
- Has access to all browser APIs
- **No** Node.js APIs (`fs`, `path`, `child_process`, etc.)

### `native.ts` (optional)

Runs in **Node.js** (the Electron main process). Use this when you need filesystem access or OS-level operations.

- Has access to all Node.js APIs
- **No** browser APIs
- Functions here are called from `index.ts` via `VencordNative.pluginHelpers.YourPluginName`

### `README.md` (recommended for official plugins)

Markdown documentation. Used to generate the plugin's page at `https://vencord.dev/plugins/YourPlugin`. Should include:
- What the plugin does
- How to use it (any keybinds, settings, etc.)
- Screenshots, GIFs, or videos showing it in action

---

## 6. Plugin Boilerplate

### User plugin (userplugins — no dev registration needed)

```ts
// src/userplugins/myPlugin/index.ts

import definePlugin from "@utils/types";

export default definePlugin({
    name: "MyPlugin",
    description: "A short description of what this plugin does.",
    authors: [{ name: "YourName", id: 123456789012345678n }],

    // Optional: called when plugin is started
    start() {},

    // Optional: called when plugin is stopped/disabled
    stop() {},
});
```

> `id` is your **Discord user ID** (snowflake). It must be a `BigInt`, so append `n` to the number literal.

### Official Vencord plugin (src/plugins)

First, add yourself to `src/utils/constants.ts` in the `Devs` object:

```ts
// In src/utils/constants.ts
export const Devs = Object.freeze({
    // ... existing entries ...
    YourName: { name: "YourName", id: 123456789012345678n },
});
```

Then in your plugin:

```ts
// src/plugins/myPlugin/index.ts

/*
 * Vencord, a Discord client mod
 * Copyright (c) 2024 Vendicated and contributors
 * SPDX-License-Identifier: GPL-3.0-or-later
 */

import { Devs } from "@utils/constants";
import definePlugin from "@utils/types";

export default definePlugin({
    name: "MyPlugin",
    description: "A short description of what this plugin does.",
    authors: [Devs.YourName],
});
```

> VSCode snippet: type `vcPlugin` in an empty `index.ts` and accept the "Define Plugin" suggestion.

### Official Equicord plugin (src/equicordplugins)

First, add yourself to `src/utils/constants.ts` in the `EquicordDevs` object:

```ts
// In src/utils/constants.ts
export const EquicordDevs = Object.freeze({
    // ... existing entries ...
    YourName: { name: "YourName", id: 123456789012345678n },
});
```

Then in your plugin:

```ts
// src/equicordplugins/myPlugin/index.ts

import { EquicordDevs } from "@utils/constants";
import definePlugin from "@utils/types";

export default definePlugin({
    name: "MyPlugin",
    description: "A short description of what this plugin does.",
    authors: [EquicordDevs.YourName],
});
```

---

## 7. Plugin Metadata Reference

All fields available in `definePlugin({...})`:

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | ✅ | Short, unique name shown in settings |
| `description` | `string` | ✅ | One-line description shown in settings |
| `authors` | `PluginAuthor[]` | ✅ | List of authors (`Devs.X` or `{ name, id }`) |
| `patches` | `Patch[]` | ❌ | Array of webpack patches to apply |
| `settings` | `SettingsDefinition` | ❌ | Plugin settings object |
| `dependencies` | `string[]` | ❌ | Other plugin names this depends on |
| `required` | `boolean` | ❌ | If true, plugin cannot be disabled |
| `enabledByDefault` | `boolean` | ❌ | If true, enabled on fresh install |
| `hidden` | `boolean` | ❌ | If true, not shown in the settings list |
| `start()` | `function` | ❌ | Called when plugin starts |
| `stop()` | `function` | ❌ | Called when plugin stops |
| `flux` | `FluxEvents` | ❌ | Subscribe to Flux (Redux) events |
| `contextMenus` | `ContextMenus` | ❌ | Inject items into context menus |
| `renderChatBarButton` | `function` | ❌ | Add a button to the chat bar |
| `toolboxActions` | `ToolboxActions` | ❌ | Add actions to the Vencord toolbox |

---

## 8. Settings API

Use `definePluginSettings` to add a settings UI that appears in Discord's settings panel under the plugin.

```ts
import { definePluginSettings } from "@api/Settings";
import { OptionType } from "@utils/types";
import definePlugin from "@utils/types";

const settings = definePluginSettings({
    // Boolean toggle
    enableFeature: {
        type: OptionType.BOOLEAN,
        description: "Enable the main feature",
        default: true,
    },
    // Text input
    customMessage: {
        type: OptionType.STRING,
        description: "Custom message to display",
        default: "Hello!",
    },
    // Number input
    opacity: {
        type: OptionType.NUMBER,
        description: "Opacity (0–100)",
        default: 80,
    },
    // Dropdown / select
    theme: {
        type: OptionType.SELECT,
        description: "Color theme",
        options: [
            { label: "Light", value: "light", default: true },
            { label: "Dark", value: "dark" },
        ],
    },
});

export default definePlugin({
    name: "MyPlugin",
    description: "Plugin with settings",
    authors: [{ name: "YourName", id: 0n }],
    settings,

    start() {
        // Access a setting value:
        const msg = settings.store.customMessage;
        console.log(msg); // "Hello!"
    },
});
```

**Available `OptionType` values:**

| Type | UI element |
|---|---|
| `OptionType.BOOLEAN` | Toggle switch |
| `OptionType.STRING` | Text input |
| `OptionType.NUMBER` | Number input |
| `OptionType.SELECT` | Dropdown |
| `OptionType.SLIDER` | Slider |
| `OptionType.COMPONENT` | Custom React component |

---

## 9. Webpack Module Finders

Discord's code is bundled as webpack modules. Vencord exposes utilities in `@webpack` and `@webpack/common` to find and use these modules.

### Common pre-exported modules

Most things you need are already re-exported from `@webpack/common`:

```ts
import {
    React,
    ReactDOM,
    UserStore,
    GuildStore,
    ChannelStore,
    MessageStore,
    SelectedChannelStore,
    PermissionsBits,
    RestAPI,
    Toasts,
    FluxDispatcher,
    NavigationRouter,
    ModalAPI,
    Forms,           // Discord's form components
    Button,
    TextInput,
    Tooltip,
    Menu,            // Context menu components
} from "@webpack/common";
```

### Finding modules manually

When a module isn't pre-exported, use the finders from `@webpack`:

```ts
import { findByProps, find, findByCode } from "@webpack";

// Find a module that has specific properties
const SomeModule = findByProps("sendMessage", "editMessage");

// Find a module whose exports match a filter function
const AnotherModule = find(m => m.default?.displayName === "MessageContent");

// Find a module by a string of code it contains
const ThirdModule = findByCode("isStaff", "hasFlag");
```

### Lazy finders (recommended)

Using `findLazy` / `findByPropsLazy` defers the search until the value is first accessed, avoiding timing issues:

```ts
import { findByPropsLazy } from "@webpack";

// This does NOT search immediately — it creates a proxy
const MessageActions = findByPropsLazy("sendMessage", "receiveMessage");

export default definePlugin({
    start() {
        // The actual search happens here, at first access
        MessageActions.sendMessage(channelId, { content: "Hi" });
    },
});
```

> **Always prefer `...Lazy` variants** (`findLazy`, `findByPropsLazy`, `findByCodeLazy`) over their immediate counterparts. Immediate finders can fail if the module hasn't loaded yet.

---

## 10. Patches (Code Modification)

Patches are the most powerful tool. They use **regex replacements** to rewrite Discord's JavaScript in place, before it runs.

### Patch object structure

```ts
import definePlugin from "@utils/types";

export default definePlugin({
    name: "MyPlugin",
    description: "...",
    authors: [],

    patches: [
        {
            // Find: a unique string of code to locate the webpack module
            find: "someUniqueStringInDiscordsCode",

            // Replacement: what to match and what to replace it with
            replacement: {
                match: /someRegexPattern/,
                replace: "newCode",
            },
        },
    ],
});
```

### Multiple replacements in one module

```ts
patches: [
    {
        find: "someUniqueString",
        replacement: [
            {
                match: /firstPattern/,
                replace: "firstReplacement",
            },
            {
                match: /secondPattern/,
                replace: "secondReplacement",
            },
        ],
    },
],
```

### Capture groups in replacements

Use `$1`, `$2`, etc. to refer to regex capture groups:

```ts
replacement: {
    match: /(return\s*\w+\.createElement\()(\w+\.default)/,
    replace: "$1MyWrapper, { original: $2 }",
}
```

### Injecting calls with `$self`

`$self` in a replacement string refers to the plugin object itself. This lets you call plugin methods from within patched code:

```ts
replacement: {
    match: /\.renderAttachments\(/,
    replace: ".renderAttachments($self.myMethod(",
}

// In the plugin object:
myMethod(arg) {
    console.log("Called from patch with", arg);
}
```

### Full patch example: suppress a specific check

```ts
patches: [
    {
        find: "isMessageEdited",
        replacement: {
            match: /(\w+)\.isDeleted\(\)/,
            replace: "false",
        },
    },
],
```

### How to find what to patch

1. Open Discord DevTools (Ctrl+Shift+I)
2. Go to **Sources** → **Search** tab (Ctrl+Shift+F)
3. Search for a string related to the behavior you want to change
4. Find a **unique string** near the code you want — this becomes your `find:`
5. Write a regex that matches exactly the code you want to replace

> **Tip:** `find:` should be unique enough to identify exactly one webpack module. If it matches multiple modules, the patch may apply to the wrong one or cause errors.

---

## 11. React & UI

### Injecting a component via a patch

```tsx
// index.tsx (note: .tsx for JSX)
import { React } from "@webpack/common";
import definePlugin from "@utils/types";

function MyBadge({ user }) {
    return (
        <span style={{ color: "red", marginLeft: "4px" }}>
            [{user.id}]
        </span>
    );
}

export default definePlugin({
    name: "ShowUserId",
    description: "Shows user IDs next to names",
    authors: [],

    patches: [
        {
            find: "renderNameWithBadge",
            replacement: {
                match: /(\w+\.default\.getName\(\))/,
                replace: "[$1, $self.renderExtra(arguments[0])]",
            },
        },
    ],

    renderExtra({ user }) {
        return <MyBadge user={user} />;
    },
});
```

### Opening a Modal

```tsx
import { openModal, ModalRoot, ModalHeader, ModalContent, ModalFooter } from "@utils/modal";
import { Button, Forms } from "@webpack/common";

function MyModal({ modalProps }) {
    return (
        <ModalRoot {...modalProps}>
            <ModalHeader>
                <Forms.FormTitle>My Modal</Forms.FormTitle>
            </ModalHeader>
            <ModalContent>
                <Forms.FormText>Hello from a modal!</Forms.FormText>
            </ModalContent>
            <ModalFooter>
                <Button onClick={modalProps.onClose}>Close</Button>
            </ModalFooter>
        </ModalRoot>
    );
}

// Call this anywhere to open it:
openModal(props => <MyModal modalProps={props} />);
```

### Showing a Toast notification

```ts
import { Toasts } from "@webpack/common";

Toasts.show({
    message: "Something happened!",
    type: Toasts.Type.SUCCESS,  // SUCCESS | FAILURE | MESSAGE
    id: Toasts.genId(),
    options: { duration: 3000 },
});
```

---

## 12. Context Menus

Use the `contextMenus` field to add items to any right-click menu in Discord.

```tsx
import definePlugin from "@utils/types";
import { Menu } from "@webpack/common";

export default definePlugin({
    name: "MyContextMenu",
    description: "Adds an item to message context menus",
    authors: [],

    contextMenus: {
        // Key is the context menu's internal name
        "message": (children, { message }) => {
            children.push(
                <Menu.MenuItem
                    id="my-plugin-copy-id"
                    label="Copy Message ID"
                    action={() => {
                        navigator.clipboard.writeText(message.id);
                    }}
                />
            );
        },
    },
});
```

**Common context menu names:**

| Menu name | Shown when right-clicking |
|---|---|
| `message` | A message |
| `user-context` | A user (in member list or DM) |
| `guild-context` | A server name |
| `channel-context` | A channel |
| `textarea-context` | The message input box |

---

## 13. Native (Node.js) Code

If your plugin needs filesystem access or other Node.js APIs, use `native.ts`.

```ts
// native.ts
import { IpcMainInvokeEvent } from "electron";
import * as fs from "fs";
import * as path from "path";

export async function readFile(_event: IpcMainInvokeEvent, filePath: string): Promise<string> {
    return fs.readFileSync(path.resolve(filePath), "utf-8");
}
```

Call it from `index.ts`:

```ts
// index.ts
import { PluginNative } from "@utils/types";

const Native = VencordNative.pluginHelpers.MyPlugin as PluginNative<typeof import("./native")>;

export default definePlugin({
    name: "MyPlugin",
    description: "...",
    authors: [],

    async start() {
        const content = await Native.readFile("/some/path/file.txt");
        console.log(content);
    },
});
```

---

## 14. Lifecycle Hooks

### `start()` and `stop()`

```ts
export default definePlugin({
    name: "MyPlugin",
    description: "...",
    authors: [],

    start() {
        document.addEventListener("keydown", this.onKeyDown);
    },

    stop() {
        // Always clean up everything start() did
        document.removeEventListener("keydown", this.onKeyDown);
    },

    onKeyDown(e: KeyboardEvent) {
        if (e.key === "F2") console.log("F2 pressed");
    },
});
```

> **Always clean up in `stop()`.** Anything added in `start()` must be removed in `stop()`, or it will persist even after the plugin is disabled.

### Flux event subscription

Subscribe to Discord's internal Redux-like event system:

```ts
export default definePlugin({
    name: "MyPlugin",
    description: "...",
    authors: [],

    flux: {
        MESSAGE_CREATE({ message }) {
            console.log("New message:", message.content);
        },
        PRESENCE_UPDATE({ updates }) {
            // called when someone's online status changes
        },
    },
});
```

---

## 15. Build & Test Workflow

After any change to plugin files:

```sh
# Standard build
pnpm build

# Include dev-only plugins
pnpm build --dev

# Watch mode (rebuilds on file save, if available)
pnpm watch
```

Then **restart Discord** to load the updated client.

### Checking patch application in DevTools

Open Discord DevTools (Ctrl+Shift+I), go to Console, and look for:

```
[Vencord] Patch "[PatchName]" did not apply
```

If you see this, your `find:` string or `match:` regex didn't match anything in the current Discord build. Adjust them.

---

## 16. Submission Checklist

Before submitting a PR to Vencord or Equicord:

- [ ] Plugin is in `src/plugins/` (Vencord) or `src/equicordplugins/` (Equicord)
- [ ] Folder name is camelCase
- [ ] Entry file is named `index.ts` or `index.tsx`
- [ ] Author entry exists in `Devs` / `EquicordDevs` in `src/utils/constants.ts`
- [ ] License header is present at the top of all source files
- [ ] `README.md` is present with description and screenshots/GIFs
- [ ] Plugin builds without errors (`pnpm build`)
- [ ] Plugin does not break other plugins or base functionality
- [ ] Settings (if any) are properly typed with sensible defaults
- [ ] `stop()` cleans up everything `start()` created
- [ ] No `console.log` left in production code paths (use `Logger` from `@utils/Logger`)
- [ ] PR title and description clearly explain what the plugin does

---

## 17. Common Mistakes

| Mistake | Result | Fix |
|---|---|---|
| Folder is PascalCase or has spaces | Plugin silently fails to load | Rename folder to camelCase |
| Entry file not named `index.ts` | Plugin not found | Rename to `index.ts` or `index.tsx` |
| Plugin placed in wrong directory | Not compiled in | Move to correct folder |
| Forgot to rebuild after change | Old code still running | Run `pnpm build` and restart Discord |
| Using immediate finder (`findByProps`) at module top level | Module not yet loaded, returns undefined | Use `findByPropsLazy` instead |
| `find:` string matches multiple modules | Patch applies to wrong module | Use a more unique string |
| Not cleaning up in `stop()` | Listeners linger after plugin is disabled | Always mirror `start()` in `stop()` |
| Using Node.js APIs in `index.ts` | Runtime crash | Move to `native.ts` |
| Using browser APIs in `native.ts` | Runtime crash | Move to `index.ts` |
| BigInt ID without `n` suffix | TypeScript error | Use `123456789n`, not `123456789` |

---

## 18. Real-World Examples to Study

Reading existing plugins is the single best way to learn patterns.

### Vencord official plugins

Browse all: https://github.com/Vendicated/Vencord/tree/main/src/plugins

Recommended starting points:

- **`betterRoleDot`** — Simple patch, minimal code, ideal first read
- **`showHiddenChannels`** — Patching permission checks
- **`viewRaw`** — Context menu injection + modal usage
- **`messageLogger`** — Flux events, local storage, UI injection
- **`pronounDB`** — Fetching external APIs, injecting into user profiles
- **`fakeNitro`** — Complex multi-patch plugin with settings

### Equicord official plugins

Browse all: https://github.com/Equicord/Equicord/tree/main/src/equicordplugins

---

## Quick Reference

```ts
// Minimal working plugin
import definePlugin from "@utils/types";

export default definePlugin({
    name: "PluginName",
    description: "What it does.",
    authors: [{ name: "You", id: 0n }],
    patches: [],
    start() {},
    stop() {},
});
```

```ts
// Patch template
{
    find: "uniqueStringFromDiscordSource",
    replacement: {
        match: /regexMatchingCodeToReplace/,
        replace: "newCodeString or $1 capture groups",
    },
}
```

```ts
// Safe lazy module import
import { findByPropsLazy } from "@webpack";
const Module = findByPropsLazy("methodA", "methodB");
```

```ts
// Settings definition + access
import { definePluginSettings } from "@api/Settings";
import { OptionType } from "@utils/types";

const settings = definePluginSettings({
    myToggle: { type: OptionType.BOOLEAN, description: "...", default: true },
});
// Access anywhere: settings.store.myToggle
```
