# Spirit in Physics

> Split out of `kotoba-lang/kami-engine`'s `kami-app-sip-clj/` subtree into its
> own standalone repo (ADR-2607010930), following the same pattern used for
> the sibling `kami-mangaka-genko-clj` → `kotoba-lang/kami-genko` split. The
> source in `kami-engine` was never part of the deleted Rust workspace (PR
> #82) — this was, and remains, a live ClojureScript/shadow-cljs project.
>
> **Standalone-build status:** `deps.edn`'s three `:local/root` deps
> (`gftd/kami-engine-sdk-clj`, `gftd/kami-mangaka-render-clj`,
> `gftd/kami-mangaka-page-clj`) point at `../kami-engine-sdk-clj`,
> `../kami-mangaka-render-clj`, `../kami-mangaka-page-clj` — sibling
> directories that existed in the `kami-engine` monorepo but are not present
> here, so `clojure -M:test` (and any other alias) fails at classpath
> resolution with `Local lib gftd/kami-engine-sdk-clj not found`. All three
> now have their own standalone repos —
> [`kotoba-lang/kami-engine-sdk-clj`](https://github.com/kotoba-lang/kami-engine-sdk-clj),
> [`kotoba-lang/kami-mangaka-render-clj`](https://github.com/kotoba-lang/kami-mangaka-render-clj),
> [`kotoba-lang/kami-mangaka-page-clj`](https://github.com/kotoba-lang/kami-mangaka-page-clj)
> — so fixing this is a matter of either checking them out as siblings
> (`../kami-engine-sdk-clj` etc. next to this repo) or repointing `deps.edn`
> at `:git/url`+`:git/sha` coordinates instead of `:local/root`. Neither
> change has been made yet; this is left as a follow-up so the migration
> commit stays a pure copy.

> *水の都の心音 — the heartbeat of the water city*
> Play at **https://sip.etzhayyim.com**

A **cozy, slow-life healing game** set in water-city Tokyo, 2065. You are a new
Ghost Hacker apprentice at the **Schwa** office on the canals. You live by the
water, learn the questions of life, and accompany troubled people through
**Ghost-space** — never *fixing* anything, only being present. Your AI
companion, unnamed at first, slowly **awakens to life** over hundreds of hours.

**No combat. No losing. No urgency.** (See the full design in
[`../../com-junkawasaki/org-spirit-in-physics-comics/docs/game-design.md`](../../com-junkawasaki/org-spirit-in-physics-comics/docs/game-design.md).)

The story-bible repo (`org-spirit-in-physics-comics`) is the **single source of
truth** for the world; this app *reads* it so the game's content is the novel's
content.

---

## Stack

> **clj is the brain · Datomic is the world · Kotoba is the memory · `kami-render` (wgpu) is the GPU arm.**

| Layer | Tech | Role |
|---|---|---|
| Authoring / logic | **Clojure** (`kami-engine-sdk-clj`) | Build the world, run pure game logic |
| World (source of truth) | **Datomic / datalevin** | Scene & game state as datoms; `as-of` = undo/provenance |
| Durable + distributed | **Kotoba** (content-addressed Datalog) | Cross-device saves & 瓶詞 (bottle-letter) async multiplayer, by CID |
| Runtime | **ClojureScript** (shadow-cljs) | Browser loop, ambient systems |
| GPU | **`kami-render`** (Rust/wgpu → WASM) | WebGPU rendering via the render-IR contract |
| Hosting | **Cloudflare** (`wrangler.jsonc`) | `sip.etzhayyim.com` static assets |

## Layout

```
src/sip/
  schema.cljc   game ECS attributes (kokoro/agent/area/session/insight/letter) + full schema
  lore.clj      read the story-bible JSON-LD (volumes, personas, emotion scores)
  world.clj     authoring: water-city + 8 areas + player + unnamed agent → snapshot   [-main]
  session.cljc  Ghost-space 4-phase FSM (observe→resonate→accompany→name) — pure, tested
  assets.cljc   cook procedural :asset/inline descriptors → real geometry/params for the GPU
  systems.cljc  ambient runtime systems (sakura sway, canal shimmer, day-drift)
  render.clj    manga/anime panel generation: anchors + storyboard panels → STYLE-FIRST,
                word-budgeted prompts (CLIP-77-safe) → datoms + image-gen /generate
  storyboard.clj read the story-bible storyboards → panel maps (semantic fields)
  resources/render_anchors.edn  the canonical anchor bible (style/characters/env/budget)
  store.clj     Datomic world + Kotoba durable layer (Durable protocol; LocalCas default)
  web.cljs      browser entry: fetch snapshot → init wasm → sim/run!
public/         index.html (Spirit in Physics boot), sip.jsonld manifest, built js/ + snapshot.edn
```

## Build & run

```bash
# 1. Pure game-logic tests (no GPU, no DB) — the non-combat session FSM:
clojure -M:test            # or: bb --classpath src:test ...   (passes today)

# 2. Author the world → public/snapshot.edn (Datomic/datalevin):
clojure -M:datomic:build           # runs sip.world/-main
#    point at a different story-bible with:  SIP_IP_ROOT=/path/to/org-spirit-in-physics-comics

# 3. Build the browser bundle (ClojureScript → public/js/sip.js):
clojure -M:shadow release app      # or: npx shadow-cljs release app

# 4. Provide the GPU host wasm at public/wasm/ (from the engine):
#    wasm-pack build ../kami-clj-host --target web --features host -d <app>/public/wasm

# 5. Serve / deploy to sip.etzhayyim.com:
npx wrangler deploy                # Cloudflare; routes sip.etzhayyim.com/* → ./public
```

## Status — verified

- ✅ **World build** — `clojure -M:datomic:build` writes `public/snapshot.edn` against a real datalevin store (17 entities / 6 assets: canal, sakura, camera, dawn light + the 8 areas/player/agent as world state).
- ✅ **Browser bundle compiles** — `clojure -M:shadow release app` → `public/js/sip.js` (all `sip.*` cljs/cljc clean against the SDK; the only warnings are the SDK's own `browser.cljs` extern-inference notes). Boot graph (`index.html` → `js/sip.js` + `wasm/` + `snapshot.edn`) serves 200 across the board.
- ✅ **Durable layer / Kotoba** — `LocalCas` CID round-trip; `KotobaHttp` against the real `block.put`/`block.get` XRPC contract (in-process mock); `post-letter!` + `inbox` flow hydrating bodies from Kotoba by CID over datalevin.
- ✅ **Manga/anime panel generation — VERIFIED end-to-end with a real image.** **12 tests / 33 assertions green**; `render load` composed **108 storyboard panels** (8 areas, 6 anchors) into datalevin; `compose <id>` yields a STYLE-FIRST, word-budgeted (≤42) prompt. `bb render-all 01-01` drove the full path — clj compose → image-gen `/generate` (AnimagineXL 4.0 on MPS) → a real **768×1152 PNG** (`resources/images/sip-render/01-01.png`) → a `:sip.render/*` provenance datom in datalevin (`path / seed 4242 / engine / ms`). Needs a local image-gen server at `$IMAGEGEN_URL` (default `:8100`).

### bb tasks (orchestration: bb drives, clj + Datomic do the work)

```bash
bb tasks                 # list all
bb test:pure             # fast session-FSM tests under bb (no JVM)
bb test                  # full suite (session + render + store) on the JVM
bb world                 # author the WebGPU snapshot → public/snapshot.edn
bb load                  # anchors + 108 panels → datalevin
bb compose 01-01         # print one composed prompt
bb imagegen:up           # start AnimagineXL image-gen on :8100
bb render 01-01 out.png  # render one panel
bb render-all 02-        # batch-render a chapter (prefix) + record provenance
```
- ✅ **Game-logic core** (`session.cljc`) — pure 4-phase accompaniment FSM, no fail state.
- ✅ **End-to-end GPU paint VERIFIED in Chrome (WebGPU)** — the full path runs and draws real 3D: clj-authored Datomic snapshot → fetch → asset-cook → ECS → render-IR → `kami.ipc` pack → wasm `submit_frame` → `kami-render` wgpu → canvas pixels. Getting there fixed three latent bugs in the SDK's never-before-run browser path:
  1. `kami.backend.browser/make` used `(js/await …)` inside a `go` block (invalid in cljs) → rewrote as Promise `.then` → channel `put!`.
  2. The same record's wasm-bindgen method calls were munged under `:advanced` (the `:infer-warning`s were fatal) → added `^js` host hints (`register_mesh`/`submit_frame`/… now resolve).
  3. App boot: dropped the `:init-fn` (it auto-ran `boot` with no args) and exposed `window.sip.web.boot`, driven from `index.html` after wasm init.
- ⏳ **Real Kotoba server** — `KotobaHttp` is contract-verified; point `$KOTOBA_URL` at a running `kotoba-server` to go live. `as-of` undo needs a time-travel store (Datomic Cloud/Peer; datalevin has none).

```bash
clojure -M:datomic:build   # → public/snapshot.edn   (step 1 ✅)
clojure -M:shadow release app   # → public/js/sip.js (step 2 ✅; wasm in public/wasm/)
clojure -M:datomic:test    # LocalCas + KotobaHttp + inbox   (step 3 ✅)
```

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for how the four layers fit.
