# agents.md — Blog tecnico con Astro

## Istruzioni per l'agente

1. Leggi questo file per intero prima di scrivere qualsiasi codice.
2. Non inventare dipendenze non elencate nella sezione configurazione.
3. Usa TypeScript ovunque sia supportato da Astro.
4. Il sito deve passare `astro build` senza errori.
5. Non aggiungere animazioni JavaScript lato client a meno che non sia esplicitamente richiesto.
6. **Crea `log.md` nella root del progetto** e aggiornalo dopo ogni operazione significativa (file creato, dipendenza installata, errore risolto).
7. **Crea `context.md` nella root del progetto** con un riepilogo delle decisioni prese, delle convenzioni adottate e dello stato attuale del progetto. Aggiornalo ogni volta che completi una sezione.

### Formato log.md

```markdown
# Log

## [data e ora]
- Azione eseguita
- Eventuale errore e come è stato risolto
```

### Formato context.md

```markdown
# Contesto progetto

## Stato attuale
[cosa è stato completato, cosa manca]

## Decisioni prese
[scelte fatte e motivazione]

## Convenzioni adottate
[naming, struttura, pattern usati]
```

---

## Obiettivo

Costruire un blog tecnico personale ispirato a [jimmybogard.com](https://www.jimmybogard.com/).
Il sito deve essere completamente statico, deployabile gratuitamente, e ottimizzato per la lettura di articoli tecnici lunghi.

---

## Stack

| Layer | Tecnologia |
|---|---|
| Framework | **Astro** v4+ |
| Contenuto | File Markdown con frontmatter |
| Styling | **Tailwind CSS** v3 |
| Syntax highlighting | Shiki (integrato in Astro) |
| Ricerca | **Pagefind** (indicizzazione a build time) |
| Deploy | **Cloudflare Pages** (piano free) |
| CI | GitHub Actions |

---

## Struttura delle cartelle

```
/
├── src/
│   ├── components/
│   │   ├── Header.astro
│   │   ├── Footer.astro
│   │   ├── PostCard.astro
│   │   ├── TagBadge.astro
│   │   ├── Pagination.astro
│   │   └── Search.astro
│   ├── layouts/
│   │   ├── BaseLayout.astro
│   │   └── PostLayout.astro
│   ├── pages/
│   │   ├── index.astro
│   │   ├── about.astro
│   │   ├── search.astro
│   │   ├── [slug].astro
│   │   ├── rss.xml.ts
│   │   ├── 404.astro
│   │   └── tag/
│   │       └── [tag].astro
│   ├── content/
│   │   ├── config.ts
│   │   └── posts/
│   │       └── *.md
│   └── utils/
│       ├── reading-time.ts
│       └── format-date.ts
├── public/
│   └── favicon.svg
├── .github/
│   └── workflows/
│       └── deploy.yml
├── astro.config.mjs
├── tailwind.config.mjs
├── tsconfig.json
├── package.json
├── log.md          ← da creare e aggiornare
└── context.md      ← da creare e aggiornare
```

---

## Design system

### Palette (Tailwind default, nessun colore custom)

| Ruolo | Classe Tailwind |
|---|---|
| Sfondo pagina | `bg-white dark:bg-zinc-950` |
| Testo principale | `text-zinc-900 dark:text-zinc-100` |
| Testo secondario | `text-zinc-500 dark:text-zinc-400` |
| Titoli principali | `text-green-600 dark:text-green-400` |
| Accento / link | `text-blue-600 dark:text-blue-400` |
| Bordo sottile | `border-zinc-200 dark:border-zinc-800` |
| Sfondo code block | `bg-zinc-950` (sempre scuro) |

### Tipografia

Font di sistema, nessuna richiesta esterna:
```css
font-family: ui-sans-serif, system-ui, -apple-system, sans-serif;
```

| Elemento | Classe |
|---|---|
| Titolo blog (header) | `text-xl font-bold text-green-600 dark:text-green-400` |
| h1 pagina post | `text-3xl font-bold text-green-600 dark:text-green-400` |
| Titolo card post | `text-xl font-semibold text-green-600 dark:text-green-400` |
| Corpo del testo | `text-base leading-7` |
| Meta (data, min lettura) | `text-sm text-zinc-500` |
| Tag badge | `text-xs font-medium` |

### Layout

- Larghezza massima: `max-w-2xl` centrato con `mx-auto`
- Padding: `px-4 sm:px-6`
- Nessuna sidebar, lista post verticale

### Dark mode

`darkMode: 'class'` in Tailwind. Script inline in `<head>` per evitare flash:

```html
<script is:inline>
  const theme = localStorage.getItem('theme');
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  if (theme === 'dark' || (!theme && prefersDark)) {
    document.documentElement.classList.add('dark');
  }
</script>
```

Toggle nel Header salva la preferenza in `localStorage`.

---

## Schema dei contenuti

### Frontmatter

```yaml
---
title: "Titolo del post"
date: 2026-04-20
description: "Max 160 caratteri"
tags: ["tag1", "tag2"]
draft: false
---
```

| Campo | Tipo | Obbligatorio |
|---|---|---|
| `title` | string | sì |
| `date` | YYYY-MM-DD | sì |
| `description` | string max 160 | sì |
| `tags` | string[] min 1 | sì |
| `draft` | boolean | no (default false) |
| `cover` | string (path) | no |

### Schema Zod — `src/content/config.ts`

```typescript
import { defineCollection, z } from 'astro:content';

const posts = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    date: z.coerce.date(),
    description: z.string().max(160),
    tags: z.array(z.string()).min(1),
    draft: z.boolean().default(false),
    cover: z.string().optional(),
  }),
});

export const collections = { posts };
```

### Regole

- Ordinamento: sempre per `date` discendente
- Draft esclusi in produzione (`import.meta.env.PROD`), visibili in dev
- Slug = nome file senza estensione
- Crea 3 post di esempio con contenuto tecnico realistico (min 300 parole ciascuno)

---

## Utility

### `src/utils/reading-time.ts`

```typescript
export function readingTime(body: string): string {
  const words = body.trim().split(/\s+/).length;
  const minutes = Math.ceil(words / 200);
  return `${minutes} min read`;
}
```

### `src/utils/format-date.ts`

```typescript
export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('en-GB', {
    day: 'numeric',
    month: 'long',
    year: 'numeric',
  }).format(date);
}
```

---

## Pagine

### Homepage — `src/pages/index.astro` → `/`

- Lista PostCard verticale, 10 post per pagina
- Paginazione: pagina 1 = `/`, pagina 2 = `/page/2`
- `getStaticPaths` per la paginazione
- Esclude draft in produzione

### Singolo post — `src/pages/[slug].astro` → `/nome-post`

- `getStaticPaths` per ogni post non-draft
- Tag badges cliccabili in cima
- Data formattata + minuti di lettura
- Corpo con classe `prose` (Tailwind Typography)
- Syntax highlighting Shiki tema `github-dark`
- Meta tag Open Graph completi

### About — `src/pages/about.astro` → `/about`

- Pagina statica con `prose`
- Placeholder: chi è l'autore, di cosa parla il blog, link social

### Archivio tag — `src/pages/tag/[tag].astro` → `/tag/nome-tag`

- `getStaticPaths` per ogni tag unico
- Lista completa post con quel tag, senza paginazione
- Ordinati per data discendente

### Ricerca — `src/pages/search.astro` → `/search`

- Pagina che monta l'UI di Pagefind
- Importa il CSS e il JS di Pagefind dal path `/pagefind/pagefind-ui.js`
- Usa il Web Component `<div id="search">` + inizializzazione script
- Pagefind viene generato da `npx pagefind --site dist` dopo il build

```html
<!-- src/pages/search.astro -->
<link href="/pagefind/pagefind-ui.css" rel="stylesheet" />
<div id="search"></div>
<script>
  window.addEventListener('DOMContentLoaded', () => {
    new PagefindUI({ element: '#search', showSubResults: true });
  });
</script>
<script src="/pagefind/pagefind-ui.js"></script>
```

### Feed RSS — `src/pages/rss.xml.ts` → `/rss.xml`

Usa `@astrojs/rss`, include tutti i post non-draft, ordinati per data.

### 404 — `src/pages/404.astro`

Testo "Pagina non trovata" + link alla homepage.

---

## Componenti

### `src/layouts/BaseLayout.astro`

```typescript
interface Props {
  title: string;
  description: string;
  ogType?: 'website' | 'article';
  publishedTime?: string;
}
```

Contiene: `<html>`, `<head>` con meta + OG + RSS autodiscovery, script dark mode inline, `<body>` con `max-w-2xl`, `<Header />`, `<slot />`, `<Footer />`.

### `src/layouts/PostLayout.astro`

```typescript
interface Props {
  title: string;
  description: string;
  date: Date;
  tags: string[];
  readingTime: string;
}
```

Estende BaseLayout con `ogType="article"` e `publishedTime`.

### `src/components/Header.astro`

```
[Nome Blog]          [About] [Ricerca] [RSS] [Toggle dark/light]
```

- Nome blog: link `/`, `text-xl font-bold`
- Link: About `/about`, Ricerca `/search`, RSS `/rss.xml`
- Toggle: script che fa toggle della classe `dark` su `<html>` e salva in localStorage
- `<hr>` sotto la nav

### `src/components/Footer.astro`

```html
<footer class="mt-16 pt-8 border-t border-zinc-200 dark:border-zinc-800 text-sm text-zinc-500 text-center">
  © {anno} Nome Autore · <a href="https://astro.build">Powered by Astro</a>
</footer>
```

### `src/components/PostCard.astro`

```typescript
interface Props {
  title: string;
  slug: string;
  date: Date;
  description: string;
  tags: string[];
  readingTime: string;
}
```

Output: `<article>` con tag badges, titolo linkato, descrizione, data + minuti di lettura. Separatore `border-b` tra card.

### `src/components/TagBadge.astro`

```typescript
interface Props { tag: string; }
```

Pillola: `rounded-full px-2 py-0.5 text-xs font-medium`, link a `/tag/[tag]`.

### `src/components/Pagination.astro`

```typescript
interface Props {
  currentPage: number;
  totalPages: number;
  prevUrl: string | undefined;
  nextUrl: string | undefined;
}
```

Nav con link "← Post più recenti" e "Post più vecchi →", label "Pagina X di Y".

---

## Configurazione

### `package.json`

```json
{
  "name": "blog",
  "type": "module",
  "scripts": {
    "dev": "astro dev",
    "build": "astro build && npx pagefind --site dist",
    "preview": "astro preview"
  },
  "dependencies": {
    "astro": "^4.0.0",
    "@astrojs/tailwind": "^5.0.0",
    "@astrojs/rss": "^4.0.0",
    "@astrojs/sitemap": "^3.0.0",
    "@tailwindcss/typography": "^0.5.0",
    "tailwindcss": "^3.4.0",
    "pagefind": "^1.0.0"
  }
}
```

### `astro.config.mjs`

```javascript
import { defineConfig } from 'astro/config';
import tailwind from '@astrojs/tailwind';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://tuoblog.pages.dev',
  integrations: [tailwind(), sitemap()],
  markdown: {
    shikiConfig: {
      theme: 'github-dark',
      wrap: true,
    },
  },
});
```

### `tailwind.config.mjs`

```javascript
export default {
  content: ['./src/**/*.{astro,html,js,jsx,md,mdx,ts,tsx}'],
  darkMode: 'class',
  theme: { extend: {} },
  plugins: [require('@tailwindcss/typography')],
};
```

### `tsconfig.json`

```json
{
  "extends": "astro/tsconfigs/strict",
  "compilerOptions": {
    "strictNullChecks": true,
    "baseUrl": ".",
    "paths": {
      "@components/*": ["src/components/*"],
      "@layouts/*": ["src/layouts/*"],
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

### `.gitignore`

```
node_modules/
dist/
.astro/
.env
.env.*
!.env.example
```

### `.github/workflows/deploy.yml`

```yaml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: blog
          directory: dist
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

Secrets da aggiungere nelle impostazioni GitHub del repo:
- `CLOUDFLARE_API_TOKEN` — permesso "Cloudflare Pages: Edit"
- `CLOUDFLARE_ACCOUNT_ID` — ID account dalla dashboard Cloudflare

### `public/favicon.svg`

SVG semplice 32×32, sfondo scuro, iniziali o icona geometrica bianca.
