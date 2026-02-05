# **True GTM Server-Side (server container) + Cloudflare proxy (same-origin)**

You run an actual **GTM Server container** (Cloud Run/App Engine/other), and use Cloudflare (Workers) so requests look first-party. Google’s official sGTM docs cover the server container + custom domain + sending data flow. ([Google for Developers][2])

## 1. Server
### Architecture

React (browser) → **your domain** (Cloudflare) → **Worker route** → **Tagging server** (GTM Server container on Cloud Run/App Engine/Stape/etc.) → GA4 / Ads / etc.

Google confirms server-side tagging can be set up on GCP (Cloud Run/App Engine) or another platform, and then you map a custom domain and send data to the server container. ([Google for Developers][2])

### Step 1 — Create GTM Server container

In GTM:

* Create a **Server** container
* Note the server container’s default endpoint / preview URLs
* Add Clients/Tags you need (GA4, Ads, Meta CAPI, etc.)

(Exact vendor tag setup varies; the important part is you have the server container running somewhere.)

### Step 2 — Host the tagging server

Common choices:

* **Google Cloud Run** or **App Engine** (officially documented by Google as supported setup paths). ([Google for Developers][2])
* Managed providers (e.g., Stape) also work, but you’ll still do the same Cloudflare “front it with your domain” steps.

### Step 3 — Decide your first-party endpoint format

Two common patterns:

* **Subdomain method:** `https://tag.yourdomain.com` (simpler)
* **Same-origin path method:** `https://yourdomain.com/sgtm/*` (often best for cookie lifetime and strict browsers)

If you want **same-origin through Cloudflare**, note these gotchas:

* Your site traffic must be proxied via Cloudflare.
* Cloudflare SSL/TLS should be **Full** mode.
* Avoid caching and avoid query-string normalization/sorting that can break some tags. ([stape.io][4])

### Step 4 — Cloudflare Worker to proxy to the tagging server

Create a Worker that forwards requests from:

* `https://yourdomain.com/sgtm/*`
  to
* `https://<your-tagging-server-host>/*`

**Worker example (generic proxy)**

```js
export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Only proxy sGTM path
    if (!url.pathname.startsWith("/sgtm/")) {
      return fetch(request);
    }

    // Your tagging server base URL (Cloud Run / App Engine / managed host)
    const TAGGING_ORIGIN = "https://tagging.example.com";

    // Strip the /sgtm prefix when forwarding
    const forwardPath = url.pathname.replace(/^\/sgtm/, "");
    const forwardUrl = new URL(TAGGING_ORIGIN);
    forwardUrl.pathname = forwardPath;
    forwardUrl.search = url.search;

    // Clone request with updated URL
    const newReq = new Request(forwardUrl.toString(), request);

    // Optional: preserve original host info for server container logic
    const headers = new Headers(newReq.headers);
    headers.set("X-Forwarded-Host", url.host);
    headers.set("X-Forwarded-Proto", url.protocol.replace(":", ""));
    const finalReq = new Request(newReq, { headers });

    const resp = await fetch(finalReq);

    // Avoid caching tracking responses
    const outHeaders = new Headers(resp.headers);
    outHeaders.set("Cache-Control", "no-store");

    return new Response(resp.body, {
      status: resp.status,
      statusText: resp.statusText,
      headers: outHeaders,
    });
  },
};
```

**Worker route**

* Route: `yourdomain.com/sgtm/*` → your Worker

### Step 5 — Point your Web container (React site) to the server container

In your **Web GTM container**, configure tags to send to your server container endpoint, usually called something like:

* Server URL: `https://yourdomain.com/sgtm` (path method)
  or
* Server URL: `https://tag.yourdomain.com` (subdomain method)

Google’s server-side docs explicitly call out “send data to server container” and “map a custom domain” as key steps. ([Google for Developers][2])

### Step 6 — Validate & debug

* Use GTM **Preview** for both Web and Server containers.
* In browser DevTools:

  * confirm requests go to `yourdomain.com/sgtm/...`
* If using same-origin, re-check Cloudflare requirements:

  * Full SSL/TLS
  * no caching / no query rewriting behaviors that interfere with tracking requests ([stape.io][4])



## 2. React
What Cloudflare sGTM changes (and what it doesn’t)

### ✅ What changes

* Network path:
  Browser → **your domain (Cloudflare)** → server / Google
* Cookies & endpoints become **first-party**
* You get better control, resilience, and privacy options

### ❌ What does NOT change

* **Your event source**
* **Your GTM Web container logic**
* **Your React → GTM integration**

GTM still needs **events** to react to — and that comes from the **dataLayer**.

---

## Event flow with React + Cloudflare sGTM

```
React UI action
   ↓
window.dataLayer.push({...})
   ↓
GTM Web container (triggers fire)
   ↓
Send event to sGTM endpoint (via Cloudflare)
   ↓
Server container → GA4 / Ads / Meta / etc.
```

If you don’t push events → **nothing fires**.

---

## When do you push to `dataLayer`?

### You MUST push when:

* Page is a SPA (React/Vite/Next)
* Route changes
* Button clicks
* Form submits
* Ecommerce events
* Custom business events

### You do NOT need to push manually when:

* Using **GTM auto events**:

  * Click triggers
  * Scroll depth
  * History change trigger (sometimes)
* But even then, **explicit pushes are more reliable** in SPAs

---

## Recommended pattern for React apps

### 1️⃣ Centralize event pushing

Create a small helper instead of calling `window.dataLayer.push` everywhere.

```ts
// analytics.ts
export const pushEvent = (event: string, data: Record<string, any> = {}) => {
  window.dataLayer = window.dataLayer || [];
  window.dataLayer.push({
    event,
    ...data,
  });
};
```

### 2️⃣ Use it in components

```tsx
import { pushEvent } from "./analytics";

<button
  onClick={() =>
    pushEvent("cta_click", {
      cta_name: "signup",
      page: "home",
    })
  }
>
  Sign up
</button>
```

---

## SPA page views (VERY important)

React does **not** trigger traditional page loads.

### You must push page views manually

```ts
pushEvent("page_view", {
  page_path: window.location.pathname,
  page_title: document.title,
});
```

Trigger this:

* on route change (React Router / Next router events)

---

## Does sGTM allow skipping `dataLayer`?

Only in **very specific cases**:

### You can skip dataLayer IF:

* You use **gtag.js directly**
* OR you send events **directly from your backend** to sGTM

Example backend hit:

```
POST https://yourdomain.com/sgtm/g/collect
```

But once you’re using **GTM Web container**:

> **dataLayer is the contract** between your app and GTM

---

## Common misconceptions (worth calling out)

❌ “Since it’s server-side, I don’t need dataLayer”
→ **False**

❌ “Cloudflare sGTM auto-captures events”
→ **False**

❌ “I can rely on DOM listeners only”
→ Risky in SPAs, breaks easily

✅ **Explicit dataLayer events = deterministic + debuggable**

---

## Best practices checklist
* ✅ Always push **business events** explicitly
* ✅ Use **custom events**, not DOM-only triggers
* ✅ Keep event names stable (`snake_case`)
* ✅ Send clean, flat payloads
* ✅ Let **sGTM enrich/forward**, not your frontend

---

