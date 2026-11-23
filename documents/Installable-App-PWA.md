# Installable app (PWA): how the website became a phone, tablet and desktop app

Done 22 Nov 2026 (UI release `aa7b9f4`, live on Test and Prod). Summary in
[ARCHITECTURE.md §7.10](../ARCHITECTURE.md#710-brand-and-installable-app-22-nov-2026); this is the step-by-step record.

**Audience:** developers maintaining `ShriyaMiningIndustries_UI`, and whoever sends the app to new users (section 5).

The code snippets below are the files as deployed, not a template. Where they differ from common PWA guides, the reason
is given.

---

## 1. Objective and decision

- **Goal:** the website `https://app.shriyamining.com` works as an app on Android, iPhone, iPad and computers. It keeps
  the same screens, brand and live API, and has its own icon and its own window without browser bars.
- **What we avoided:**
  - Apple App Store fees ($99 a year);
  - Google Play's $25 registration;
  - a second, native codebase to maintain.
- **Solution:** a Progressive Web App (PWA). The existing Vue app on Azure Static Web Apps gained a web app manifest,
  icons and a service worker. Nothing else changed: same repo, same pipeline, same site.
- **Cost: $0.**
  - There are no new Azure resources. The SWA Free plan already serves the static files.
  - There are no store accounts.
  - The service worker makes no extra API calls (see [ARCHITECTURE.md §10](../ARCHITECTURE.md#10-cost-model)).
- **Updates need no store review.** The installed app loads the live site over HTTPS. A push to `test` or `main`
  deploys as usual (ARCHITECTURE.md §6), and installed users get it the next time they open the app. Nobody needs
  to reinstall or rebuild anything.

---

## 2. Implementation (`ShriyaMiningIndustries_UI`)

### A. Brand and app icons (`public/`)

| File | Size | Used for |
|---|---|---|
| `public/icon-192.png` | 192 × 192 | manifest icon, Android home screen, `apple-touch-icon` (iPhone/iPad) |
| `public/icon-512.png` | 512 × 512 | manifest icon, Android splash screen, desktop install |
| `public/favicon.svg` | vector | browser tab; also the logo in the top bar and on the sign-in screen |

- **The design:** a serif **S** in sandstone over a gold rule, on the top bar's dark earth colour. The PNG icons have
  **MINING** in small capitals under the S. The tab icon shows only the S, because MINING isn't readable at 16 px.
  The owner chose a logo built from the name, not a picture.
- **Source:** the icons are generated, not drawn by hand. From the UI repo root:

  ```bash
  python scripts/brand/make_brand.py      # SVG sources (colours from src/styles/theme.css)
  node scripts/brand/render_pngs.mjs      # exact PNG sizes with headless Edge/Chrome
  ```

- **Extra renders** (maskable, 16/32 px, logo lock-ups) are kept locally in `public/old/`. That folder is git-ignored
  and is not deployed.
- **Changing the icons later:**
  1. Overwrite `public/icon-192.png` and `public/icon-512.png` (keep the names and exact sizes).
  2. Push to `test`, check, then push to `main`.
  3. If phones or browsers keep showing the old icon, add a version to the file references, e.g. `/icon-192.png?v=2`.
     Do this in `public/manifest.json` and in the `apple-touch-icon` tag in `index.html`.
  4. On iPhone/iPad the home-screen icon is copied when the app is added. Users must remove the app and add it again
     to see a new icon.

### B. Web app manifest (`public/manifest.json`)

The manifest gives the app's name, its own window and its icons.

```json
{
  "name": "Shriya Mining Industries · Orders & Invoicing",
  "short_name": "Shriya Mining",
  "description": "Purchase orders, invoices and tracking for Shriya Mining Industries.",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "background_color": "#f7f3ee",
  "theme_color": "#3a2c20",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" }
  ]
}
```

- **Colours** come from `theme.css`:
  - `theme_color` `#3a2c20` is the dark earth of the top bar, so the phone's status bar matches the top bar.
  - `background_color` `#f7f3ee` is the page's cream, shown on the Android splash screen while the app loads.
- **Only PNG icons are listed.** `favicon.svg` was in the first version of `icons` and was removed on 23 Nov 2026.
  Chrome DevTools shows warnings for an SVG manifest icon, and the PNGs already cover every size installers need.
  `favicon.svg` is still the tab icon, through `index.html`.

### C. Service worker (`public/sw.js`)

Browsers want a service worker before they offer to install. Ours **caches nothing on purpose**:

```js
self.addEventListener('install', () => {
  self.skipWaiting()                       // take over at once, don't wait for open tabs to close
})

self.addEventListener('activate', event => {
  event.waitUntil((async () => {
    const names = await caches.keys()      // remove any cache a future version might leave behind
    await Promise.all(names.map(name => caches.delete(name)))
    await self.clients.claim()
  })())
})

// A fetch handler (browsers look for one before offering to install). With no respondWith, every request goes
// to the network exactly as it would without a service worker.
self.addEventListener('fetch', () => {})
```

**Why it caches nothing:**
1. A new release shows at once; no user is stuck on an old version.
2. Nothing a signed-in user saw (invoices, buyers, prices) stays on the device after sign-out.
3. It never stands between the app and the API's sign-in cookies (ARCHITECTURE.md §8).

**Why there's no `event.respondWith(fetch(event.request))`:** many guides include it. It only re-sends every request
through the worker, which changes nothing, and it turns a dropped connection into a service-worker error. Leaving the
handler empty gives the same installability with the browser's normal behaviour.

**Offline:** the app needs the live API, so it doesn't work offline. Adding an offline page later would be a
deliberate change. It would need a cache of the app shell only (never API responses) and a cache name with a version.

### D. `index.html` and the Vue entry (`src/main.ts`)

The `<head>` of `index.html`:

```html
<link rel="icon" type="image/svg+xml" href="/favicon.svg" />
<link rel="icon" type="image/png" sizes="192x192" href="/icon-192.png" />
<link rel="apple-touch-icon" href="/icon-192.png" />
<link rel="manifest" href="/manifest.json" />
<meta name="theme-color" content="#3a2c20" />
<meta name="apple-mobile-web-app-title" content="Shriya Mining" />
```

`src/main.ts` registers the worker after the page loads:

```ts
if (import.meta.env.PROD && 'serviceWorker' in navigator) {
  window.addEventListener('load', () => { navigator.serviceWorker.register('/sw.js').catch(() => {}) })
}
```

- **Only in built sites** (`import.meta.env.PROD`). `npm run dev` never registers a worker, so local development
  can't be confused by an old one.
- **A failed registration is ignored.** The website works without the worker; only the install offer depends on it.

### E. Azure Static Web Apps settings (`public/staticwebapp.config.json`)

The PWA-related parts:

```json
{
  "routes": [
    { "route": "/sw.js", "headers": { "Cache-Control": "no-cache" } },
    { "route": "/manifest.json", "headers": { "Cache-Control": "no-cache" } }
  ],
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/api/*", "/.auth/*", "/assets/*", "/*.{json,js,css,svg,png,jpg,ico}", "/brand/*"]
  },
  "mimeTypes": { ".json": "application/json" }
}
```

- **The fallback exclude list:** Vue Router uses history mode, so any unknown address is answered with `index.html`.
  This list keeps `/manifest.json`, `/sw.js` and the icons from being answered with the app's start page.
  `*.json`, `*.js`, `*.svg` and `*.png` cover all of them.
- **`no-cache` on `sw.js` and `manifest.json`:** browsers check for a new version on every visit, so a change to
  either takes effect on the next visit.
- **The Content-Security-Policy** already allows these: `worker-src 'self'` and `img-src 'self'`.

### F. Release

1. Waited for OneDrive to finish syncing the project folder, then checked that the files were complete.
2. Before pushing, ran `npm run type-check`, `npm run test`, `npm run build` and `npm run check:devices`.
3. Committed and pushed to `feature/po-invoice-ledger`, then to `test` and `main` together (owner's go-ahead).
   Azure Static Web Apps built `dist/` and deployed both sites.
4. Ran `scripts/smoke-test.sh` against both sites (section 3).

---

## 3. Testing and verification

**Automated, on every deploy** (`scripts/smoke-test.sh`, check 7). The check confirms each file is served as itself,
not as the app's start page:

| Address | Expected |
|---|---|
| `/manifest.json` | 200, `application/json` |
| `/sw.js` | 200, JavaScript, `Cache-Control: no-cache` |
| `/icon-192.png`, `/icon-512.png` | 200, `image/png` |
| `/favicon.svg` | 200, `image/svg+xml` |

This passed on Prod (`https://app.shriyamining.com`) and on Test on 22 Nov 2026.

**Local production build:** the service worker registers with scope `/` and becomes active, and the manifest lists
both icons. The top bar's logo loads.

**Chrome DevTools** (F12 → *Application* on `https://app.shriyamining.com`):
- **Manifest:**
  - name *Shriya Mining Industries · Orders & Invoicing*, short name *Shriya Mining*, start URL `/`;
  - display `standalone`;
  - the 192 and 512 icon previews show with no warnings. The SVG icon warning went away when `favicon.svg` left the
    manifest.
- **Service workers:** `/sw.js` is *activated and is running*.

**On devices** (checked by the owner, 22 Nov 2026):

| Device | Steps | Expected |
|---|---|---|
| Computer (Chrome / Edge) | Install icon in the address bar (screen with a down arrow) → *Install* | Opens in its own window with the S icon; no address bar |
| Android (Chrome) | ⋮ menu → *Install app* (or *Add to Home screen*) | Icon on the home screen; opens full screen without browser bars |
| iPhone / iPad (Safari) | *Share* → *Add to Home Screen* → *Add* | Square S icon on the home screen; opens full screen |

---

## 4. Troubleshooting

| What you see | Cause | Fix |
|---|---|---|
| No *Install* offer in Chrome or Edge | Manifest or worker not loading | DevTools → *Application* → *Manifest*: errors are listed there. Check `/manifest.json` and `/sw.js` return JSON / JavaScript, not HTML (the fallback exclude list) |
| Old icon after an icon change | Browser or phone cache | Add `?v=2` to the icon references (section 2A). On iPhone, remove the app and add it again |
| Installed app shows an old release | Should not happen: the worker caches nothing | DevTools → *Application* → *Service workers* → *Unregister*, then reload. Report it: something started caching |
| iPhone: no *Add to Home Screen* | Opened in another browser or inside WhatsApp | Open the link in **Safari** (in WhatsApp, use ⋯ → *Open in Safari*) |

---

## 5. Sending the app to users

**The usual way, for everyone (Android, iPhone, iPad, computers):** send the link `https://app.shriyamining.com`
by WhatsApp or email, with these steps:

> **Android (Chrome):** open the link → tap ⋮ → *Install app* (or *Add to Home screen*).
> **iPhone / iPad:** open the link in **Safari** → tap *Share* → *Add to Home Screen* → *Add*.
> **Computer (Chrome / Edge):** open the link → click the install icon at the right of the address bar → *Install*.

Each person still needs their own sign-in. An Admin invites them (ARCHITECTURE.md §8); installing the app doesn't
give anyone access.

### Install guides to send (copy and paste into WhatsApp or email)

**Shriya Mining Industries app – Android install guide**

1. Open **Chrome** on your Android phone and go to https://app.shriyamining.com
2. Tap the three dots (⋮) in the top-right corner of Chrome.
3. Tap **Install app** (or **Add to Home screen**).
4. Tap **Install** (or **Add**) on the confirmation pop-up.
5. Go to your Home Screen or App Drawer and open the **Shriya Mining** app icon.
6. You'll now see the app in full-screen mode without the browser address bar.

**Shriya Mining Industries app – iPhone / iPad install guide**

1. Open **Safari** on your iPhone or iPad and go to https://app.shriyamining.com
   (If you tapped the link in WhatsApp, tap ⋯ or the compass icon and choose **Open in Safari**.)
2. Tap the **Share** button (the square with an arrow pointing up): at the bottom of the screen on iPhone, at the top
   on iPad.
3. Scroll down and tap **Add to Home Screen**.
4. Tap **Add** in the top-right corner.
5. Go to your Home Screen and open the **Shriya Mining** app icon.
6. You'll now see the app in full-screen mode without the Safari address bar.

**Shriya Mining Industries app – computer install guide (Chrome or Edge)**

1. Open **Chrome** or **Edge** and go to https://app.shriyamining.com
2. Click the install icon at the right end of the address bar (a screen with a down arrow), or open the browser menu
   and choose **Install Shriya Mining** (Edge: *Apps → Install this site as an app*).
3. Click **Install** on the confirmation pop-up.
4. The app opens in its own window. Find it later in the Start menu (Windows) or Applications (Mac) as **Shriya Mining**.

After installing, sign in as usual. Updates arrive by themselves: there's nothing to update or reinstall.

### Optional: an Android `.apk` file to send on WhatsApp

Only if someone needs a file to tap rather than a link:

1. Open [PWABuilder](https://www.pwabuilder.com) (Microsoft, free) and enter `https://app.shriyamining.com`.
2. Choose *Package for stores → Android* and download the package.
3. Send the `.apk` (e.g. renamed `ShriyaMining.apk`) as a WhatsApp **Document**.
4. The recipient taps it and allows *Install unknown apps* for WhatsApp.

Know before using it:
- **It's still the website.** PWABuilder wraps the same site; updates still arrive by deploying, with no new `.apk`.
- **Without a Digital Asset Links file, it shows a browser bar.** That file is `/.well-known/assetlinks.json` on the
  site, holding the package's signing fingerprint. Adding it is a small UI-repo change; it's not done yet.
- **Android warns about installs from outside Google Play.** For most users the link and *Install app* is simpler and
  safer. Keep the signing key PWABuilder gives you somewhere safe, never in a repo, or later versions can't replace
  the first.

**iPhone and iPad:** Apple doesn't allow installing app files from outside the App Store. iPhone and iPad users always
use Safari → *Add to Home Screen*.

---

## 6. Changelog

| Date | Change |
|---|---|
| 22 Nov 2026 | Brand monogram and icons (`icon-192.png`, `icon-512.png`, `favicon.svg`), `manifest.json`, `sw.js` (caches nothing), `index.html` tags, registration in `src/main.ts`, `no-cache` routes, smoke-test check 7. Live on Test and Prod (UI `aa7b9f4`). |
| 23 Nov 2026 | `favicon.svg` removed from the manifest's `icons` (DevTools warning); PNG icons only. The top bar stays on screen while pages scroll, on every page and device (ARCHITECTURE.md §7.9). Live on Test and Prod (UI `d45d2b9`). | 
