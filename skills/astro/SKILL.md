---
name: astro
description: Use when building or updating Astro sites, components, content-driven pages, islands, content collections, view transitions, or Astro server features such as SSR, actions, API routes, middleware, and adapters.
---

# Astro

## Overview

Astro is a content-first web framework built around static HTML by default, with optional islands for client-side interactivity and opt-in server features for fullstack work.

Use Astro's defaults aggressively: render as much as possible to HTML, add JavaScript only where needed, and turn on server rendering only for routes or components that truly need it.

## Part 1: Astro Core

### Quick Reference

| Concept | Default | Key API |
| --- | --- | --- |
| `.astro` component | Server-rendered HTML | `Astro.props`, `interface Props` |
| Slots | Empty unless filled | `<slot />`, `<slot name="...">` |
| Styles | Scoped | `<style>`, `<style is:global>` |
| Client JS | None | `<script>`, `<script is:inline>` |
| File routing | `src/pages/` | `[slug].astro`, `[...slug].astro` |
| Dynamic static routes | Not auto-generated | `getStaticPaths()` |
| Client islands | No JS by default | `client:load/idle/visible/media/only` |
| Server islands | Off | `server:defer` |
| Content collections | Loader required (v5+) | `defineCollection()`, `glob()` |
| Collection queries | Explicit | `getCollection()`, `getEntry()` |
| Render entry body | Manual | `render()` -> `<Content />` |
| Bulk imports | Lazy by default | `import.meta.glob()` |
| Env vars | Build/runtime aware | `import.meta.env`, `PUBLIC_*` prefix |
| Image optimization | Built-in | `<Image />`, `<Picture />` from `astro:assets` |

### Component Model

```astro
---
interface Props {
  title: string;
  description?: string;
}

const { title, description = "No description." } = Astro.props;
---

<section class="card">
  <h2>{title}</h2>
  <p>{description}</p>
  <slot />
</section>

<style>
  .card { padding: 1rem; border: 1px solid #d1d5db; border-radius: 0.75rem; }
</style>
```

Named slots:

```astro
<!-- Layout.astro -->
<article>
  <header><slot name="header" /></header>
  <div class="content"><slot /></div>
</article>
```

```astro
<!-- Usage -->
<Layout>
  <h2 slot="header">Title</h2>
  <p>Body content here.</p>
</Layout>
```

### Routing

File-based from `src/pages/`. Dynamic routes need `getStaticPaths()` in static mode.

```astro
---
// src/pages/blog/[slug].astro
export async function getStaticPaths() {
  const posts = [
    { slug: "hello-astro", title: "Hello Astro" },
    { slug: "content-layer", title: "Content Layer" },
  ];
  return posts.map((post) => ({
    params: { slug: post.slug },
    props: { post },
  }));
}

const { post } = Astro.props;
---

<h1>{post.title}</h1>
```

Rest params with `[...slug].astro`:

```astro
---
export async function getStaticPaths() {
  return [
    { params: { slug: undefined }, props: { title: "Docs Home" } },
    { params: { slug: "guides" }, props: { title: "Guides" } },
    { params: { slug: "guides/install" }, props: { title: "Install Guide" } },
  ];
}
const { title } = Astro.props;
---
<h1>{title}</h1>
```

### Islands Architecture

All components render to static HTML by default — zero JS. Use client directives to hydrate interactive pieces:

| Directive | When it hydrates |
| --- | --- |
| `client:load` | Immediately on page load |
| `client:idle` | When browser is idle |
| `client:visible` | When component enters viewport |
| `client:media="(query)"` | When media query matches |
| `client:only="react"` | Client-only, skip SSR entirely |

```astro
---
import ThemeToggle from "../components/ThemeToggle.svelte";
import NewsletterForm from "../components/NewsletterForm.jsx";
import CartDrawer from "../components/CartDrawer.vue";
---

<ThemeToggle client:load />
<NewsletterForm client:visible />
<CartDrawer client:idle />
```

Guidance:
- Default to **no directive** (static)
- Prefer `client:visible` or `client:idle` over `client:load`
- Use `client:only` only for browser-only libs that cannot SSR
- Each island is isolated — you can mix React, Svelte, Vue on one page

### Server Islands

Defer rendering of dynamic server components with `server:defer`. The page shell renders immediately; the component loads afterward.

```astro
---
import UserAvatar from "../components/UserAvatar.astro";
import AvatarFallback from "../components/AvatarFallback.astro";
---

<UserAvatar server:defer>
  <AvatarFallback slot="fallback" />
</UserAvatar>
```

Inside the deferred component, access cookies/sessions:

```astro
---
const session = Astro.cookies.get("session");
const userName = session ? "Brice" : "Guest";
---
<p>Signed in as {userName}</p>
```

Use for personalized UI that shouldn't block first paint. Don't use for content that could be static.

### Content Collections (v5+ Content Layer)

Define in `src/content.config.ts`. Collections require a `loader` in v5+.

```ts
// src/content.config.ts
import { defineCollection, reference } from "astro:content";
import { glob } from "astro/loaders";
import { z } from "astro/zod";

const authors = defineCollection({
  loader: glob({ pattern: "**/*.json", base: "./src/data/authors" }),
  schema: z.object({
    name: z.string(),
    bio: z.string().optional(),
  }),
});

const blog = defineCollection({
  loader: glob({ pattern: "**/*.{md,mdx}", base: "./src/content/blog" }),
  schema: z.object({
    title: z.string(),
    pubDate: z.coerce.date(),
    author: reference("authors"),
    tags: z.array(z.string()).default([]),
  }),
});

export const collections = { blog, authors };
```

Query and render:

```astro
---
import { getCollection, getEntry, render } from "astro:content";

// List all
const posts = await getCollection("blog");

// Single entry + render
const entry = await getEntry("blog", "hello-astro");
const { Content } = await render(entry);
---

<ul>
  {posts.map((p) => <li><a href={`/blog/${p.id}/`}>{p.data.title}</a></li>)}
</ul>

<article>
  <h1>{entry.data.title}</h1>
  <Content />
</article>
```

Custom loaders for APIs/CMS:

```ts
const docs = defineCollection({
  loader: myCmsLoader({ endpoint: "https://cms.example.com/docs" }),
});
```

### Assets

```astro
---
import { Image, Picture } from "astro:assets";
import hero from "../assets/hero.jpg";
---

<Image src={hero} alt="Hero" width={1200} height={800} />
<Picture src={hero} alt="Hero" widths={[480, 960, 1440]} formats={["avif", "webp"]} />
```

---

## Part 2: Astro Server / SSR / Fullstack

### Quick Reference

| Concept | Key API | Notes |
| --- | --- | --- |
| Static output | `output: "static"` | Default, full SSG |
| Server output | `output: "server"` | All pages SSR by default |
| Per-route control | `export const prerender = true/false` | Mix static + SSR |
| Actions | `defineAction()` from `astro:actions` | Type-safe server functions |
| Form actions | `accept: "form"` | Progressive enhancement |
| API endpoints | `GET`, `POST` in `src/pages/api/*.ts` | Return `new Response()` |
| Middleware | `defineMiddleware()`, `sequence()` | Auth, locale, request state |
| View transitions | `<ClientRouter />` from `astro:transitions` | SPA-like navigation |
| Adapters | Node, Vercel, Netlify, Cloudflare | Required for server features |

### Rendering Modes

```js
// astro.config.mjs — mostly static site with some SSR pages
import { defineConfig } from "astro/config";
import node from "@astrojs/node";

export default defineConfig({
  output: "static",
  adapter: node(),
});
```

Per-page SSR opt-in:

```astro
---
export const prerender = false; // this page renders on demand

const user = Astro.cookies.get("session");
---
<h1>{user ? "Welcome back" : "Welcome"}</h1>
```

Note: `output: "hybrid"` was removed in Astro v5. Use `output: "static"` + `prerender = false` per route instead.

### Actions (Type-Safe Server Functions)

Define in `src/actions/index.ts`:

```ts
// src/actions/index.ts
import { defineAction } from "astro:actions";
import { z } from "astro/zod";

export const server = {
  subscribe: defineAction({
    accept: "form",
    input: z.object({ email: z.string().email() }),
    handler: async ({ email }) => {
      return { message: `Subscribed ${email}` };
    },
  }),

  searchProducts: defineAction({
    input: z.object({ query: z.string().min(2) }),
    handler: async ({ query }) => {
      return [{ id: "p1", name: `Match for ${query}` }];
    },
  }),
};
```

Call from client:

```astro
<script>
  import { actions } from "astro:actions";

  document.querySelector("#search")?.addEventListener("click", async () => {
    const { data, error } = await actions.searchProducts({ query: "astro" });
    if (error) return console.error(error);
    console.log(data);
  });
</script>
```

Call from server:

```astro
---
import { actions } from "astro:actions";
const { data, error } = await Astro.callAction(actions.searchProducts, { query: "astro" });
---
<pre>{JSON.stringify(data, null, 2)}</pre>
```

### API Endpoints

```ts
// src/pages/api/hello.ts
import type { APIRoute } from "astro";

export const GET: APIRoute = async ({ request, cookies, locals }) => {
  return new Response(JSON.stringify({ message: "Hello" }), {
    headers: { "Content-Type": "application/json" },
  });
};

export const POST: APIRoute = async ({ request }) => {
  const body = await request.json();
  return new Response(JSON.stringify({ received: body }), { status: 201 });
};
```

### Middleware

```ts
// src/middleware.ts
import { defineMiddleware, sequence } from "astro:middleware";

const withAuth = defineMiddleware(async (context, next) => {
  const session = context.cookies.get("session");
  context.locals.user = session ? { id: "u1", name: "Brice" } : null;
  return next();
});

const withRequestId = defineMiddleware(async (context, next) => {
  context.locals.requestId = crypto.randomUUID();
  return next();
});

export const onRequest = sequence(withRequestId, withAuth);
```

### View Transitions

Use `<ClientRouter />` (replaces legacy `<ViewTransitions />`):

```astro
---
import { ClientRouter } from "astro:transitions";
---

<html lang="en">
  <head>
    <ClientRouter />
  </head>
  <body>
    <header transition:persist>
      <nav transition:animate="slide">
        <a href="/">Home</a>
        <a href="/blog">Blog</a>
      </nav>
    </header>
    <main transition:name="page-content">
      <slot />
    </main>
  </body>
</html>
```

- `transition:persist` — keep state across navigations
- `transition:animate` — control animation style
- `transition:name` — pair elements across pages

### Adapters

Required for any server features (SSR, actions, server islands, API routes with request context).

```js
// astro.config.mjs
import vercel from "@astrojs/vercel/serverless";

export default defineConfig({
  adapter: vercel(),
  output: "server",
});
```

Options: `@astrojs/node`, `@astrojs/vercel`, `@astrojs/netlify`, `@astrojs/cloudflare`

## Common Mistakes

- Forgetting `client:*` directives — framework components are static HTML by default, interactive code won't run.
- Exposing secrets with `PUBLIC_` prefix — only browser-safe values belong there.
- Using old v4 content collection patterns — v5+ requires a `loader` in `defineCollection()`.
- Missing `getStaticPaths()` for dynamic routes in static builds.
- Using `output: "hybrid"` — removed in Astro v5; use `static` + per-route `prerender = false`.
- Mixing `output: "static"` with server features (actions, server islands) without installing an adapter.
- Overusing `client:load` — prefer `client:idle` or `client:visible` for better performance.
- Using `server:defer` for content that could be built statically.
- Putting global CSS in scoped `<style>` blocks and expecting it to affect other components.

## Practical Defaults

1. Start with static `.astro` components
2. Add client islands only around interactive UI
3. Use content collections for structured local content
4. Use `output: "static"` unless the app is mostly request-driven
5. Add `prerender = false` only on routes that need server rendering
6. Use actions for validated forms and server mutations
7. Use middleware for shared request state (auth, locale)
8. Install the adapter early once you introduce any server feature
