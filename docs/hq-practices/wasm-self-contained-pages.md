---
summary: "One-file interactive HTML with embedded Rust wasm: build profile and size, one engine for CLI and page with a staleness gate, inline CSP, SVG canvas accessibility, headless Chrome checks"
group: docs
tags: [wasm, html, diagrams, rust]
applies-to: [all]
reviewed: 2026-11-07
status: stable
---

# Self-contained pages with embedded WebAssembly

A page that is **one HTML file** (inline CSS, JS, SVG, data and a wasm module) works from `file://`, from a chat
attachment and from a USB stick, needs no server and makes no network request. The price is size, a few browser rules
that differ from served pages, and a build product (the `.wasm`) that can drift from its source. This page collects the
rules; [diagrams-as-code.md](diagrams-as-code.md#a-data-model-drawn-by-a-small-renderer) has the diagram-specific
lessons, [rust-build-times.md](https://github.com/evocaslab/hq/blob/main/practices/rust-build-times.md) the dev-profile side.

## Embedding and loading

| Rule | Why |
| --- | --- |
| Embed the module as **base64 in a `const`**, decode to bytes, compile with `WebAssembly.compile` / `instantiate` (async) | `fetch()` and `instantiateStreaming` need a URL; a `file://` page is an opaque origin and sibling files trigger CORS errors (MDN). wasm-bindgen's `--target web` output "cannot directly open `index.html`" for the same reason. Base64 costs about a third more bytes (MDN) but survives any text pipeline |
| Decode with `Uint8Array.fromBase64` when present, `atob` + `charCodeAt` as the fallback | `atob` returns a string of code points, built for text; `fromBase64` gives bytes directly and is Baseline 2025 (newly available), so keep the fallback for a year or two |
| Never the synchronous `new WebAssembly.Module(bytes)` for a module above a few KB | MDN: compilation of large modules is expensive and "some browsers may throw a `RangeError`" on the UI thread; the exact limit is not documented (not found) |
| Stub every declared import with a throwing function and check `WebAssembly.Module.imports(mod)` is empty in a test | a module built without `wasm-bindgen` and without std I/O has no imports; one appearing means a dependency pulled in the host (time, random, print) and the page would fail only at run time |
| **Keep the page working without the module**: the static SVG stands, interactive extras switch off, a status line says so | the module can fail to compile (an old browser, a CSP, a stale asset) and a diagram that still reads is the point of a one-file page |
| Data: URIs for images and attachments, a `blob:` URL only for downloads created at run time | top-level navigation to `data:` is blocked in browsers; a download link needs `download=` and a blob or data URL (MDN) |

## Build profile and size

| Rule | Why |
| --- | --- |
| A named profile: `inherits = "release"`, `opt-level = "z"` (measure `"s"` too), `lto = true`, `codegen-units = 1`, `strip = true`, `panic = "abort"` | Cargo: `s`/`z` optimise for size but "not necessarily smaller", so measure; fat LTO inlines and prunes across crates; one codegen unit lets LTO see everything; `wasm32-unknown-unknown` is already `-Cpanic=abort` by default (rustc), so the explicit line documents it and keeps a future default change out |
| Run **`wasm-opt -Oz`** (Binaryen) on the linked module in the build recipe, with `--strip-producers` and `--strip-debug` | the rustwasm book measures another 15-20 % after rustc; `strip = true` still leaves a `producers` section. Install the pinned release binary (Binaryen ships Linux, macOS, Windows builds) or the `wasm-opt` crate; `wasm-pack` would do the same step |
| **Measure the three sizes** and put a ceiling in a test: raw `.wasm`, gzip, base64-in-page | the raw size is what the browser compiles, gzip is what mail and git transfer, base64 is what the user's file weighs; a test with the old number written in it shows every jump in the diff |
| Find the bloat with `twiggy top` before cutting | formatting (`format!`, `to_string`), panic messages, generic monomorphisation and `serde_json` are the usual top entries (rustwasm book); guessing wastes a day |
| Raw `extern "C"` ABI (alloc, call, dealloc, pointers as `u32`) when the interface is "one string in, one string out"; `wasm-bindgen` when the module must hold JS objects, DOM handles or many typed functions | wasm-bindgen adds a generated JS glue file and a `_bg.wasm` and exists to pass strings, objects and classes instead of "purely integers and floats"; one JSON request does not need it, and the glue is one more file to inline and keep in step. Pack `(ptr << 32) \| len` into a `u64` only for 32-bit memories |
| `no_std` is not a size tool here: keep `std` (`alloc` + `serde_json` is the real weight), unless the module must drop the allocator | on `wasm32-unknown-unknown` std's I/O parts return errors or panic and `dlmalloc` (about 10 KB) is the allocator; `no_std` removes `HashMap`, `String` formatting conveniences and stack-overflow protection (Rust Embedded Book) for a few KB |
| Keep the `unsafe` to the ABI crate, with `# Safety` contracts, and leave the engine crate `unsafe_code = "deny"` | the pointer dance is the one place memory safety can go wrong; a reviewer reads one file |

## One engine, identical output

| Rule | Why |
| --- | --- |
| **Integers only** in layout and geometry; no `f32`/`f64` in anything that reaches the output | wasm arithmetic is deterministic apart from NaN bit patterns, but Rust's `sin`, `cos`, `exp`, `powf`, `ln` are documented "non-deterministic ... varies by platform, Rust version"; `sqrt`, `floor`, `round` are exact. Integer maths has none of this |
| No `HashMap`/`HashSet` iteration into the output; `BTreeMap`, `IndexMap` or sorted `Vec` | std `HashMap` is randomly seeded per instance on native ("arbitrary order"), and on `wasm32-unknown-unknown` it is *not* randomised: the two builds would disagree in order, which is the worst kind of bug to find |
| No randomness, time, environment or locale in the engine | `std::time`, `std::env` are unavailable or wrong on the target, and they break "same input, same picture" anyway |
| **Golden test** for the output and a **bridge test** that the wasm entry point renders byte-identical to the CLI path for the same request | the golden shows every layout change in the diff; the bridge test catches a request field the CLI applies and the page forgot |
| **Staleness in the gate, not only at run time**: a hook rebuilds the module (or compares a fingerprint of the engine sources recorded beside the asset) and fails when the committed `.wasm` differs | a run-time checksum tells the *reader* the page is stale; the gate tells the *author* before the commit. Keep the run-time check too: it is what protects a page rendered by an old binary |
| The page starts from the state it was rendered with (view, orientation, warnings) and re-renders once at load to compare a checksum of the SVG | the first relayout is the stale-asset test; a mismatch switches relayout off and says which command rebuilds the asset |
| **Run the committed asset itself** once per gate: headless Chrome on a rendered fixture page, assert the page reports "wasm active" and zero console errors | the native bridge test never executes the `.wasm`; this is the only test of the module the user gets |

## The page itself

| Rule | Why |
| --- | --- |
| **No external request**: no CDN script, font or image; a test greps the page for `http://`, `https://` and `//` outside `href` of model links | one request makes the page a web page again (tracking, offline failure, a frozen CDN) |
| A **CSP `<meta>`** as the first element of `<head>`: `default-src 'none'; script-src 'unsafe-inline' 'wasm-unsafe-eval'; style-src 'unsafe-inline'; img-src data: blob:` plus what the page needs (`connect-src 'self'` for a local edit server) | CSP3: a `meta` policy governs only what follows it, so place it early; `frame-ancestors`, `report-uri`, `sandbox` are ignored in `meta`. `'wasm-unsafe-eval'` allows compiling wasm without `'unsafe-eval'` (MDN). `'unsafe-inline'` is unavoidable in a one-file page (a hash would change on every render; a nonce needs a server), so the gain is the `default-src 'none'`: nothing else can load, even if a label carries markup. Verified on Chrome 157 from `file://` |
| Escape model data into JS as JSON with `<` written `<`; attributes and text through one escaper | the model is user text in a `<script>`; `</script>` in a label ends the script otherwise |
| Size budget per page, stated in the README and tested: the fixed part (CSS, JS, wasm) and a rule for data (attachment limit, `--documents` switch) | a diagram page that weighs 20 MB is not sent; the wasm alone is about 1 MB in base64 at 800 KB raw |
| **Print stylesheet**: `@media print` hides toolbar, panel, tooltip and live regions, turns animation off, lets the SVG scale to the page; `@page { size: A3 landscape; margin: 8mm }` where the picture is wide | Chrome's `--print-to-pdf` uses the page's `@page` size and the print styles; the same rules serve Ctrl+P for the reader |
| Theme by `prefers-color-scheme` with a manual override stored in `localStorage` inside `try/catch`; **motion** off under `prefers-reduced-motion` (CSS and `matchMedia` for JS-driven motion) and a pause button | WCAG 2.3.3 (C39/SCR40: the query in CSS and JS); storage throws in some private windows |
| Phone width (375 px) and the print size are both looked at, as screenshots in the gate | a canvas sized for a monitor is unreadable on the device the page is opened on first |

## An SVG canvas people can use

| Rule | Why |
| --- | --- |
| Root `<svg>` has `role="graphics-document"` (with `document` or `group` as the fallback token) and `aria-labelledby` to a `<title>` and `<desc>`, **not `role="img"`** when children are interactive | WAI-ARIA Graphics: `graphics-document` is the role for charts, maps and diagrams whose children "may include links or interactive widgets"; under `img` the children are presentational and focusable boxes vanish from the accessibility tree |
| Boxes are `<g role="graphics-object">` (or `button`/`link` when they act) with `<title>` as the first child and `aria-label`; decorative groups get `role="none"` | graphics-object marks "a distinct object or sub-component with semantic meaning"; `<title>` is the accessible short text and the browser tooltip (MDN); SVG 2 §15.7: a rendered element with a valid `tabindex` is focusable, a non-rendered one never |
| **One tab stop for the canvas, arrows inside** (roving `tabindex`: the active box `0`, the rest `-1`), Enter/Space to open, Esc to close and return focus | WAI-ARIA APG: a composite is a single tab stop; a hundred boxes each with `tabindex="0"` are a hundred Tab presses before the next toolbar button |
| A visible focus ring that contrasts 3:1 with what it sits on; the selected state distinct from focus | WCAG 2.4.7 (AA) and 1.4.11 (AA, the ring is non-text content); APG: colours and gradients disappear in high contrast, so use an outline |
| **Forced colours**: `@media (forced-colors: active)` with system colours (`CanvasText`, `Canvas`, `Highlight`) where a fill carries meaning; a manual high-contrast toggle for everyone else | in forced-colors mode SVG `fill` and `stroke` attributes are overridden at paint time (MDN), so colour-coded lines become one colour: shapes and dash patterns must carry the meaning (the colour-is-never-the-only-cue rule) |
| A live region (`role="status"`, `aria-live="polite"`) announces relayout, fold, filter results and errors | a sighted user sees the picture move; a screen-reader user hears nothing otherwise |
| A text alternative of the whole canvas (a table view of nodes and edges) in the page | WCAG 1.1.1 for a complex image; it also helps search and agents |

## Headless checks

| Rule | Why |
| --- | --- |
| `chrome --headless --disable-gpu --dump-dom file:///…/page.html` and assert on the serialised DOM (a `body.wasm` class, a status text) | `--dump-dom` runs the scripts and prints the resulting DOM to stdout, unlike `curl`; `file://` works in headless Chrome, no server needed |
| `--virtual-time-budget=N` to fast-forward timers and animations, `--timeout=N` as the hard cap | `virtual-time-budget` "acts as a fast-forward for any time-dependent code" (`setTimeout`, `setInterval`); without it a 3 s timer costs 3 s per check |
| `--enable-logging=stderr --v=0` and grep stderr for `INFO:CONSOLE` lines; fail on any `console.error` | page console output appears as `[…:INFO:CONSOLE:5] "message", source: file:///… (5)` (verified on Chrome 157; the format is not in the Chromium docs: not found) |
| `--screenshot --window-size=375,812` and `--window-size=1600,1000` for the two looks; `--print-to-pdf --no-pdf-header-footer` for the PDF | the screenshot shows what the string tests cannot (clipping, overlap, unreadable contrast); `--no-pdf-header-footer` drops date, URL and page number |
| Use `--no-sandbox` only inside a container or CI sandbox; pass `--user-data-dir` to a scratch folder | the first keeps Chrome from refusing to start as root; the second keeps the user's profile out |
| The old `chrome-headless-shell` is a separate binary since Chrome 132; scripts call the normal `chrome --headless` | `--headless=old` has no effect any more (Chromium headless README) |

## Layout cost and the size budget (lessons from hq-diagram, 2026-11-07)

| Rule | Why |
| --- | --- |
| Keep an `#[ignore]` timing test on a generated graph (hundreds of boxes) next to the goldens, run in release | fixture goldens never show a cost cliff: Columns took 25 s for 300 boxes while every fixture laid out in milliseconds |
| Count calls (searches, routes, scorings) before capping sweeps | 18k crossing counts showed that a nested search per block, not the sweeps, was the cost; one threshold (skip block moves above 120 boxes) took it to 0.16 s and left every fixed model byte-identical |
| Put a cap in the shared layout, not the CLI | the page's wasm relayout runs the same code with no threads, so it stays responsive on a large model for free |
| Set a size ceiling test for the wasm and the page, and install `wasm-opt` (binaryen) in the pipeline | an unoptimised module sits close to the ceiling after two feature slices (895,953 of 900,000 bytes), so the next change forces a ceiling bump |
| Keep serve-only code (share, edit server) in the binary crate, not the library | a library change invalidates the wasm fingerprint hook and forces a rebuild |
| After a landing, tell the maintainer to restart a running `serve` | it keeps its old binary: the maintainer saw "no changes" from a stale server |

## Sources

All read 2026-11-07.

- Rust and WebAssembly book, [Shrinking .wasm size](https://rustwasm.github.io/docs/book/reference/code-size.html): profile, `wasm-opt -Os/-Oz` 15-20 %, panics, allocator, formatting, twiggy.
- wasm-bindgen, [Optimizing for size](https://rustwasm.github.io/docs/wasm-bindgen/reference/optimize-size.html), [Introduction](https://rustwasm.github.io/docs/wasm-bindgen/introduction.html), [Without a bundler](https://rustwasm.github.io/docs/wasm-bindgen/examples/without-a-bundler.html) (cannot open `index.html` directly, CORS); wasm-pack, [Cargo.toml configuration](https://rustwasm.github.io/docs/wasm-pack/cargo-toml-configuration.html) (`wasm-opt` flags).
- Binaryen, [README](https://github.com/WebAssembly/binaryen/blob/main/README.md) (prebuilt releases, runs on the linked module); [`wasm-opt` crate](https://docs.rs/wasm-opt/latest/wasm_opt/) (`-Os`, `-Oz`, features).
- Cargo, [Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html); rustc, [wasm32-unknown-unknown](https://doc.rust-lang.org/rustc/platform-support/wasm32-unknown-unknown.html) (`-Cpanic=abort` default, dlmalloc, std parts erroring, HashMap not randomised); std, [HashMap](https://doc.rust-lang.org/std/collections/struct.HashMap.html), [f64](https://doc.rust-lang.org/std/primitive.f64.html) (unspecified precision); Rust Embedded Book, [no_std](https://docs.rust-embedded.org/book/intro/no-std.html).
- WebAssembly design, [Nondeterminism](https://github.com/WebAssembly/design/blob/main/Nondeterminism.md).
- MDN: [Loading and running](https://developer.mozilla.org/en-US/docs/WebAssembly/Guides/Loading_and_running), [`WebAssembly.Module()`](https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/JavaScript_interface/Module/Module), [CSP `script-src`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/script-src) (`wasm-unsafe-eval`, hashes, nonces), [Same-origin policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) (file origins), [Base64](https://developer.mozilla.org/en-US/docs/Glossary/Base64), [`Uint8Array.fromBase64`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array/fromBase64), [data: URLs](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Schemes/data), [forced-colors](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors), [prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion), [Printing](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_media_queries/Printing), [SVG `<title>`](https://developer.mozilla.org/en-US/docs/Web/SVG/Reference/Element/title).
- W3C: [CSP3 §3.3 meta element](https://www.w3.org/TR/CSP3/#meta-element); [WAI-ARIA Graphics Module](https://www.w3.org/TR/graphics-aria-1.0/); [SVG 2 §15.7 Focus](https://www.w3.org/TR/SVG2/interact.html); [ARIA APG Keyboard interface](https://www.w3.org/WAI/ARIA/apg/practices/keyboard-interface/); WCAG 2.2 [2.4.7](https://www.w3.org/WAI/WCAG22/Understanding/focus-visible.html), [1.4.11](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html), [2.3.3](https://www.w3.org/WAI/WCAG22/Understanding/animation-from-interactions.html).
- Chrome: [Headless command-line reference](https://developer.chrome.com/docs/automation-and-testing/headless-cli) (`--dump-dom`, `--screenshot`, `--print-to-pdf`, `--no-pdf-header-footer`, `--timeout`, `--virtual-time-budget`), [Headless mode](https://developer.chrome.com/docs/chromium/headless), [Chromium headless README](https://chromium.googlesource.com/chromium/src/+/main/headless/README.md), [Getting started with Headless Chrome](https://developer.chrome.com/blog/headless-chrome) (2017, `--window-size`, `--repl`), [How to enable logging](https://www.chromium.org/for-testers/enable-logging/).
- Observed locally, 2026-11-07, Google Chrome 157 on Linux: a `file://` page with a CSP `meta` (`default-src 'none'; script-src 'unsafe-inline' 'wasm-unsafe-eval'`) compiled an inline base64 module, `--virtual-time-budget=5000` fired a 3 s timer at once, `console.error` showed as an `INFO:CONSOLE` line on stderr; `hq-diagram`'s rendered fixture page activated its embedded module (`<body class="wasm">`) with zero console lines; its module is 796 KB raw, 227 KB gzip, with a `producers` section left after `strip = true`.
- Not found: an official statement of Chrome's synchronous-compile size limit; official documentation of the `INFO:CONSOLE` log line format; a measured `-Os` vs `-Oz` difference for Rust output (measure per module).
