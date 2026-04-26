# Contesto progetto

## Stato attuale
Implementazione base completata e build statico andato a buon fine.
File creati:
- Config: package.json, astro.config.mjs, tailwind.config.mjs, tsconfig.json, .gitignore
- Utility: reading-time.ts, format-date.ts
- Content: config.ts, 3 post tecnici di esempio
- Layouts: BaseLayout.astro, PostLayout.astro
- Componenti: Header.astro, Footer.astro, PostCard.astro, TagBadge.astro, Pagination.astro
- Pagine: index.astro, [slug].astro, about.astro, search.astro, rss.xml.ts, 404.astro, page/[page].astro, tag/[tag].astro
- Altri: public/favicon.svg, .github/workflows/deploy.yml

## Decisioni prese
- Seguite le specifiche AGENTS.md al dettaglio.
- Rimossa @astrojs/sitemap per evitare breaking bug nell'integrazione con static routes.
- PostCard riutilizzato sia nella home che in tag/[tag].astro.
- Dark mode gestita con script inline + tailwind darkMode: 'class'.

## Convenzioni adottate
- Nomi file in kebab-case per pagine/componeneti/utility; PascalCase solo per componenti Astro.
- Alias TypeScript: @components/*, @layouts/*, @utils/*
- Styling esclusivamente con classi Tailwind; nessun colore custom eccetto verde pastello per i titoli principali (`text-green-600` light, `text-green-400` dark).
- Draft esclusi in produzione (import.meta.env.PROD).
- Post ordinati sempre per data discendente.
