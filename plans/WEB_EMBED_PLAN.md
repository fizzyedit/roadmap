# Web embed: `fizzy.js`, fizzy on a host page's canvas

Status: proposed (fizzyedit/fizzy#528). Requested by Graphl, which runs fizzy's wasm on a canvas
inside its users' pages, where an iframe is not an option.
Related: `APP_BUILD_API_PLAN.md` (`addWeb`'s `exports` and `page` are how an app ships this),
`EMBEDDED_EDITOR_PLAN.md` (the same idea natively: fizzy as a guest in someone else's host).

## The problem

The page is the bootstrap. `web/index.html` (≈ 1200 lines) is one inline module that:

- **Builds the `fizzy` and `env` import namespaces inline** (`fizzyImports`: 16 functions —
  storage, fetch, image, request, OAuth, plugin load/remember/forget, fullscreen, clock) and
  the growable function table side modules join.
- **Monkeypatches `WebAssembly.instantiateStreaming` globally** to slip those namespaces in,
  because `web/web.js`'s `dvui(canvas, url)` passes only `{ dvui }`. Every wasm instantiation
  on the host page then gets fizzy's imports and sets fizzy's `wasmInstance`.
- **Listens on the page, not the canvas:** `keydown` on `window` (capture; it asks the wasm
  `FizzyWebKeyBound` and `preventDefault`s the app's chords, which takes the host page's
  shortcuts), `fullscreenchange` on `document` (Keyboard Lock), and in worker mode `focus`,
  `resize` and two `matchMedia` listeners on `window`, and a hidden input prepended to
  `document.body`. `web.js` adds its own `window` `resize`, which its canvas `ResizeObserver`
  already covers. Nothing is ever removed: there is no teardown.
- **Reads its options from `location`:** `?storage`, `?persist`, `?worker`, `?plugin`, `?open`,
  `?show`, `?demo`, `?tape`, and resolves `web.wasm`, `web.js`, `fizzy-worker.js`, the font and
  `plugins/<id>/<id>.wasm` against the page's URL.

`web/fizzy-worker.js` holds a second copy of `fizzyImports` and the plugin loader (moved there
for worker mode). Graphl carries a third, and has to keep it in step with fizzy's export list.

## The rule

- **The page is a caller.** Everything fizzy needs to run is one ES module, `web/fizzy.js`,
  whose one entry is `startFizzy(options)`. `index.html` becomes options-from-the-URL plus that
  call, and is the reference embed, not the bootstrap.
- **Nothing global.** No patched built-ins. Listeners go on the canvas (and the hidden input
  fizzy owns, placed beside the canvas, not in `body`). The few that must be on `window` or
  `document` (an OAuth popup's `message`, Keyboard Lock in full screen) are added only while
  needed. Every listener, observer, timer and worker is registered with one `AbortController`
  and released by `destroy()`.
- **Imports are composed, not patched.** `web.js` takes extra namespaces; `fizzy.js` passes
  `fizzy` and `env`, and the app passes its own. An app's wasm (built with `addWeb`) may import
  from a namespace of its own and export names of its own.
- **Options, not the URL.** Every `?param` becomes a `startFizzy` option; only `index.html`
  reads `location`. Assets resolve against `options.base` (default: `fizzy.js`'s own
  `import.meta.url`), never the page.
- **One import table, one copy.** The page and the worker import the same `fizzy.js` pieces.

## The API, when done

```js
import { startFizzy } from "./fizzy/fizzy.js";

const app = await startFizzy({
  canvas: document.querySelector("#graph"),   // or a selector
  base: "/static/fizzy/",                     // where web.wasm, the font, plugins/ are
  wasm: undefined,                            // URL, Response, or WebAssembly.Module; default base + "web.wasm"
  imports: { graphl: { graphl_log(ptr, len) { … } } },   // more namespaces for the app's wasm
  storage: { name: "graphl", persist: true },  // or { store } with getItem/setItem/removeItem/key/length
  worker: "auto",                              // "auto" | true | false
  plugins: [],                                 // plugin URLs to load at start (`?plugin=`)
  open: [], tape: null, demo: [],              // what `?open=`, `?tape=`, `?demo=` do today
  keys: "focused",                             // "focused": app chords only while the canvas has focus
  beforeRun(instance) {},                      // after instantiate, before dvui_init
});
app.instance;        // the WebAssembly.Instance (main thread only; null in worker mode)
app.requestRender();
app.destroy();       // dvui_deinit, every listener and observer released, the worker terminated
```

## Steps

Each step is one fizzy PR, `web:` area. All are JavaScript and HTML except step 4's build half;
the Zig gates still run (`zig build check-web`), and each step says in its PR how it was seen:
fizzy's page in Chrome (worker mode) and Firefox and Safari (page mode), on a phone through the
recipe in fizzy's `web/AGENTS.md` for steps 2 and 3, and the embed page from step 5.

1. **`web.js`: extra imports and a teardown.** `dvui(canvas, wasmRef, { imports, signal })`
   merges `imports` beside `dvui` (refusing to replace `dvui`) and, for a `WebAssembly.Module`
   or a `Response`, instantiates the same way. `Dvui` registers every listener with `signal`
   (or its own controller), drops the `window` `resize` listener (the canvas `ResizeObserver`
   already sees every size change that matters), and gains `destroy()`: abort, cancel the
   pending animation frame, `dvui_deinit`, release GL objects. dvui's own `web.js` gets the same
   change upstream (the dvui upstreaming plan (kept private)), since the request was for dvui's.
   *Verify:* fizzy's page unchanged in all three browsers; a scratch page that starts and
   destroys the app three times shows no listener left (`getEventListeners` in Chrome) and no
   growth in GL objects.

2. **`web/fizzy.js`: the bootstrap as a module.** Move out of `index.html` into `fizzy.js`:
   `fizzyImports`, the function table, storage (`store`, `memoryStore`, `askToPersist`), the
   plugin loader (`loadPlugin`, remembered and cached plugins, `readDylinkNeeds`), OAuth,
   images, `playTape`/`playDemo`, the wheel/pinch handling, worker mode's page half
   (`startWorker`). `startFizzy(options)` instantiates through step 1's `imports`, so the
   global `instantiateStreaming` patch is deleted. `fizzy-worker.js` imports `fizzyImports` and
   the loader from `fizzy.js` (both written to run in either scope) instead of keeping its copy.
   Paths resolve against `options.base`; `web.js`'s font fetch takes a URL from its caller.
   `index.html` keeps the loading and error panels and maps its URL to options. Install
   `fizzy.js` in `zig-out/web` (`build/web.zig`, beside `web.js`; the cache-buster stamps it).
   *Verify:* every URL parameter still works on fizzy's page (`?plugin=` with a side module,
   `?tape=`, `?demo=`, `?open=` a zip, `?storage=` + `?persist=0`, an OAuth sign-in for Drive,
   `?worker=0/1`); the page is under 200 lines; one copy of the imports (`grep -c
   fizzy_web_storage_get web/*.js` is 1).

3. **Listeners scoped to the canvas.** In `fizzy.js`: app chords are taken on the canvas and the
   hidden input (capture), not `window`, so the host page keeps its shortcuts whenever fizzy is
   not focused; `keys: "page"` keeps today's window-wide behaviour for fizzy's own full-page app.
   Keyboard Lock is asked for only when the canvas (or its container) is the full-screen element.
   Worker mode's `focus`/`resize`/`matchMedia` listeners move under the controller; the hidden
   input is inserted beside the canvas. The OAuth `message` listener already comes and goes; it
   joins the controller so `destroy()` mid-sign-in is clean. Two `startFizzy` calls on one page,
   each with its own canvas and storage name, both work: each has its own instance, table and
   store (the wasm's globals are per instance already).
   *Verify:* an embed page with a text field beside the canvas: ⌘S/Ctrl+S in the field reaches
   the page, in the canvas reaches fizzy; two embeds side by side; `destroy()` on one leaves the
   other running.

4. **The app's own imports and exports.** With `APP_BUILD_API_PLAN.md` step 4 (`addWeb`'s
   `exports` and `page`): an app's wasm adds exports through `addWeb`, imports through
   `extern "<namespace>"` in its own Zig, and supplies them through `startFizzy({ imports })`.
   `beforeRun(instance)` runs between instantiation and `dvui_init`, for an app that writes
   into memory before the first frame. `addWeb` also installs `fizzy.js` and the files a page
   needs, so an app's page imports `./fizzy.js` like fizzy's own.
   *Verify:* example-app's blocks build (APP_BUILD_API step 4) gets a web page that calls an
   app export and an app import; `check-examples.sh` builds it.

5. **Documented, and an embed example.** `docs/WEB_EMBED.md` (public): the options, what
   `fizzy.js` touches outside the canvas and when, destroy semantics, the asset layout under
   `base`, COOP/COEP and MIME notes for serving the wasm, and the export/import contract an app's
   wasm keeps. A `web/embed.html` beside `index.html` embeds fizzy in a page with other content
   and is served by `scripts/serve-web.sh` at `/embed.html`. fizzy's `web/AGENTS.md` points at
   `fizzy.js` as the one place the page's half lives.
   *Verify:* the doc's example is the embed page's code; the page works in all three browsers.

## What stays compatible

- **fizzyed.it/app and /testapp** keep every URL parameter; the page maps them to options.
- **fizzyed.it's iframe embeds** (`?storage=`, `?persist=0`) keep working; they could move to
  `fizzy.js` later (see open questions).
- **Side-module plugins** load as before: `env` and `fizzy` namespaces and the function table are
  the same, built in one place.
- **The export list** in `build/web.zig` is unchanged until step 4 lets an app extend it.

## Open questions

- **Upstream first or fizzy first?** fizzy's `web.js` is a fork of dvui's. Land step 1 in fizzy
  and offer it to dvui, or wait on dvui? Proposed: fizzy first, offered upstream in
  the dvui upstreaming plan (kept private)'s next batch.
- **Worker mode in an embed.** Worker mode moves the canvas to an `OffscreenCanvas`; the canvas
  can't be used by the host afterwards, and `destroy()` can't give it back. Default `worker:
  false` for embeds, `"auto"` only on fizzy's own page?
- **Keys while unfocused.** Should an embed ever take chords while the canvas is not focused
  (fizzy's full-page app does)? Proposed: never, unless `keys: "page"`.
- **Storage default for an embed.** `storage.name` required, or default to the build's name with
  the risk of two embeds sharing settings? Proposed: required when `persist` is on.
- **fizzyed.it's embeds.** Move the website's iframes to `fizzy.js` (one less page per embed), or
  keep iframes there for isolation?
- **Versioning `fizzy.js`.** It is the app's contract with the wasm it ships beside. Is "same
  build, same directory" enough, or does `fizzy.js` check the wasm's export list on start and
  say which export is missing?
