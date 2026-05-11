Next.js Integration Review

1. Current State (what's actually there)

apps/next/
├── next.config.js          # output: 'export', distDir: '../../dist'
├── app/
│   ├── layout.jsx          # 65 KB (favicon hacks + 8 <Script> tags inline)
│   ├── page.jsx            # / → ClientPage
│   ├── ClientPage.jsx      # dynamic(old_code/CustomerApp/App, {ssr: false})
│   ├── not-found.jsx       # renders same ClientPage  ← only fallback for unmatched URLs
│   ├── cuisines/page.jsx   # SSG page (force-static)
│   │   ├── ClientCuisinesPage.jsx
│   │   └── CuisinesShell.jsx   # mounts its OWN StoreWrapper + NavigationContainer
│   ├── navigation-wrapper.jsx  # standalone NavigationContainer helper
│   └── styles-provider.jsx     # legacy, unused
├── stubs/                  # 7 native-module stubs
└── scripts/prerender-cuisines.js  # Puppeteer post-build HTML snapshot

old_code/AppModules/RouterModule/Utils/RouterConfig.js defines ~80 screens across customer/franchise/tablet variants, including parameterised SEO routes like
:town/:slug_name/:currentStoreID/ordernow, cuisines/:selectedCuisines?, cuisine/:selectedCuisines/:town, blogs/:blog_name, etc.

2. What works well

- transpilePackages + webpack alias mirror of the old webpack build — correct approach.
- ssr: false for the RN tree — necessary; RN-Web with redux-persist/localStorage can't SSR cleanly.
- Client-only scoping of React aliases and ModuleFederationPlugin — correctly avoids server prerender errors.
- 25 TDZ + ESM-strictness fixes in old_code are real bugs the webpack build was hiding; keep them.
- React Navigation linking config is left untouched — right call. Next owns the URL, RN-Nav owns the in-app state.

3. Real problems (ordered by impact)

┌─────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┬──────────────────────────────────────┐
│  #  │                                                                 Issue                                                                  │            Why it matters            │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│     │ No app/[[...slug]]/page.jsx catch-all. Only / and /cuisines exist; everything else relies on not-found.jsx, which works only in next   │ Hard 404s in production for every    │
│ 1   │ dev. After output: 'export', unmatched URLs (e.g. /london/pizza-place/123/ordernow) 404 on a CDN unless an SPA-fallback rewrite is     │ deep link.                           │
│     │ configured on the host. The NEXT_JS_MIGRATION.md claims a catch-all but the file isn't there.                                          │                                      │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│     │ CuisinesShell re-bootstraps a parallel app tree. It mounts its own StoreWrapper + MUIThemeProvider + NavigationContainer with a tiny   │ "Double router" class of bugs:       │
│ 2   │ linking config (cuisines only). If a user lands on /cuisines and then in-app navigates to a takeaway, they're inside the wrong         │ broken back button, lost state,      │
│     │ navigator.                                                                                                                             │ double-fetch.                        │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ 3   │ Puppeteer post-build prerender is brittle. Network-allowlist racing, networkidle0 heuristics, no determinism. Doesn't scale to the 10+ │ Snapshots silently break; no CI      │
│     │  SEO routes that actually matter (home, takeaway list, takeaway info, menu, top10, region, locations, blogs).                          │ signal until a crawler complains.    │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ 4   │ ssr: false everywhere ⇒ zero useful HTML for crawlers (except the one Puppeteered page). All generateMetadata/<head> content for       │ Static export gives no SEO benefit   │
│     │ parameterised routes is missing. JSON-LD (LocalBusiness, Restaurant, BreadcrumbList, ItemList) is not emitted.                         │ over the old webpack SPA.            │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ 5   │ layout.jsx is 65 KB — multiple inline <script> blobs (favicon DOM scrubber, fallback loader, third-party SDKs).                        │ Bad TTI, hard to audit, blocks       │
│     │                                                                                                                                        │ Lighthouse.                          │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ 6   │ Version skew: eslint-config-next@13.2.0 with next@15.5.3; app/styles-provider.jsx references a removed app/src/Provider.               │ Lint silently weak, dead file.       │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│     │ No generateStaticParams for parameterised routes. With output: 'export', you can't dynamic-route without it, so SEO routes can never   │ Can't actually ship SSG for          │
│ 7   │ be SSG-d.                                                                                                                              │ takeaway/menu pages under the        │
│     │                                                                                                                                        │ current export config.               │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│     │ No trailingSlash decision. Static-export servers vary; without explicit config the /cuisines page resolves to cuisines.html and the    │ Inconsistent URLs →                  │
│ 8   │ puppeteer script writes dist/cuisines.html (works), but /london/pizza-place/ordernow/ vs /london/pizza-place/ordernow is left to       │ duplicate-content penalties.         │
│     │ whatever host serves the static files.                                                                                                 │                                      │
├─────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ 9   │ apps/next/out/ committed alongside .next/ — looks like a stale build artefact.                                                         │ Repo bloat; sometimes shadows real   │
│     │                                                                                                                                        │ output.                              │
└─────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┴──────────────────────────────────────┘

4. Industry direction (2025–2026)

Based on current docs (Next.js 15, Solito 5, RN-Web, Expo Router):

- transpilePackages is the only supported path — next-transpile-modules is archived since Next 13.1. ✅ You're already on this.
- CSR-shell via next/dynamic({ ssr: false }) inside a 'use client' component, rendered from a server page.tsx, is still the canonical RN-on-Next pattern. Next 15 forbids ssr: false
directly in server components — your ClientPage.jsx split is correct.
- Solito 5 (Oct 2025) dropped its react-native-web dependency and now positions itself as web-first. It's the recommended layer if you want a single Link/useRouter API across RN-Native
and Next. If you don't share Link code today, adopting Solito is optional, not urgent.
- Mixing force-static pages with a [[...slug]] catch-all works as long as paths don't overlap. Parallel routes for SSG-vs-CSR in one slot are still unreliable (vercel/next.js#59820).
- For SEO pages with rich content reuse from RN, two valid options:
- (a) Next-only React components for the few public pages, using generateMetadata and either rendering plain HTML or reusing your shared design primitives (NOT the full RN screens).
Fastest, cleanest.
- (b) RN-Web's AppRegistry.getApplication() SSR API, called inside a server component via useServerInsertedHTML. Works but the output is <div>-based — semantically poor for crawlers,
heavy build cost.
- Industry trend: (a) for marketing/listing pages, (b) only if you genuinely need an authenticated RN shell with HTML.
- Expo Router now does build-time static rendering with universal linking and is a credible alternative when web is mobile-companion. For your case (web is a first-class product with
deep SEO needs and a heavy existing webpack/Next pipeline), staying on Next.js is right.
- output: 'export' constraints in Next 15: no searchParams, no ISR, every dynamic route needs generateStaticParams returning all paths at build time, no API routes (you already removed
/api/hello). If the takeaway catalogue is large or changes daily, drop output: 'export' and run Next as a real Node server / Vercel — that unlocks ISR for /:town/:slug_name/.... This is
the single biggest leverage point.

Sources: Next.js 15 release notes, Single-Page Applications, static-exports, Solito 5 announcement, RN-Web rendering.

5. Target architecture (Next.js as a thin wrapper)

apps/next/
├── next.config.js
├── app/
│   ├── layout.jsx                    # slim down: move favicon/script hacks to /public + <Script src=…>
│   │
│   ├── (marketing)/                  # Route group: Next-native SEO pages, no RN tree
│   │   ├── page.jsx                  # /  (home) — generateMetadata + HTML
│   │   ├── about/page.jsx
│   │   ├── privacy/page.jsx
│   │   ├── terms/page.jsx
│   │   ├── cookies/page.jsx
│   │   ├── contact/page.jsx
│   │   ├── locate-us/page.jsx
│   │   ├── blogs/[[...slug]]/page.jsx
│   │   ├── cuisines/page.jsx         # rewrite as Next page (no RN tree, no puppeteer)
│   │   ├── cuisine/[slug]/[town]/page.jsx
│   │   ├── [town]/page.jsx           # locations
│   │   └── [town]/[slug_name]/        # takeaway info + menu (parameterised SEO)
│   │       ├── info/page.jsx
│   │       ├── reviews/page.jsx
│   │       └── ordernow/page.jsx     # SSG/ISR, JSON-LD Restaurant + Menu
│   │
│   └── [[...slug]]/page.jsx          # CSR catch-all — mounts old_code/CustomerApp/App
│                                     # handles auth, cart, checkout, profile, orders, chat
│
├── lib/                              # shared server helpers
│   ├── seo/                          # generateMetadata, JSON-LD builders
│   └── api/                          # server-side data fetching (replaces redux saga calls)
│
└── stubs/                            # unchanged

Key rules:
1. One NavigationContainer, ever. The catch-all owns the RN tree; SEO pages do not mount React Navigation at all. Click handlers on SEO pages use next/link (or <a>) to hard-navigate into
the CSR shell, which then takes over.
2. SEO pages reuse Redux state via preload. Server fetches the same backend endpoint your saga uses, renders HTML, then __PRELOADED_STATE__ is hydrated into the CSR shell when the user
clicks through.
3. generateMetadata per public route — title, description, canonical, OG, JSON-LD.
4. Drop output: 'export' for these pages so you can use ISR (revalidate: 3600). Keep export only if a CDN-only deployment is a hard constraint — in which case generateStaticParams for
the top-N takeaways and use the CSR shell for the long tail.
5. Delete CuisinesShell + prerender-cuisines.js once (marketing)/cuisines/page.jsx lands. They're the proof-of-concept the real implementation replaces.

6. Complete task list (in dependency order)

Phase A — Stabilise current setup (no architecture change)

1. Add the missing catch-all apps/next/app/[[...slug]]/page.jsx + ClientShell.jsx that mounts old_code/CustomerApp/App exactly like the current ClientPage.jsx. Move root app/page.jsx
into it.
2. Delete app/not-found.jsx (replaced by the catch-all) and app/styles-provider.jsx (dead — imports nonexistent app/src/Provider).
3. Set trailingSlash: true (or false) explicitly in next.config.js and align with the CDN/host config. Update RouterConfig.js's getCustomPathFromState if needed.
4. Decide on output: 'export'. If you want ISR/proper SEO, remove it and add a Node deployment target; otherwise document the SPA-fallback rewrite required on the static host
(S3/CloudFront/Netlify all need try_files-style rules).
5. Bump eslint-config-next to 15.5.x; run pnpm lint and fix.
6. Delete apps/next/out/ (committed stale build artefact) and add it to .gitignore.
7. Slim app/layout.jsx: move the favicon scrub + fallback-loader scripts to apps/next/public/inline/*.js and load them via <Script strategy="beforeInteractive" src="/inline/..." />.
Target <3 KB JSX.
8. Regression test: spot-check 10 representative URLs on a built dist/ served via npx serve — at minimum /, /login, /profile, /order-tracking/<id>/<orderId>,
/london/pizza-place/123/ordernow, /checkout, /cuisines, /blogs, /about.

Phase B — Real SEO for marketing & static pages (Next-only components)

9. Build lib/seo/ helpers: buildMetadata({ title, description, canonical, og, jsonLd }), buildBreadcrumbsJsonLd, buildRestaurantJsonLd, buildItemListJsonLd. Keep them server-side, no RN
imports.
10. Port static content pages to Next server components: /about, /privacy, /terms, /cookies, /contact, /locate-us (HTML + per-page generateMetadata). Replace the catch-all entries for
these paths.
11. Replace /cuisines properly: Next server component that fetches cuisines from the same API your saga calls, renders semantic <ul> of cuisines + cities, full metadata + JSON-LD
ItemList. Delete CuisinesShell.jsx, ClientCuisinesPage.jsx, scripts/prerender-cuisines.js, and the prerender-cuisines npm script.
12. Blogs: /blogs index + /blogs/[slug] server pages with generateStaticParams (or ISR if not exporting). Article body + JSON-LD BlogPosting.

Phase C — Parameterised SEO pages (the high-value SEO surface)

13. /[town] (locations) — server page; SSG top N towns via generateStaticParams, ISR for the rest if not exporting.
14. /[town]/[slug_name]/info — restaurant info page; JSON-LD Restaurant + LocalBusiness. Click-through to the CSR menu.
15. /[town]/[slug_name]/ordernow — menu landing; SSG/ISR the menu HTML for crawlers, hydrate into the RN-Web menu UI on interaction. Decide: full Next reimplementation (cleanest) vs
RN-Web AppRegistry.getApplication() SSR (faster to ship, ugly markup). Recommend: Next reimplementation, share data-fetching only.
16. /[town]/[slug_name]/reviews, /[town]/[slug_name]/images — same pattern.
17. /cuisine/[slug]/[town] — cuisine-in-town page with ItemList of takeaways.
18. Sitemap: app/sitemap.ts enumerating all SSG paths (towns × takeaways) + static pages.
19. robots.ts with environment-aware indexing rules.

Phase D — Cross-cutting cleanup

20. next/image wrapper at packages/ui/Image.web.tsx (and .native.tsx) so the same <Image> import emits <img> on RN-Native and next/image on Next-server pages. Use only on SEO pages
first.
21. next/font for Lato → CSS variable; reference var(--font-lato) from RN-Web styles. Remove the 12 inline @font-face declarations from globals.css.
22. Document the routing contract in NEXT_JS_MIGRATION.md: which paths are Next-owned (SEO) vs catch-all (CSR shell). Reconcile with RouterConfig.js's mobile/tablet linking.
23. CI guard: build + serve + curl-check 20 known URLs (200 status, expected <title>, expected JSON-LD). Add a Lighthouse SEO budget for the homepage and one takeaway page.
24. Evaluate Solito 5 adoption once the wrapper is stable — only if you want one <Link> / useRouter API across Expo and Next. Optional.

Phase E — Optional later

25. Replace app/(marketing)/cuisines and /locations with edge runtime if traffic warrants (export const runtime = 'edge').
26. Consider Partial Prerendering when it leaves experimental for the parameterised takeaway pages.
27. Drop output: 'export' entirely once a Node host is in place; switch to ISR with revalidate per route.

---
Bottom line: the foundation is sound, but as-is the Next.js layer gives almost no SEO benefit over the old webpack SPA, and output: 'export' without a catch-all will break every deep
link in production. Phase A is mandatory before this can ship; Phase B–C is where Next.js actually starts earning its keep.
