---
name: svelte-and-sveltekit
description: Use when writing, reviewing, or refactoring Svelte 5 components or SvelteKit apps that use runes, shared state, file-based routing, load functions, remote functions, forms, hooks, navigation, or environment modules.
---

# Svelte and SvelteKit

## Overview
Prefer modern Svelte 5 runes and SvelteKit server-first patterns. Default to shared rune state over legacy stores, callback props over dispatchers, snippets over slots, server `load` or remote functions over fetching your own API, and private env modules for secrets.

## Svelte 5

### Quick Reference

| Need | Use | Avoid |
| --- | --- | --- |
| Local state | `$state()` | `let` + `$:` chains |
| Shallow/non-deep state | `$state.raw()` | deep proxying huge objects |
| Derived values | `$derived()` / `$derived.by()` | `derived()` stores, `$:` |
| Side effects | `$effect()` / `$effect.pre()` | `$:` for effects |
| Props | `$props()` | `export let` in runes mode |
| Two-way prop binding | `$bindable()` | implicit binding |
| Events | callback props | `createEventDispatcher` |
| Content projection | snippets | `<slot>` |
| Shared app state | `.svelte.ts` / `.svelte.js` + `$state` | `writable/readable/derived` |

### Core Patterns

```svelte
<script lang="ts">
	let count = $state(0);
	let settings = $state.raw({ dense: false }); // reassign, don't deep-mutate
	let doubled = $derived(count * 2);
	let label = $derived.by(() => `${count} / ${doubled}`);

	$effect.pre(() => console.log('before DOM update', count));
	$effect(() => {
		const id = setInterval(() => count++, 1000);
		return () => clearInterval(id);
	});

	let { title, value = $bindable(), ...rest } = $props();
	$inspect(count, doubled);
</script>

<input bind:value {...rest} />
<h2>{title}: {label}</h2>
```

### Global Shared State (replaces stores)

Create `.svelte.ts` files — the Svelte compiler processes runes in these files.

```ts
// src/lib/state/user.svelte.ts
export const userState = $state({ name: '', loggedIn: false });

export class CartState {
	items = $state<{ id: string; qty: number }[]>([]);
	total = $derived(this.items.reduce((n, i) => n + i.qty, 0));
}
export const cartState = new CartState();
```

```svelte
<script lang="ts">
	import { userState } from '$lib/state/user.svelte';
	userState.name = 'Ada'; // reactive everywhere
</script>

<p>{userState.name}</p>
```

### Component Patterns: Callback Props + Snippets

```svelte
<!-- Child.svelte -->
<script lang="ts">
	let { onSave, item, row } = $props();
</script>

<button onclick={() => onSave(item)}>Save</button>
{@render row(item)}
```

```svelte
<!-- Parent.svelte -->
<script lang="ts">
	import Child from './Child.svelte';
	const save = (item) => console.log(item);
	let item = $state({ id: 1, name: 'A' });
</script>

<Child {item} onSave={save}>
	{#snippet row(product)}
		<strong>{product.name}</strong>
	{/snippet}
</Child>
```

### Common Mistakes (Svelte 5)
- Do not use `writable`, `readable`, or `derived` stores — shared rune state replaces them.
- Do not use `createEventDispatcher` — pass callbacks as props.
- Do not use `<slot>` — use snippets and `{@render ...}`.
- Do not use `$:` — use `$derived` for values and `$effect` for side effects.
- Do not put rune state in plain `.ts` — must be `.svelte.ts` or `.svelte.js`.

---

## SvelteKit

### Quick Reference

| Need | Use | Avoid |
| --- | --- | --- |
| Route UI | `+page.svelte`, `+layout.svelte`, `+error.svelte` | custom routers |
| Shared/load data | `+layout.ts` / `+layout.server.ts` | prop drilling |
| Page-specific data | `+page.ts` / `+page.server.ts` | client-only boot fetches |
| Read data on server | `query()` | `+server.ts` for simple reads |
| Mutate data | `command()` / `form()` | extra internal API layers |
| Progressive forms | `form()` or actions + `use:enhance` | JS-only forms |
| Auth/session | `hooks.server.ts` + `event.locals` | client-trusted auth |
| Navigation | `$app/navigation` | `window.location` |
| App state | `$app/state` | ad hoc globals |
| Secrets | `$env/static/private` or `$env/dynamic/private` | public env modules |

### Data Loading

```ts
// src/routes/dashboard/+page.server.ts
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals }) => {
	return { user: locals.user, projects: await db.projects.forUser(locals.user.id) };
};
```

```svelte
<!-- src/routes/dashboard/+page.svelte -->
<script lang="ts">
	let { data } = $props();
</script>

<h1>{data.user.name}</h1>
```

Use universal `+page.ts` when code may run on server and client. Use `+page.server.ts` when you need DB access, filesystem, cookies, `locals`, or private env.

### Remote Functions (modern RPC pattern)

Define in `*.remote.ts` files. These replace many uses of `+server.ts`.

```ts
// src/routes/posts/posts.remote.ts
import * as v from 'valibot';
import { query, command, form } from '$app/server';
import { redirect } from '@sveltejs/kit';

// query() — for reading data (GET-like)
export const getPosts = query(async () => db.post.all());
export const getPost = query(v.string(), (slug) => db.post.bySlug(slug));

// command() — for mutations (POST-like)
export const publishPost = command(v.string(), async (slug) => {
	await db.post.publish(slug);
	await getPosts().refresh(); // re-fetch related queries
});

// form() — for progressive-enhancement forms
export const createPost = form(
	v.object({ title: v.string() }),
	async ({ title }) => {
		const slug = await db.post.create(title);
		await getPosts().refresh();
		redirect(303, `/posts/${slug}`);
	}
);
```

```svelte
<script lang="ts">
	import { createPost, getPosts, publishPost } from './posts.remote';
	const posts = getPosts();
</script>

<!-- form() returns props for progressive enhancement -->
<form {...createPost}>
	<input name="title" />
	<button>Create</button>
</form>

{#each posts.current as post}
	<button onclick={() => publishPost(post.slug)}>Publish</button>
{/each}
```

### Form Actions (classic pattern, still valid)

```ts
// src/routes/login/+page.server.ts
import type { Actions } from './$types';
import { fail } from '@sveltejs/kit';

export const actions: Actions = {
	default: async ({ request }) => {
		const data = await request.formData();
		const email = data.get('email');
		if (!email) return fail(400, { email, missing: true });
		// ... authenticate
	}
};
```

```svelte
<script lang="ts">
	import { enhance } from '$app/forms';
	let { form } = $props();
</script>

<form method="POST" use:enhance>
	<input name="email" value={form?.email ?? ''} />
	{#if form?.missing}<p>Email required</p>{/if}
	<button>Log in</button>
</form>
```

### Hooks

```ts
// src/hooks.server.ts
import type { Handle, HandleFetch, HandleServerError } from '@sveltejs/kit';

export const handle: Handle = async ({ event, resolve }) => {
	event.locals.user = await auth.fromCookies(event.cookies);
	return resolve(event);
};

export const handleFetch: HandleFetch = async ({ request, fetch }) => fetch(request);

export const handleError: HandleServerError = ({ error, event }) => {
	console.error(error, event.url.pathname);
};
```

### Useful $app Modules

- `$app/state`: `page`, `navigating`, `updated`
- `$app/navigation`: `goto`, `invalidate`, `invalidateAll`
- `$app/environment`: `browser`, `dev`, `building`
- `$env/static/private|public`, `$env/dynamic/private|public`

### Common Mistakes (SvelteKit)
- Prefer remote functions before reaching for `+server.ts` — reserve endpoints for true HTTP APIs, webhooks, or external consumers.
- Do not fetch your own SvelteKit endpoints from server `load` — call the DB/service directly.
- Do not put secrets in `$env/static/public` or `$env/dynamic/public`.
- Do not trust client auth state — set session/user data in `hooks.server.ts`.
- Remote function modules use `*.remote.ts` naming convention.
