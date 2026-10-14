# Hello World Performance Comparison

The same "Hello World" page built five ways, to compare what each stack ships to the browser.

Every app renders one `<h1>Hello World</h1>` and nothing else. Any byte above the vanilla baseline is framework overhead.

## Apps

| Folder | Stack | Rendering | Build output |
|---|---|---|---|
| [`my-vanilla-app`](my-vanilla-app) | Plain HTML | Static file | `out/` |
| [`my-react-app`](my-react-app) | React 19.3 + Vite 8.3 | Client-side render (SPA) | `dist/` |
| [`my-next-app`](my-next-app) | Next.js 16.4 (App Router, Turbopack) | Server (`next start`), prerendered | `.next/` |
| [`my-next-static-app`](my-next-static-app) | Next.js 16.4 (`output: "export"`) | Static export | `out/` |
| [`hello-streak-app`](hello-streak-app) | Streak.js (`streak-forge` 4.1.11-beta.0) | Pre-rendered static HTML, zero framework JS | `out/` |

## Results: first page load

Bytes the browser downloads to show `/`. Measured from the production build output. Gzip is level 9.

| App | Requests | HTML (raw / gzip) | JS (raw / gzip) | **Total gzip** | vs. vanilla |
|---|---:|---:|---:|---:|---:|
| Vanilla HTML | 1 | 66 B / 64 B | 0 | **64 B** | 1x |
| Streak.js | 2 | 2.8 KB / 1.3 KB | 1.7 KB / 1.0 KB | **2.2 KB** | ~35x |
| React + Vite | 2 | 388 B / 268 B | 214 KB / 66.1 KB | **66.4 KB** | ~1,060x |
| Next.js (static export) | 6 | 4.5 KB / 1.6 KB | 446 KB / 131.6 KB | **133.3 KB** | ~2,130x |
| Next.js (server) | 6 | 4.5 KB / 1.6 KB | 446 KB / 131.6 KB | **133.3 KB** | ~2,130x |

![First-load size](https://img.shields.io/badge/Streak.js-2.2%20KB-brightgreen) ![First-load size](https://img.shields.io/badge/React%20%2B%20Vite-66%20KB-yellow) ![First-load size](https://img.shields.io/badge/Next.js-133%20KB-red)

Notes:

- Favicons and the `noModule` legacy chunk in Next.js are not counted. Modern browsers skip `noModule` scripts.
- The Streak.js HTML includes about 2.4 KB of inlined loader script. The total above already counts it.
- Streak.js also builds `motion.js` (81 KB, 28 KB gzip) and an asset worker. These load on demand only, not on first paint.
- Both Next.js builds ship the same client chunks. Server mode adds a Node.js process but no extra browser bytes for this page.

## What drives the difference

| Factor | Vanilla | Streak.js | React + Vite | Next.js |
|---|---|---|---|---|
| Content in initial HTML | Yes | Yes | No (empty `#root`) | Yes |
| Framework runtime in browser | None | None | React + ReactDOM | React + ReactDOM + router + RSC runtime |
| Hydration | None | None | Full client render | Full hydration |
| JS needed to see text | No | No | Yes | No (but JS still downloads and runs) |
| Server needed | Any static host | Any static host | Any static host | Static host (export) or Node.js (server) |

Key points:

- **React + Vite** sends an empty page. The text appears only after 66 KB of JS downloads, parses and runs. This hurts First Contentful Paint (FCP) and Largest Contentful Paint (LCP) on slow devices.
- **Next.js** sends the text in HTML, so FCP is fast. It then downloads 133 KB of JS to hydrate a page that has no interactivity. This costs main-thread time and Total Blocking Time (TBT).
- **Streak.js** sends the text in HTML and a 1 KB helper script. It does not ship React to the browser. Result is close to the vanilla baseline.

## Reproduce

Prerequisites: Node.js 20+, Bun 1.2+, Python 3.

```bash
# React + Vite
cd my-react-app && npm install && npm run build          # output: dist/

# Next.js (server)
cd my-next-app && npm install && npm run build && npm start

# Next.js (static export)
cd my-next-static-app && npm install && npm run build    # output: out/

# Streak.js
cd hello-streak-app && bun install && bun run build      # output: out/
```

Measure gzip size of one file:

```bash
gzip -9c path/to/file.js | wc -c
```

### Runtime metrics (Lighthouse)

Bundle size is only one factor. To compare FCP, LCP, TBT and Speed Index, serve each build and run Lighthouse with mobile throttling:

```bash
npx serve my-react-app/dist -l 3001
npx lighthouse http://localhost:3001 --preset=perf --form-factor=mobile --output=json --output-path=./react.json
```

Repeat for each app on its own port. Run each test 3 times and use the median.

| App | FCP | LCP | TBT | Speed Index | Score |
|---|---:|---:|---:|---:|---:|
| Vanilla HTML | – | – | – | – | – |
| Streak.js | – | – | – | – | – |
| React + Vite | – | – | – | – | – |
| Next.js (static export) | – | – | – | – | – |
| Next.js (server) | – | – | – | – | – |

## Environment

Measured on 2026-10-14 with Node.js 23.5, Bun 1.2.9, Linux.
