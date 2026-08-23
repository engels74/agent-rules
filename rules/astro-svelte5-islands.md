---
type: "agent_requested"
description: "Bun + Astro 7 + Svelte 5 islands + UnoCSS + shadcn-svelte + Biome coding guidelines"
---

# Astro 7 Islands with Svelte 5, UnoCSS, and Bun

This stack is a content-first, zero-JS-by-default site engine (Astro 7) that ships islands of interactivity as compiled Svelte 5 components, styled by UnoCSS's on-demand atomic engine with shadcn-svelte components adapted via `unocss-preset-shadcn`, all run and packaged by Bun and kept clean by Biome. Optimize for **static HTML by default, minimal hydrated JavaScript, and type-safe server boundaries** (content collections, actions, `astro:env`). The framework's job is to render HTML on the server; a Svelte island exists only where you explicitly opted into client interactivity with a `client:*` directive.

The biggest way agents write wrong-but-plausible code here is importing habits from adjacent ecosystems: writing Svelte 4 syntax (`export let`, `on:click`, `$:`, stores-first) instead of Svelte 5 runes; writing Tailwind-project assumptions (a `tailwind.config.js`, PostCSS, `@apply`) instead of UnoCSS's `uno.config.ts`; reaching for Node/npm APIs and `package-lock.json` instead of Bun's runtime and text lockfile; and treating an Astro island like a React app root (passing functions or non-serializable props across the island boundary). Everything below shows the one modern, correct way.

## Stack snapshot & versions

- **Research date:** 2026-08-22
- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories.

| Tool | Target version | Notes |
| --- | --- | --- |
| Bun | 1.4.x | Rust rewrite; runtime + package manager + test runner + bundler. Node 26-compatible. |
| Astro | 7.2.x (7.0 released June 22, 2026) | Rust `.astro` compiler, Vite 8 + Rolldown, Sätteri Markdown, queued rendering default. |
| `@astrojs/svelte` | 9.x | The Svelte 5 line. Install `@astrojs/svelte@5` only if you need Svelte 3/4. |
| Svelte | 5.x | Runes; attachments since 5.29; `$props.id()` since 5.20. |
| nanostores | 1.x | Cross-island state. Svelte needs no adapter (store contract); `@nanostores/svelte-runes` 1.x only for `.svelte.ts`. |
| TypeScript | 5.9.x | `erasableSyntaxOnly` since 5.8; `verbatimModuleSyntax` since 5.0. |
| UnoCSS | 66.x | `presetWind4` (Tailwind v4-compatible). |
| `unocss-preset-shadcn` | 1.0.1 | Supports `presetWind4` by default; legacy Wind3 at `unocss-preset-shadcn/v3`. |
| shadcn-svelte | 1.5.0 (released Aug 2, 2026) | Svelte 5; CLI defaults to Tailwind v4 — see UnoCSS coexistence section. |
| Biome | 2.5.x | Formatter + linter; framework file support is opt-in and experimental. |

Runtime floors: Astro 7 requires **Node.js 22+** (met by Bun's Node compatibility). Assume every floor is met; do not hedge for older runtimes.

## Project layout & Bun as the runner

Run everything through Bun. Use `bun --bun` so Astro's CLI executes on the Bun runtime rather than Node.

```jsonc
// package.json
{
  "name": "site",
  "type": "module",
  "private": true,
  "scripts": {
    "dev": "bun --bun astro dev",
    "build": "bun --bun astro build",
    "preview": "bun --bun astro preview",
    "check": "astro check && biome check .",
    "fix": "biome check --write .",
    "test": "bun test"
  },
  "dependencies": {
    "astro": "^7.2.0",
    "@astrojs/svelte": "^9.0.0",
    "@astrojs/node": "^11.0.0",
    "svelte": "^5.38.0",
    "nanostores": "^1.5.0",
    "@nanostores/persistent": "^1.3.0",
    "clsx": "^2.1.1",
    "tailwind-merge": "^3.0.0",
    "tailwind-variants": "^3.3.0"
  },
  "devDependencies": {
    "@biomejs/biome": "^2.5.0",
    "unocss": "^66.8.0",
    "@unocss/astro": "^66.8.0",
    "@unocss/reset": "^66.8.0",
    "unocss-preset-shadcn": "^1.0.1",
    "unocss-preset-animations": "^1.3.0",
    "typescript": "^5.9.0"
  }
}
```

Standard project shape:

```text
├─ astro.config.mts        # Astro config (typed, .mts)
├─ uno.config.ts           # UnoCSS config — the styling source of truth
├─ biome.json              # formatter + linter
├─ bunfig.toml             # Bun runtime/install config
├─ bun.lock                # text lockfile — commit this
├─ tsconfig.json
├─ components.json         # shadcn-svelte registry config
└─ src/
   ├─ pages/               # file-based routes (.astro, .ts endpoints)
   ├─ layouts/
   ├─ components/          # .astro + .svelte islands
   │  └─ ui/               # shadcn-svelte components live here
   ├─ lib/
   │  ├─ utils.ts          # cn() helper
   │  └─ stores/           # nanostores + .svelte.ts rune modules
   ├─ actions/index.ts     # astro:actions
   ├─ content.config.ts    # content collections
   ├─ middleware.ts
   └─ env.d.ts
```

**Bun install & lockfile.** Bun v1.2 changed the default lockfile format to the text-based `bun.lock`; commit it, and never keep a `package-lock.json` or `bun.lockb` alongside. To migrate a legacy binary lockfile, run `bun install --save-text-lockfile --frozen-lockfile --lockfile-only` and delete `bun.lockb`. Run `bun install --frozen-lockfile` in CI to fail on drift.

```toml
# bunfig.toml
[install]
# Bun defaults to the isolated linker for new workspaces; explicit here for clarity.
linker = "isolated"

[install.lockfile]
# Only "yarn" is supported as an extra printed format; usually omit this entirely.
# print = "yarn"

[test]
coverage = false
```

For monorepos use Bun workspaces with **catalogs** to pin one version of a shared dependency across packages:

```jsonc
// package.json (workspace root)
{
  "workspaces": ["packages/*"],
  "catalog": {
    "svelte": "^5.38.0",
    "astro": "^7.2.0"
  }
}
```

```jsonc
// packages/web/package.json
{ "dependencies": { "svelte": "catalog:", "astro": "catalog:" } }
```

Bun-native APIs worth using in scripts, endpoints, and tooling (not in hydrated island code, which runs in the browser): `Bun.file()` for zero-copy file reads, `Bun.$` for shell scripting, and `bun:sqlite` for a synchronous embedded DB.

```ts
// scripts/seed.ts — run with: bun scripts/seed.ts
import { Database } from "bun:sqlite";
import { $ } from "bun";

const db = new Database("data/app.sqlite", { create: true });
db.run("CREATE TABLE IF NOT EXISTS posts (id TEXT PRIMARY KEY, title TEXT)");

const raw = await Bun.file("data/posts.json").json();
const insert = db.prepare("INSERT OR REPLACE INTO posts (id, title) VALUES (?, ?)");
for (const p of raw) insert.run(p.id, p.title);

await $`echo "seeded ${raw.length} posts"`;
```

## Astro configuration & rendering modes

Use a typed `astro.config.mts`. Wire the Svelte integration and UnoCSS, and choose an adapter only when you need on-demand rendering.

```ts
// astro.config.mts
import { defineConfig, envField } from "astro/config";
import svelte from "@astrojs/svelte";
import node from "@astrojs/node";
import UnoCSS from "unocss/astro";

export default defineConfig({
  // "static" is the default. Use "server" only if most routes are on-demand.
  output: "static",
  adapter: node({ mode: "standalone" }), // required for actions, sessions, server islands
  integrations: [
    UnoCSS({ injectReset: true }), // injects @unocss/reset Tailwind reset
    svelte(),
  ],
  env: {
    schema: {
      PUBLIC_SITE_NAME: envField.string({ context: "client", access: "public", default: "Site" }),
      DATABASE_URL: envField.string({ context: "server", access: "secret" }),
      LOG_LEVEL: envField.enum({
        context: "server", access: "public",
        values: ["debug", "info", "warn", "error"], default: "info",
      }),
    },
  },
});
```

**Rendering model.** Astro prerenders every page to static HTML unless you opt a route into on-demand rendering. With `output: "static"` (the default), mark individual routes on-demand with `export const prerender = false`. With `output: "server"`, everything is on-demand and you opt *out* per route with `export const prerender = true`. Actions, sessions, and server islands need an adapter and an on-demand context.

```astro
---
// src/pages/dashboard.astro — on-demand in an otherwise static build
export const prerender = false;
---
```

Astro 7's queued rendering (~2.4× faster on expression-dense pages) and the Rust compiler are on by default — no config. Note the compiler is now JSX-strict: unclosed tags error instead of being auto-corrected, and whitespace between inline elements is collapsed; insert `{' '}` where you need a literal space.

### Advanced routing (`src/fetch.ts`)

Add `src/fetch.ts` only when you need full control of the request pipeline (auth before actions, a proxied API, Hono middleware). Omitting the file keeps Astro's default behavior.

```ts
// src/fetch.ts
import { astro, FetchState } from "astro/fetch";

export default {
  fetch(request: Request) {
    const state = new FetchState(request);
    if (state.url.pathname.startsWith("/api/legacy")) {
      const url = new URL(state.url.pathname, "https://backend.example.com");
      return fetch(new Request(url, request));
    }
    return astro(state);
  },
};
```

### Route caching

Route caching is stable in Astro 7. Configure a provider once, then set directives per response or declaratively via `routeRules`.

```ts
// astro.config.mts (excerpt)
import { defineConfig, memoryCache } from "astro/config";
export default defineConfig({
  cache: { provider: memoryCache() },
  routeRules: { "/blog/[...path]": { maxAge: 300, swr: 60 } },
});
```

## Content collections & the Content Layer API

Define collections in `src/content.config.ts` using loaders. Use the built-in `glob()` and `file()` loaders from `astro/loaders`; write a custom loader for remote sources. The legacy `type: 'content'` syntax is gone — always use `loader`.

```ts
// src/content.config.ts
import { defineCollection, z } from "astro:content";
import { glob, file } from "astro/loaders";

const blog = defineCollection({
  // deferRender defers Markdown rendering to page render time (lower build memory)
  loader: glob({ pattern: "**/*.md", base: "./src/content/blog", deferRender: true }),
  schema: ({ image }) =>
    z.object({
      title: z.string(),
      pubDate: z.coerce.date(),
      draft: z.boolean().default(false),
      cover: image().refine((img) => img.width >= 1200, {
        message: "Cover must be ≥1200px wide",
      }),
      tags: z.array(z.string()).default([]),
    }),
});

const authors = defineCollection({
  loader: file("src/data/authors.json"), // array of objects, each with a unique id
  schema: z.object({ id: z.string(), name: z.string(), url: z.string().url().optional() }),
});

export const collections = { blog, authors };
```

Query with the typed `astro:content` API. `getCollection` returns fully typed entries; `render()` produces the `<Content />` component.

```astro
---
// src/pages/blog/[...slug].astro
import { getCollection, render } from "astro:content";
import Layout from "../../layouts/Base.astro";

export async function getStaticPaths() {
  const posts = await getCollection("blog", ({ data }) => !data.draft);
  return posts.map((post) => ({ params: { slug: post.id }, props: { post } }));
}

const { post } = Astro.props;
const { Content } = await render(post);
---
<Layout title={post.data.title}>
  <article><Content /></article>
</Layout>
```

A custom remote loader is just an object with a `load()` method:

```ts
// src/loaders/products.ts
import type { Loader } from "astro/loaders";
export function productLoader(apiUrl: string): Loader {
  return {
    name: "product-loader",
    async load({ store, meta, logger, parseData }) {
      const res = await fetch(apiUrl);
      const products = await res.json();
      store.clear();
      for (const p of products) {
        const data = await parseData({ id: String(p.id), data: p });
        store.set({ id: data.id, data });
      }
      logger.info(`Loaded ${products.length} products`);
    },
  };
}
```

## Svelte 5 islands: props, hydration, and directives

A `.svelte` file becomes an island the moment you import it into an `.astro` file and add a `client:*` directive. Without a directive it renders to static HTML with no JavaScript shipped.

```astro
---
// src/pages/index.astro
import Counter from "../components/Counter.svelte";
import Search from "../components/Search.svelte";
---
<Counter client:load start={5} />        {/* hydrate immediately */}
<Search client:visible />                  {/* hydrate when scrolled into view */}
<Counter client:idle />                    {/* hydrate when main thread idle */}
<Counter client:media="(min-width: 768px)" /> {/* hydrate at breakpoint */}
<Counter client:only="svelte" />           {/* skip SSR, render only on client */}
```

| Directive | Hydrates | Use for |
| --- | --- | --- |
| `client:load` | Immediately on load | Above-the-fold, must-work-now widgets |
| `client:idle` | On `requestIdleCallback` | Low-priority interactivity |
| `client:visible` | On intersection | Below-the-fold islands (the default choice) |
| `client:media` | When media query matches | Responsive-only UI (mobile menu) |
| `client:only="svelte"` | Client only, no SSR HTML | Components that touch `window`/`localStorage` at init |

**Critical island rule: props must be serializable.** Astro serializes island props over the wire, so pass strings, numbers, booleans, plain objects, arrays, `Date`/`Map`/`Set` — **never functions or class instances**. Cross-island communication happens through shared state (nanostores), not callbacks.

### Runes in depth

Runes are compiler keywords (no import) usable in `.svelte`, `.svelte.ts`, and `.svelte.js` files. The five you use constantly are `$state`, `$derived`, `$effect`, `$props`, `$bindable`.

```svelte
<!-- src/components/Counter.svelte -->
<script lang="ts">
  interface Props { start?: number; step?: number; }
  let { start = 0, step = 1 }: Props = $props();

  let count = $state(start);
  let doubled = $derived(count * 2);
  // $derived.by for multi-statement derivations
  let parity = $derived.by(() => (count % 2 === 0 ? "even" : "odd"));

  // $effect is an escape hatch for side effects — NOT for syncing state
  $effect(() => {
    document.title = `Count: ${count}`;
    return () => { /* cleanup on teardown / before re-run */ };
  });
</script>

<button onclick={() => (count += step)}>
  {count} (×2 = {doubled}, {parity})
</button>
```

Rune reference and when to reach for each:

| Rune | Purpose | Version |
| --- | --- | --- |
| `$state` | Deep-reactive mutable value | 5.0 |
| `$state.raw` | Non-deep state (reassign only; cheaper for large objects/arrays) | 5.0 |
| `$state.snapshot` | Plain (non-proxied) copy for logging / passing to non-Svelte code | 5.0 |
| `$derived` / `$derived.by` | Computed values (always consistent, lazily recomputed) | 5.0 |
| `$effect` / `$effect.pre` / `$effect.root` | Side effects; pre runs before DOM update; root creates a manually disposed scope | 5.0 |
| `$props` / `$bindable` | Component inputs; `$bindable` enables two-way binding | 5.0 |
| `$props.id()` | Unique, SSR-stable per-instance id (for `for`/`aria-*`) | 5.20 |
| `$host` | The custom-element host (only in `customElement` compile mode) | 5.0 |
| `$inspect` | Dev-only reactive logging | 5.0 |

**The single sharpest gotcha:** do not use `$effect` to keep one piece of state in sync with another — use `$derived`. Reserve `$effect` for genuine side effects (network calls, manual DOM work, third-party libs). Also: plain class fields are not reactive; declare them with `$state`.

```ts
// src/lib/stores/cart.svelte.ts — reactive state shared across a Svelte island tree
class Cart {
  items = $state<{ id: string; price: number }[]>([]);
  total = $derived(this.items.reduce((s, i) => s + i.price, 0));
  add(item: { id: string; price: number }) { this.items.push(item); }
}
export const cart = new Cart();
```

### Snippets replace slots; events are attributes

Svelte 5 removed slots in favor of snippets rendered with `{@render}`, and event directives (`on:click`) in favor of event attributes (`onclick`). `children` is the default snippet.

```svelte
<!-- src/components/Card.svelte -->
<script lang="ts">
  import type { Snippet } from "svelte";
  interface Props {
    title: string;
    header?: Snippet;             // optional named snippet
    children: Snippet;            // default content
    row: Snippet<[{ id: string }]>; // parameterized snippet (tuple of args)
  }
  let { title, header, children, row }: Props = $props();
</script>

<section>
  {#if header}<header>{@render header()}</header>{/if}
  <h2>{title}</h2>
  {@render children()}
  {@render row({ id: "abc" })}
</section>
```

```svelte
<!-- consumer -->
<Card title="Reports">
  {#snippet header()}<strong>Q3</strong>{/snippet}
  {#snippet row(item)}<tr><td>{item.id}</td></tr>{/snippet}
  <p>Default children content.</p>
</Card>
```

Use `{@render children?.()}` with optional chaining when a snippet may be undefined, or an `{#if}`/`{:else}` block for fallback content.

### Attachments, keyed each, class/style directives

Attachments (`{@attach}`, available in Svelte 5.29 and newer) replace actions (`use:`) and are fully reactive — they re-run when their dependencies change and return a cleanup function.

```svelte
<script lang="ts">
  import tippy from "tippy.js";
  import type { Attachment } from "svelte/attachments";
  let text = $state("Hello");
  function tooltip(content: string): Attachment {
    return (node) => {
      const instance = tippy(node, { content });
      return () => instance.destroy();
    };
  }
</script>
<input bind:value={text} />
<button {@attach tooltip(text)}>Hover me</button>
```

Always key `{#each}` blocks over a stable id to preserve identity and state across reorders. Prefer `class:`/`style:` directives and the object/array `class` form over string concatenation:

```svelte
{#each items as item (item.id)}
  <li class:active={item.id === selected} style:--w={item.width}>{item.label}</li>
{/each}
```

### Mounting, typing, and stores vs runes

Instantiate components imperatively with `mount`/`unmount`, never `new Component`. Type a component reference with the `Component` type and snippets with `Snippet`.

```ts
import { mount, unmount, type Component } from "svelte";
import Widget from "./Widget.svelte";
const app = mount(Widget, { target: document.getElementById("app")!, props: { start: 3 } });
// later: unmount(app);
```

**Stores vs runes:** use runes (`$state` in `.svelte.ts`) for new shared state within a Svelte island tree. Svelte stores still work and remain useful for interop, but runes are the default. For state shared **across separate Astro islands** (different framework roots that cannot share a Svelte module scope reliably at runtime), use nanostores — see below.

## Cross-island state with nanostores

Astro islands are isolated; two `client:*` components do not share a module instance in a way you should rely on for app state. Astro's official recommendation for shared client-side storage is **nanostores**, which ships less than 1 KB of JS with zero dependencies and is framework-agnostic; pair it with `@nanostores/persistent` for `localStorage` sync across page navigations.

```ts
// src/lib/stores/cart.ts
import { atom, map, computed } from "nanostores";
export const isCartOpen = atom(false);
export const cartItems = map<Record<string, { qty: number; price: number }>>({});
export const total = computed(cartItems, (items) =>
  Object.values(items).reduce((s, i) => s + i.qty * i.price, 0),
);
export function addItem(id: string, price: number) {
  const current = cartItems.get()[id];
  cartItems.setKey(id, { qty: (current?.qty ?? 0) + 1, price });
}
```

Consume the store in a `.svelte` island with the `$` auto-subscription shorthand. Nanostores implement Svelte's store contract, so **there is no adapter package to install** and nothing to import but the store itself:

```svelte
<script lang="ts">
  import { total, isCartOpen } from "../lib/stores/cart";
</script>
{#if $isCartOpen}<aside>Total: ${$total}</aside>{/if}
```

`$` auto-subscription is a compiler feature of `.svelte` components, so it is unavailable in `.svelte.ts`/`.svelte.js` modules. Use `@nanostores/svelte-runes` there and read through `.current`; it is built on `createSubscriber`, so one subscription is shared per store and SSR stays correct.

```ts
// src/lib/stores/cart-view.svelte.ts
import { useStore } from "@nanostores/svelte-runes";
import { total } from "./cart";

const totalView = useStore(total);
export function formatted(): string {
  return `$${totalView.current}`;
}
```

There is no `@nanostores/svelte` package — the npm registry has never carried one. The framework adapters (`@nanostores/react`, `/preact`, `/vue`, `/solid`, `/lit`, `/angular`, `/alpine`) exist for frameworks with no equivalent of Svelte's store contract; Svelte needs none.

## Astro Actions: type-safe server functions

Define server logic in `src/actions/index.ts` with `defineAction` and a Zod schema from `astro:schema`. Actions work from HTML forms without client JS and from typed client calls. `accept: "form"` parses `FormData`.

```ts
// src/actions/index.ts
import { defineAction, ActionError } from "astro:actions";
import { z } from "astro:schema";

export const server = {
  submitFeedback: defineAction({
    accept: "form",
    input: z.object({
      email: z.string().email(),
      message: z.string().min(10),
    }),
    handler: async ({ email, message }, context) => {
      if (message.includes("spam")) {
        throw new ActionError({ code: "BAD_REQUEST", message: "Rejected." });
      }
      // context has cookies, locals, request — but not getActionResult/redirect
      return { ok: true, email };
    },
  }),
};
```

Handle the no-JS form path with `Astro.getActionResult()` and narrow validation failures with `isInputError()`. Action results persist via a single-use cookie: `getActionResult()` returns a value only on the first request after submission.

```astro
---
// src/pages/feedback.astro
export const prerender = false;
import { actions, isInputError } from "astro:actions";
const result = Astro.getActionResult(actions.submitFeedback);
if (result && !result.error) return Astro.redirect("/thanks"); // POST/redirect/GET
const fieldErrors = isInputError(result?.error) ? result.error.fields : {};
---
<form method="POST" action={actions.submitFeedback}>
  <input type="email" name="email" required />
  {fieldErrors.email && <p class="text-red-500">{fieldErrors.email[0]}</p>}
  <textarea name="message"></textarea>
  {fieldErrors.message && <p class="text-red-500">{fieldErrors.message[0]}</p>}
  <button>Send</button>
</form>
```

From a Svelte island, call the action and destructure `{ data, error }` (or use `actions.name.orThrow()`):

```svelte
<script lang="ts">
  import { actions } from "astro:actions";
  let email = $state("");
  let message = $state("");
  async function save() {
    const { data, error } = await actions.submitFeedback({ email, message });
    if (error) console.error(error);
    else console.log(data.ok);
  }
</script>
```

## Middleware, locals, and environment

Middleware lives in `src/middleware.ts`. Use `defineMiddleware`, `sequence` to compose, and `context.locals` to pass typed data to pages. Type `App.Locals` in `env.d.ts`.

```ts
// src/middleware.ts
import { defineMiddleware, sequence } from "astro:middleware";

const auth = defineMiddleware(async (context, next) => {
  const token = context.cookies.get("session")?.value;
  context.locals.user = token ? await lookupUser(token) : null;
  return next();
});

const timing = defineMiddleware(async (context, next) => {
  const start = performance.now();
  const res = await next();
  res.headers.set("Server-Timing", `total;dur=${performance.now() - start}`);
  return res;
});

export const onRequest = sequence(auth, timing);
async function lookupUser(_t: string) { return { id: "1", name: "Ada" }; }
```

```ts
// src/env.d.ts
declare namespace App {
  interface Locals {
    user: { id: string; name: string } | null;
  }
}
```

Use `astro:env` for typed environment variables — import public client vars from `astro:env/client` and server secrets from `astro:env/server`. Prefer this over `import.meta.env` and `process.env` for anything in the schema; a missing required var fails loudly instead of being silently `undefined`.

```astro
---
import { DATABASE_URL } from "astro:env/server";
import { PUBLIC_SITE_NAME } from "astro:env/client";
---
<title>{PUBLIC_SITE_NAME}</title>
```

`Astro.rewrite("/other")` serves another route without a client redirect; use it for internal fallbacks and localized content.

## View transitions & image optimization

The client-side router component is `<ClientRouter />` from `astro:transitions` (renamed from `<ViewTransitions />`, which was removed in Astro 6). Put it in your `<head>` to get SPA-like navigation over MPA pages.

```astro
---
// src/layouts/Base.astro
import { ClientRouter } from "astro:transitions";
---
<html>
  <head><ClientRouter /></head>
  <body>
    <slot />
  </body>
</html>
```

Scripts run only on initial load; re-run per navigation by listening for `astro:page-load`. Preserve element state with `transition:persist` and animate with `transition:animate`.

Optimize images with `astro:assets`. Use `<Image>` for a single optimized image and `<Picture>` for multiple formats; import local images so Astro can infer dimensions and prevent layout shift.

```astro
---
import { Image, Picture } from "astro:assets";
import hero from "../assets/hero.png";
---
<Image src={hero} alt="Hero" width={1200} height={630} loading="eager" />
<Picture src={hero} formats={["avif", "webp"]} alt="Hero" />
```

## Server islands

`server:defer` renders an Astro component on the server at request time while the rest of the page is cached/static. Provide fallback content with `slot="fallback"`. Props must be serializable — functions cannot be passed.

```astro
---
import UserGreeting from "../components/UserGreeting.astro";
---
<UserGreeting server:defer userId={Astro.locals.user?.id}>
  <p slot="fallback">Loading…</p>
</UserGreeting>
```

Server islands require an adapter. Note a known interaction: combining server islands with `<ClientRouter />` has had head-element and hydration edge cases fixed across 7.x point releases — keep Astro current if you use both together.

## UnoCSS configuration

UnoCSS is the styling engine — there is no `tailwind.config.js`, no PostCSS, no `@apply`. All configuration lives in `uno.config.ts`, and the Astro integration is `unocss/astro`. Use **`presetWind4`** (the Tailwind v4-compatible preset). `presetUno`/`presetWind3` are the older generation; only fall back to `presetWind3` for compatibility with tooling that hasn't caught up.

```ts
// uno.config.ts
import { defineConfig, presetWind4, presetIcons } from "unocss";
import { presetShadcn } from "unocss-preset-shadcn";
import presetAnimations from "unocss-preset-animations";
import transformerDirectives from "@unocss/transformer-directives";
import transformerVariantGroup from "@unocss/transformer-variant-group";

export default defineConfig({
  presets: [
    presetWind4(),
    presetAnimations(),
    presetShadcn({ color: "zinc" }), // shadcn theme CSS variables
    presetIcons({ scale: 1.2, warn: true }),
  ],
  transformers: [transformerDirectives(), transformerVariantGroup()],
  shortcuts: {
    "btn": "px-4 py-2 rounded bg-primary text-primary-foreground hover:opacity-90",
    "card": "rounded-lg border bg-card p-6 shadow-sm",
  },
  theme: {
    colors: { brand: { DEFAULT: "#4f46e5", muted: "#818cf8" } },
  },
  // CRITICAL for Astro + Svelte islands + shadcn-svelte:
  content: {
    pipeline: {
      include: [
        // default extensions (already covers .astro and .svelte)
        /\.(vue|svelte|[jt]sx|mdx?|astro|elm|php|phtml|html)($|\?)/,
        // .ts/.js are NOT scanned by default — needed for tailwind-variants defs
        "(components|src)/**/*.{js,ts}",
      ],
    },
  },
});
```

**Critical insight:** UnoCSS scans `.astro`, `.svelte`, `.jsx`, `.html` and similar by default but **excludes `.ts`/`.js`**. shadcn-svelte defines component variants (via `tailwind-variants`) in `.ts` files, so without the explicit `content.pipeline.include` glob those utility classes silently never get generated. This is the single most common "styles missing on my island" bug on this stack. The magic comment `// @unocss-include` at the top of a specific file is a per-file alternative.

Bring in the browser reset either through the integration (`injectReset: true`, shown earlier) or by importing `@unocss/reset/tailwind.css` in your base layout. `presetWind4` also aligns its own preflight with Tailwind v4, so avoid double resets.

`transformerDirectives` enables `@apply`/`--at-apply` and `theme()` inside `<style>` blocks; `transformerVariantGroup` enables `hover:(bg-white text-black)` grouping. Use `attributify` only if the team wants attribute-mode utilities; it is optional. Populate `safelist` for classes generated only at runtime (e.g. from CMS data) that the static scanner can't see.

## shadcn-svelte with UnoCSS

shadcn-svelte components are copied into your repo (you own the code); they are built on **Bits UI** for accessible behavior (focus management, keyboard handling, ARIA) and styled with utility classes plus the `cn()` helper (`clsx` + `tailwind-merge`). They live in `src/lib/components/ui` (or `src/components/ui`).

```ts
// src/lib/utils.ts
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```jsonc
// components.json
{
  "$schema": "https://shadcn-svelte.com/schema.json",
  "style": "default",
  "tailwind": { "config": "", "css": "src/styles/app.css", "baseColor": "zinc" },
  "aliases": {
    "components": "$lib/components",
    "utils": "$lib/utils",
    "ui": "$lib/components/ui",
    "hooks": "$lib/hooks"
  }
}
```

**The central integration reality:** shadcn-svelte 1.x's CLI (`bunx shadcn-svelte@latest init`) defaults to **Tailwind v4** (OKLCH `@theme` variables, `@tailwindcss/vite`, no config file). Because you are running **UnoCSS as the CSS engine, not Tailwind**, do not use the CLI's Tailwind setup as-is. `unocss-preset-shadcn` (1.0.1) generates the shadcn theme CSS variables for you. In practice:

- Add `presetShadcn({ color: "zinc" })` to `uno.config.ts` — it emits the `--background`, `--foreground`, `--primary`, etc. variables the components consume. Note the required no-comma HSL variable format (`--background: 0 0% 100%;`, not `0,0%,100%`).
- Hand-place `utils.ts` and `components.json` rather than relying on the CLI's Tailwind-v4 output; keep an empty `tailwind.config.js` at the root only if a CLI command complains.
- Provide a dark-mode toggle that flips the `.dark` class on `<html>` (the preset's default `darkSelector`). Only SolidUI needs `darkSelector: '[data-kb-theme="dark"]'`.
- For dynamic themes, pass an array from `builtinColors` to `presetShadcn` and add a theme-sync script.
- `unocss-preset-shadcn` 1.0.1 targets `presetWind4` by default; if you must use the older engine, import the legacy build from `unocss-preset-shadcn/v3` alongside `presetWind3`. The preset's README predates shadcn-svelte 1.5's Tailwind-v4 defaults, so components pulled directly from the 1.5 registry (OKLCH, `@theme inline`, `tw-animate-css`) may need manual token reconciliation — verify against a build.

Use a shadcn-svelte component inside an Astro island exactly like any Svelte island (add a `client:*` directive on the wrapping component):

```astro
---
import ThemeToggle from "../components/ThemeToggle.svelte";
---
<ThemeToggle client:load />
```

## TypeScript configuration

Extend Astro's strict preset and enable the modern module and erasability flags. `astro check` (which runs the Svelte language tools for `.svelte` files) is your source of truth for type errors across `.astro`, `.svelte`, and `.ts`.

```jsonc
// tsconfig.json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "verbatimModuleSyntax": true,     // requires explicit `import type`
    "erasableSyntaxOnly": true,       // bans enum/namespace/param-props (TS 5.8+)
    "moduleResolution": "bundler",
    "module": "esnext",
    "isolatedModules": true,
    "noUncheckedIndexedAccess": true,
    "types": ["astro/client"]
  },
  "include": [".astro/types.d.ts", "src/**/*"],
  "exclude": ["dist"]
}
```

`erasableSyntaxOnly` (TypeScript 5.8) makes the compiler error on TypeScript-specific constructs that have runtime behavior (`enum`, runtime `namespace`, constructor parameter properties, `import =`), which keeps your source strippable — combine it with `verbatimModuleSyntax`. With `verbatimModuleSyntax`, always separate type-only imports:

```ts
import type { Snippet } from "svelte";
import { mount } from "svelte";
```

Type Astro component props with an interface in the frontmatter, and use `satisfies` to keep literal types while checking a shape:

```astro
---
interface Props { title: string; count?: number; }
const { title, count = 0 } = Astro.props;

const config = { theme: "dark", cols: 3 } satisfies Record<string, unknown>;
---
```

Run `astro check` for full-project type checking (it covers `.astro` and `.svelte`); reserve a standalone `svelte-check` call only for Svelte-only packages without Astro.

## Testing with `bun test`

Use Bun's built-in Jest-compatible runner (`bun:test`) for unit, logic, and store tests — it's fast and needs no config. Test framework-agnostic logic (nanostores, `.svelte.ts` pure functions, action handlers) directly.

```ts
// src/lib/stores/cart.test.ts
import { test, expect, beforeEach, spyOn, mock } from "bun:test";
import { cartItems, total, addItem } from "./cart";

beforeEach(() => cartItems.set({}));

test("addItem accumulates quantity", () => {
  addItem("a", 10);
  addItem("a", 10);
  expect(cartItems.get().a.qty).toBe(2);
  expect(total.get()).toBe(20);
});

test("spies track calls", () => {
  const obj = { save: (_x: number) => {} };
  const spy = spyOn(obj, "save");
  obj.save(1);
  expect(spy).toHaveBeenCalledTimes(1);
  expect(spy.mock.calls).toEqual([[1]]);
});

test("snapshot", () => {
  expect({ ok: true, items: 2 }).toMatchSnapshot();
});
```

`mock.module()` stubs an entire module; `mock.restore()` restores. Update snapshots with `bun test --update-snapshots`.

For **rendered Svelte component** testing, `bun test` alone does not provide a DOM. Use `@testing-library/svelte` with a DOM environment; because Astro/Svelte's toolchain is Vite-based, **Vitest** is the pragmatic choice for component-DOM and Astro Container API tests, while `bun test` handles fast pure-logic suites. Test `.astro` components by rendering them through Astro's Container API:

```ts
// component render via Astro Container API (run under Vitest)
import { experimental_AstroContainer as AstroContainer } from "astro/container";
import Card from "../src/components/Card.astro";
import { expect, test } from "vitest";

test("renders card title", async () => {
  const container = await AstroContainer.create();
  const html = await container.renderToString(Card, { props: { title: "Hi" } });
  expect(html).toContain("Hi");
});
```

| Test kind | Tool |
| --- | --- |
| Pure logic, stores, action handlers | `bun test` |
| Svelte component + DOM | Vitest + `@testing-library/svelte` |
| `.astro` component output | Vitest + Astro Container API |

## Biome: formatting, linting, imports

Biome is the formatter and linter (replacing Prettier + ESLint for JS/TS/JSON/CSS). Since v2.3 it can format and lint the JS/TS in `<script>` and CSS in `<style>` of `.astro`/`.svelte` files, but framework support is **experimental and opt-in**. Biome cannot type-check or deeply lint Svelte template control-flow — pair it with `astro check` (types) for full coverage.

```jsonc
// biome.json
{
  "$schema": "https://biomejs.dev/schemas/2.5.1/schema.json",
  "vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true },
  "files": { "ignoreUnknown": true },
  "formatter": { "enabled": true, "indentStyle": "space", "indentWidth": 2, "lineWidth": 100 },
  "assist": { "actions": { "source": { "organizeImports": "on" } } },
  "linter": { "enabled": true, "rules": { "recommended": true } },
  "html": { "formatter": { "enabled": true } },
  "overrides": [
    {
      "includes": ["**/*.svelte", "**/*.astro"],
      "linter": {
        "rules": {
          "style": { "useConst": "off", "useImportType": "off" },
          "correctness": { "noUnusedVariables": "off", "noUnusedImports": "off" }
        }
      }
    }
  ]
}
```

Turning off `useConst`, `useImportType`, `noUnusedVariables`, and `noUnusedImports` for `.svelte`/`.astro` prevents false positives from Biome's partial understanding of these embedded languages. To opt into deeper HTML-ish parsing (Svelte `{#if}` control flow, `:global`), enable `html.experimentalFullHtmlSupportEnabled` — treat it as experimental.

Commands:

```bash
biome check --write .   # format + lint + organize imports, applying safe fixes
biome format --write .  # format only
biome lint .            # lint only
biome ci .              # CI mode (no writes, non-zero on issues)
```

Biome's `assist` action organizes imports; do not also run a separate import-sorter. Biome does **not** replace `astro check`/`svelte-check` for type errors — run both in `check`.

## Deployment adapters

Choose an adapter by target: `@astrojs/node` (self-hosted/Docker, `mode: "standalone"` or `"middleware"`), `@astrojs/cloudflare`, `@astrojs/vercel`, or `@astrojs/netlify`. An adapter is required for on-demand rendering, actions, sessions, and server islands. Astro 7 adds experimental CDN cache providers (`cacheNetlify()`, `cacheVercel()`, `cacheCloudflare()`) that push route-cache directives to the edge.

## Anti-patterns to avoid

| Wrong (adjacent-ecosystem habit) | Right (this stack) |
| --- | --- |
| `export let foo` / `on:click` / `$:` | `let { foo } = $props()` / `onclick` / `$derived` |
| `$effect` to copy one state into another | `$derived` / `$derived.by` |
| `new Component({ target })` | `mount(Component, { target })` |
| Passing a callback function as an island prop | Share state via nanostores; keep props serializable |
| Expecting two `client:*` islands to share a module singleton | Cross-island state through nanostores |
| `import { useStore } from "@nanostores/svelte"` | No such package: `$store` in `.svelte`, `@nanostores/svelte-runes` (`.current`) in `.svelte.ts` |
| Adding a `tailwind.config.js` + PostCSS + `@apply` in CSS | Configure `uno.config.ts`; use `transformerDirectives` for `@apply` |
| Relying on default UnoCSS scan for `.ts` variant files | Add `content.pipeline.include` glob for `(components\|src)/**/*.{js,ts}` |
| Running `shadcn-svelte init` and getting Tailwind v4 wired in | Use `presetShadcn` in UnoCSS; hand-place `utils.ts`/`components.json` |
| `npm install` / committing `package-lock.json` | `bun install`; commit `bun.lock` |
| `process.env.SECRET` / bare `import.meta.env` for schema vars | `astro:env/server` and `astro:env/client` |
| `type: 'content'` in a collection | `loader: glob(...)` / `file(...)` from `astro/loaders` |
| `<ViewTransitions />` | `<ClientRouter />` from `astro:transitions` |
| Trusting Biome to catch Svelte template type errors | Run `astro check` for types alongside Biome |
| `client:load` on everything | Default to `client:visible`; reserve `client:load` for above-the-fold |
| Passing functions to `server:defer` props | Serializable props only; fetch inside the island |
