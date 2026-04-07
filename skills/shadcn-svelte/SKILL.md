---
name: shadcn-svelte
description: Use when adding, using, or customizing shadcn-svelte components in a Svelte or SvelteKit project, or when you need to look up component APIs, install commands, or underlying package docs for bits-ui, formsnap, paneforge, vaul-svelte, embla-carousel, or svelte-sonner.
---

# shadcn-svelte

## Overview

shadcn-svelte is a copy-paste component library for Svelte 5 + Tailwind CSS v4, built on top of **Bits UI** primitives. Components are installed into your project source (not imported from `node_modules`), so you own and customize the code.

## CLI

### Install a component (no confirmation prompt)

```bash
npx shadcn-svelte@latest add <component> -y
```

### Install multiple components

```bash
npx shadcn-svelte@latest add button card dialog tabs -y
```

### Install all components

```bash
npx shadcn-svelte@latest add -a -y
```

### Overwrite existing files

```bash
npx shadcn-svelte@latest add <component> -y -o
```

### Initialize a new project

```bash
npx shadcn-svelte@latest init
```

Key CLI flags: `-y` (skip confirmation), `-o` (overwrite), `-a` (all components), `--no-deps` (skip dependency install).

## Discovering Available Components

**Do NOT hardcode a component list.** The registry is updated regularly. Use these live sources:

### Live Registry (machine-readable)

Fetch the current component list from:

```
https://shadcn-svelte.com/registry/index.json
```

Filter entries where `"type": "registry:ui"` to get UI components. Each entry has a `name` (the CLI install name) and `registryDependencies` (other components it depends on).

Individual component JSON (for inspecting files, dependencies):

```
https://shadcn-svelte.com/registry/<name>.json
```

### LLM-Friendly Docs

Full structured docs index:

```
https://shadcn-svelte.com/llms.txt
```

### Component Documentation

Each component doc page follows this pattern:

```
https://shadcn-svelte.com/docs/components/<name>
```

For example: `https://shadcn-svelte.com/docs/components/accordion`

## Where to Look Up API Docs

Most components are thin wrappers around **Bits UI**. When you need the full API reference (props, events, state management, accessibility), check the underlying package docs.

### Bits UI (primary primitive library)

- Docs index: `https://bits-ui.com/llms.txt`
- Full docs dump: `https://bits-ui.com/docs/llms.txt`
- Per-component: `https://bits-ui.com/docs/components/<name>`
- Per-component LLM version: `https://bits-ui.com/docs/components/<name>/llms.txt`
- GitHub: `https://github.com/huntabyte/bits-ui`

The majority of shadcn-svelte components map directly to a Bits UI component of the same name (accordion, dialog, tabs, select, popover, tooltip, etc.).

### Components with different underlying packages

| shadcn-svelte component | Underlying package | Docs |
| --- | --- | --- |
| `carousel` | embla-carousel-svelte | `https://www.embla-carousel.com` |
| `chart` | LayerChart | `https://github.com/techniq/layerchart` |
| `data-table` | @tanstack/svelte-table | `https://tanstack.com/table` |
| `drawer` | vaul-svelte | `https://vaul-svelte.com` |
| `form` (Formsnap) | formsnap + sveltekit-superforms | `https://formsnap.dev` / `https://superforms.rocks` |
| `input-otp` | Bits UI PIN Input | `https://bits-ui.com/docs/components/pin-input` |
| `resizable` | paneforge | `https://paneforge.com` |
| `sonner` | svelte-sonner | `https://svelte-sonner.vercel.app` |

### Components that are shadcn-svelte only (no underlying primitive)

These are simple styled components with no external primitive dependency -- just read the shadcn-svelte docs page:

`alert`, `badge`, `button-group`, `breadcrumb`, `card`, `empty`, `field`, `input`, `input-group`, `item`, `kbd`, `native-select`, `skeleton`, `spinner`, `table`, `textarea`, `typography`

## Usage Patterns

### Importing components

Components are installed to your `$lib/components/ui/` directory (configurable in `components.json`):

```svelte
<script lang="ts">
  import { Button } from "$lib/components/ui/button";
  import * as Dialog from "$lib/components/ui/dialog";
  import * as Tabs from "$lib/components/ui/tabs";
</script>
```

Compound components (dialog, tabs, accordion, select, etc.) use namespace imports (`import * as X`).

### Basic component example

```svelte
<script lang="ts">
  import { Button } from "$lib/components/ui/button";
  import * as Card from "$lib/components/ui/card";
</script>

<Card.Root>
  <Card.Header>
    <Card.Title>Title</Card.Title>
    <Card.Description>Description</Card.Description>
  </Card.Header>
  <Card.Content>
    <p>Content here</p>
  </Card.Content>
  <Card.Footer>
    <Button>Action</Button>
  </Card.Footer>
</Card.Root>
```

### Compound component pattern (Bits UI based)

```svelte
<script lang="ts">
  import * as Dialog from "$lib/components/ui/dialog";
  import { Button } from "$lib/components/ui/button";
</script>

<Dialog.Root>
  <Dialog.Trigger>
    {#snippet child({ props })}
      <Button {...props}>Open</Button>
    {/snippet}
  </Dialog.Trigger>
  <Dialog.Content>
    <Dialog.Header>
      <Dialog.Title>Title</Dialog.Title>
      <Dialog.Description>Description</Dialog.Description>
    </Dialog.Header>
    <p>Body</p>
    <Dialog.Footer>
      <Dialog.Close>Cancel</Dialog.Close>
    </Dialog.Footer>
  </Dialog.Content>
</Dialog.Root>
```

### State management with Svelte 5 runes

Bits UI components support `bind:value` for two-way binding:

```svelte
<script lang="ts">
  import * as Tabs from "$lib/components/ui/tabs";

  let activeTab = $state("account");
</script>

<Tabs.Root bind:value={activeTab}>
  <Tabs.List>
    <Tabs.Trigger value="account">Account</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="account">Account tab</Tabs.Content>
  <Tabs.Content value="settings">Settings tab</Tabs.Content>
</Tabs.Root>
```

## Theming

shadcn-svelte uses CSS custom properties for theming. The base color is chosen during `init`. Colors are defined in your global CSS file using HSL values. See `https://shadcn-svelte.com/docs/theming` for the full variable reference.

## Common Mistakes

- Do not import from `shadcn-svelte` or `bits-ui` directly for styled components -- import from your local `$lib/components/ui/` path.
- When looking up component APIs (props, events, sub-components), check **Bits UI docs** for Bits-based components, not just the shadcn-svelte page (which focuses on styling/usage examples).
- Do not hardcode component lists -- always fetch from the live registry at `https://shadcn-svelte.com/registry/index.json`.
- Use `-y` flag when adding components in automated/agent contexts to skip confirmation prompts.
- The `form` component requires both `formsnap` and `sveltekit-superforms` -- check both package docs for form validation patterns.
