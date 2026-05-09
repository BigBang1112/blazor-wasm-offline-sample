# Blazor WebAssembly Offline Sample

[![GitHub last commit (branch)](https://img.shields.io/github/last-commit/bigbang1112/blazor-wasm-offline-sample/main?style=for-the-badge&logo=github)](https://github.com/BigBang1112/blazor-wasm-offline-sample)

A guide and working example for deploying a Blazor WebAssembly app to **GitHub Pages** with full **offline (PWA) support**. GitHub Pages imposes several restrictions that require workarounds.

## Steps

### 1. Create a default Blazor WASM project

Start from the standard Blazor WebAssembly template with PWA / offline support enabled:

```
dotnet new blazorwasm --pwa -o MyApp
```

### 2. Reduce WASM bundle size with invariant globalization

The ICU globalization data in the WASM runtime is very large. Unless you need full culture/timezone support, add these properties to your `.csproj`:

```xml
<PropertyGroup>
    <InvariantGlobalization>true</InvariantGlobalization>
    <InvariantTimezone>true</InvariantTimezone>
</PropertyGroup>
```

### 3. Handle SPA routing on GitHub Pages (404.html trick)

GitHub Pages is a static host, so it has no server-side routing. When a user navigates directly to a deep link like `https://bigbang1112.github.io/blazor-wasm-offline-sample/counter`, it returns a 404 because there is no `counter/index.html` file.

The workaround is a two-part redirect:

**Part A — `wwwroot/404.html`:** GitHub Pages serves this file for any unresolved path. It converts the URL into a query string and redirects to the root.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Single Page Apps for GitHub Pages</title>
    <script type="text/javascript">
        // https://github.com/rafrex/spa-github-pages
        // Set segmentCount to 1 for Project Pages (username.github.io/repo-name),
        // or 0 for User/Organisation Pages (username.github.io).
        var segmentCount = 1;

        var l = window.location;
        l.replace(
            l.protocol + '//' + l.hostname + (l.port ? ':' + l.port : '') +
            l.pathname.split('/').slice(0, 1 + segmentCount).join('/') + '/?p=/' +
            l.pathname.slice(1).split('/').slice(segmentCount).join('/').replace(/&/g, '~and~') +
            (l.search ? '&q=' + l.search.slice(1).replace(/&/g, '~and~') : '') +
            l.hash
        );
    </script>
</head>
<body>
</body>
</html>
```

**Part B — `wwwroot/index.html` (inside `<head>`):** Reads the query string set by `404.html` and restores the original URL in the browser's history before Blazor boots.

```html
<!-- Start Single Page Apps for GitHub Pages -->
<script type="text/javascript">
    (function (l) {
        if (l.search) {
            var q = {};
            l.search.slice(1).split('&').forEach(function (v) {
                var a = v.split('=');
                q[a[0]] = a.slice(1).join('=').replace(/~and~/g, '&');
            });
            if (q.p !== undefined) {
                window.history.replaceState(null, null,
                    l.pathname.slice(0, -1) + (q.p || '') +
                    (q.q ? ('?' + q.q) : '') +
                    l.hash
                );
            }
        }
    }(window.location))
</script>
<!-- End Single Page Apps for GitHub Pages -->
```

### 4. Serve Brotli-compressed assets

GitHub Pages cannot serve `Content-Encoding: br` headers, so the browser's built-in Brotli decompression will not kick in automatically. However, `dotnet publish` compresses all `.dll`, `.wasm`, `.pdb`, and `.dat` files into `.br` archives. You need to decompress them in JavaScript at load time.

**Step A — Add `decode.min.js` to `wwwroot/`.**

This is a minified port of the Brotli decoder. You can find it in the [google/brotli](https://github.com/google/brotli/tree/master/js) repository, or copy it from this sample's `wwwroot/decode.min.js`.

**Step B — Modify `wwwroot/index.html`.**

Add `autostart="false"` to the Blazor script tag, then start Blazor manually with a custom `loadBootResource` function that fetches `.br` files and decompresses them on the fly:

```html
<script src="_framework/blazor.webassembly#[.{fingerprint}].js" autostart="false"></script>
<script>navigator.serviceWorker.register('service-worker.js', { updateViaCache: 'none' });</script>
<script type="module">
    import { BrotliDecode } from './decode.min.js';
    Blazor.start({
        loadBootResource: function (type, name, defaultUri, integrity) {
            if (type !== 'dotnetjs' && location.hostname !== 'localhost' && type !== 'configuration') {
                return (async function () {
                    const response = await fetch(defaultUri + '.br', { cache: 'no-cache' });
                    if (!response.ok) {
                        throw new Error(response.statusText);
                    }
                    const originalResponseBuffer = await response.arrayBuffer();
                    const originalResponseArray = new Int8Array(originalResponseBuffer);
                    const decompressedResponseArray = BrotliDecode(originalResponseArray);
                    const contentType = type ===
                        'dotnetwasm' ? 'application/wasm' : 'application/octet-stream';
                    return new Response(decompressedResponseArray,
                        { headers: { 'content-type': contentType } });
                })();
            }
        }
    });
</script>
```

> **Note for .NET 10+:** `loadBootResource` is a top-level option on `Blazor.start({})`, not nested under `webAssembly: {}`. Earlier .NET versions used the nested form. Using the wrong form silently does nothing and the app loads uncompressed files.

The `location.hostname !== 'localhost'` guard ensures Brotli interception is skipped during local development, where the dev server serves files normally.

### 5. Update the service worker to cache `.br` files

This step can be skipped if you're fine with just gzip compression.

Currently, compression will work for fine for gzip assets. GitHub Pages will allocate the `.gz` files automatically, confirmed by the presence of `Content-Encoding: gzip` headers. But if we want to add Brotli support (known to be more efficient than gzip), additional work needs to be done, because GitHub Pages cannot serve `.br` files with the correct headers.

The service worker caches assets during install. Since `loadBootResource` fetches `asset.url + '.br'` at runtime, the service worker must pre-cache those same `.br` URLs — otherwise the browser fetches `.br` from the network every time, defeating offline support.

Also, the integrity hash in the manifest is for the *uncompressed* file. Including it on the `.br` request causes the browser to reject the response. It has to be stripped for Brotli assets.

Replace the default `assetsRequests` mapping in `service-worker.published.js`:

```js
const assetsRequests = self.assetsManifest.assets
    .filter(asset => offlineAssetsInclude.some(pattern => pattern.test(asset.url)))
    .filter(asset => !offlineAssetsExclude.some(pattern => pattern.test(asset.url)))
    .map(asset => {
        const isBrotliAsset = /\.(dll|wasm|pdb|dat)$/.test(asset.url);

        if (isBrotliAsset) {
            // Append .br to match what loadBootResource fetches.
            // Omit integrity — the hash is for the uncompressed file.
            return new Request(asset.url + '.br', { cache: 'no-cache' });
        } else {
            return new Request(asset.url, { integrity: asset.hash, cache: 'no-cache' });
        }
    });
```

### 6. Activate the new service worker immediately

In the `onInstall` handler, call `self.skipWaiting()` to activate the new service worker as soon as possible. This ensures that the updated caching logic takes effect right away, without waiting for users to close all tabs.

```js
self.skipWaiting();
```

### 7. Fix `onFetch` to serve `index.html` only for SPA navigation requests

Currently, all navigation requests are being served `index.html`, which breaks requests for static assets that include a file extension (e.g. `.css`, `.js`, `.png`). Instead, we should only serve `index.html` for navigation requests that do not have a file extension, while allowing requests with file extensions to pass through to the cache or network as normal.

Update `onFetch` to serve `index.html` for SPA navigation requests, and to let requests with a file extension pass through to the cache or network without forcing `index.html`:

```js
async function onFetch(event) {
    if (event.request.method !== 'GET') {
        return;
    }

    const url = new URL(event.request.url);

    // Add any paths that should always be fetched from the network here
    const directHitPaths = [];
    const isDirectHit = directHitPaths.some(path => url.pathname.startsWith(base + path));

    if (isDirectHit) {
        return fetch(event.request);
    }

    const isNavigationRequest = event.request.mode === 'navigate';
    const hasFileExtension = url.pathname.match(/\.[a-zA-Z0-9]+$/);

    const shouldServeIndexHtml = isNavigationRequest
        && !hasFileExtension
        && !manifestUrlList.some(manifestUrl => manifestUrl === event.request.url);

    const cache = await caches.open(cacheName);

    if (shouldServeIndexHtml) {
        const indexRequest = new URL('index.html', baseUrl).href;
        const cachedIndex = await cache.match(indexRequest);
        if (cachedIndex) return cachedIndex;
    }

    const cachedResponse = await cache.match(event.request);
    return cachedResponse || fetch(event.request);
}
```

### 8. Add a GitHub Actions workflow for GitHub Pages

GitHub Pages by default serves from the `username.github.io/repo-name` path, so the app must reflect the base path in the `wwwroot/index.html` `<base href="/repo-name/">` tag AND in the `wwwroot/service-worker.js` `baseUrl` variable. This can be changed either manually in these files or dynamically in the workflow in case you deploy to multiple paths. If changed dynamically, it should be done before the publish step to avoid integrity issues, but be aware this approach for example breaks the `BlazorWasmPreRendering.Build` package. Of course, you can avoid these two patches if you are deploying to the same path.

Jekyll should be also explicitly disabled with the `.nojekyll` file, otherwise GitHub Pages will ignore all files and folders that start with an underscore (e.g. `_framework`). This can be done in the workflow after the publish step.

```yml
- name: Patch base href in index.html
  run: sed -i 's/<base href="\/" \/>/<base href="\/${{ github.event.repository.name }}\/" \/>/g' BlazorWasmOfflineSample/wwwroot/index.html

- name: Patch base in service-worker.published.js
  run: sed -i 's/const base = "\//const base = "\/${{ github.event.repository.name }}\//g' BlazorWasmOfflineSample/wwwroot/service-worker.published.js

- name: Publish
  run: dotnet publish BlazorWasmOfflineSample -c Release -o publish

- name: Add .nojekyll file
  run: touch publish/wwwroot/.nojekyll

- name: Setup Pages
  uses: actions/configure-pages@v6

- name: Upload artifact
  uses: actions/upload-pages-artifact@v5
  with:
    path: publish/wwwroot/.

- name: Deploy to GitHub Pages
  id: deployment
  uses: actions/deploy-pages@v5
```

You can use [this optimized workflow](https://github.com/BigBang1112/workflows/blob/main/.github/workflows/deploy-pages.yml) to deploy faster to GitHub Pages with caching.

```yml
name: Deploy to Pages

permissions:
  contents: read
  pages: write
  id-token: write

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy to Pages
    uses: bigbang1112/workflows/.github/workflows/deploy-pages.yml@main
    with:
      dotnet-version: 10.x.x
      project: MyApp
```

## Demo

The sample is deployed [here](https://bigbang1112.github.io/blazor-wasm-offline-sample/).
