# Measurement contract: `store_click`

This documents the provider-neutral click hook wired into every page with an
App Store CTA. It is a contract for what actually fires today — not a spec
for what could be added later.

## Where it lives

Each page (`index.html` at the site root, and each `apps/<app>/index*.html`,
including localized variants) ends with an inline script that adds one
document-level click listener:

```js
document.addEventListener("click", function (event) {
  var trigger = event.target.closest && event.target.closest("[data-app][data-placement]");
  if (!trigger) return;
  var detail = { app: trigger.getAttribute("data-app"), placement: trigger.getAttribute("data-placement") };
  document.dispatchEvent(new CustomEvent("store_click", { detail: detail }));
  if (window.dataLayer && typeof window.dataLayer.push === "function") {
    window.dataLayer.push({ event: "store_click", app: detail.app, placement: detail.placement });
  }
});
```

The listener is delegated on `document`, so it fires for any click that
bubbles up through an element matching `[data-app][data-placement]` —
in practice, the App Store CTA anchors.

## When it fires

On any click on an element carrying both `data-app` and `data-placement`
attributes (or a click on a descendant of such an element, e.g. an icon or
span inside the CTA link). It does not fire on hover, focus, or keyboard
activation that doesn't dispatch a `click` event.

## Event: `store_click`

A `CustomEvent` named `store_click`, dispatched on `document`. Any script on
the page can listen for it:

```js
document.addEventListener("store_click", function (e) {
  console.log(e.detail.app, e.detail.placement);
});
```

### `detail` fields (as implemented)

| Field | Source | Meaning |
|---|---|---|
| `app` | `data-app` attribute on the clicked trigger | Which app the CTA points to, e.g. `coffee-grower-sim`, `vineyard-sim`, `olive-farm-sim` |
| `placement` | `data-placement` attribute on the clicked trigger | Where on the page the CTA lives, e.g. `home-card` (portfolio card on the root `index.html`) or `app-hero` (hero CTA on an app's own page) |

No other fields (`source`, `campaign`, `locale`, `href`) are present in the
event detail. If future work adds them, update this table to match — don't
assume they exist because they'd be useful.

## Trigger attributes

CTA anchors carry both attributes so the delegated listener above can find
them:

- `data-app="<app-slug>"` — which app.
- `data-placement="<placement-slug>"` — which spot on the page. Current
  values in use: `home-card`, `app-hero`.

## `dataLayer` forwarding (optional, best-effort)

If `window.dataLayer` exists and has a `push` function, the same click also
pushes:

```js
{ event: "store_click", app: detail.app, placement: detail.placement }
```

This is a no-op unless something on the page has already defined
`window.dataLayer` (e.g. a tag manager snippet). No such snippet is present
in this repo today — the push is dead code until one is added.

## Apple `ct` campaign token

Separately from the JS hook, each App Store link's `href` carries a `?ct=`
query parameter, e.g. `?ct=home-coffee` or `?ct=app-vineyard-hero-de`. This
is Apple's own campaign-attribution token, read by App Store Connect/App
Analytics on the App Store side after the user lands there — it is not read
or set by the `store_click` JS and doesn't appear in the event `detail`. It
lets App Store analytics distinguish which on-site link a store visit came
from (home card vs. app-page hero, and per-locale page for localized
variants).

## No analytics provider is configured

There is no Google Analytics, GTM, Plausible, or any other analytics script
loaded anywhere in this site. `store_click` and the `dataLayer` push are
hooks only — they run in the browser and go nowhere unless something else on
the page is listening or `dataLayer` is later wired to a real tag manager.
Having these hooks in place does not mean click analytics are live; it means
the site is ready to plug a provider in without touching the CTA markup or
the click-handling script again.
