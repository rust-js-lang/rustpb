# Proving a JS build equals its native build: techniques, tools, precedents, pitfalls

Scope: how to test that rustpb's generated Rust, compiled to JavaScript by rust-js, behaves exactly like the native build. Read against rustpb `docs/design.md` §6 (proof) and §7 (rust-js side). Researched 2026-11-13. Local clones were read wherever they existed. These were scala-js @5cc1be672 (2026-09-15), gleam @79c0cdd (2026-09-26), protobuf @8e2636c (2026-11-13), rust-js main @874e4b368, connect-rust, and protobuf-es @76e269e (2026-11-09, v2.17.0, cloned `--depth=1` for this note). Web sources were used for everything else.

Local-file citations use absolute paths (`file://` links). They describe the clone at the commit above.

---

## Precedents of "run the same test suite on both targets"

### Takeaway
Every mature cross-target compiler uses the same pattern:
- **one shared test source set**, compiled for both targets;
- **a typed platform query or attribute** for differences that are known and decided;
- **a checked deny-list** for tests that cannot run, so a stale entry fails the build.

Scala.js is the most complete model: shared, JVM-only and JS-only test trees, a `Platform` object of capability flags, and per-version blacklists of upstream Scala tests that are validated against the real file set. Gleam runs one language test suite on four runtimes.

### Cited Findings
- **Scala.js layout.** `test-suite/shared/src/test/scala` holds 258 Scala test files. `test-suite/js/src/test` holds 114, split into folders by required feature (`require-async-await`, `require-esmodule`, `require-modules`, …). `test-suite/jvm` holds 1. — [scala-js/test-suite](https://github.com/scala-js/scala-js/blob/HEAD/test-suite) (counted with `find … -name '*.scala'`)
- **Scala.js runs the shared tests on the JVM too.** The `testSuiteJVM` project ("Scala.js test suite on JVM") forks the JVM and forces `-Dfile.encoding=UTF-8`, an empty locale, `-Duser.timezone=Etc/GMT` and `-Dline.separator=\n`. The comment explains: "Scala.js always assumes Locale.ROOT, UTF-8 and NL as line separator by default … we have to force the same values on the JVM." — [scala-js/project/Build.scala L2542–2560](https://github.com/scala-js/scala-js/blob/HEAD/project/Build.scala)
- **Scala.js platform flags.** Each side has its own `Platform` object. The JS one sets `executingInJVM = false` and exposes `executingInWebAssembly`, `executingInNodeJS`, `assumedESVersion`, `jsBigInts`, `typedArrays`, `hasCompliantAsInstanceOfs`, `hasCompliantNullPointers`, `hasCompliantArrayIndexOutOfBounds`, `regexSupportsLookBehinds`, `isESModule` and others. The values come from the build (`BuildInfo`). — [scala-js/test-suite/js/.../utils/Platform.scala](https://github.com/scala-js/scala-js/blob/HEAD/test-suite/js/src/main/scala/org/scalajs/testsuite/utils/Platform.scala)
- **How shared tests use the flags.** They gate on these flags with JUnit `assumeTrue`/`assumeFalse` (94 call sites). The most frequent flags are `hasCompliantAsInstanceOfs` (11) and `executingInJVM` (8+4). A skipped case is therefore tied to a named semantic difference, not to "JS" in general. — grep over [scala-js/test-suite/shared](https://github.com/scala-js/scala-js/blob/HEAD/test-suite/shared/src/test/scala)
- **Scala.js runs Scala's own JUnit tests.** `scala-test-suite` compiles every upstream Scala JUnit test except those in a per-Scala-version `BlacklistedTests.txt` (241 lines for 2.13.12, grouped under headings such as "Do not compile"). The build throws `AssertionError("Sources not found for blacklisted tests: …")` when a listed file no longer exists, so the deny-list cannot rot silently. — [Build.scala L2836–2870](https://github.com/scala-js/scala-js/blob/HEAD/project/Build.scala), [BlacklistedTests.txt](https://github.com/scala-js/scala-js/blob/HEAD/scala-test-suite/src/test/resources/2.13.12/BlacklistedTests.txt)
- **Scala.js runs partest too.** `partest-suite` keeps per-Scala-version `BlacklistedTests.txt` and expected `.check` files (150 resource files). — [partest-suite resources](https://github.com/scala-js/scala-js/blob/HEAD/partest-suite/src/test/resources/scala/tools/partest/scalajs)
- **Some Scala.js checks are still manual.** `TESTING.md` lists HTML test runners with ES modules and WebAssembly ("because jsdom does not support modules yet") and source-map stepping in Chrome. — [scala-js/TESTING.md](https://github.com/scala-js/scala-js/blob/HEAD/TESTING.md)
- **Scala.js in real browsers.** It swaps the JS *environment* and keeps the tests: `Test / jsEnv := new PWEnv(browserName = "chrome")` (Playwright env) or `SeleniumJSEnv`. The test framework runs inside the page and sbt reads the results back. This summary is from rust-js's survey of the local clones. — [rust-js ADR 0027](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0027-real-browser-tests.md)
- **Gleam.** `test/language` is one Gleam test package. Its Makefile runs it as `gleam test --target erlang`, `--target javascript --runtime nodejs`, `--runtime deno` and `--runtime bun`. CI has a step per runtime ("test/language JavaScript with NodeJS/Deno/Bun"). Target-only tests are marked `@target(erlang)` / `@target(javascript)`, and the FFI has per-target files (`ffi_erlang.erl`, `ffi_javascript.mjs`). — [gleam/test/language/Makefile](https://github.com/gleam-lang/gleam/blob/HEAD/test/language/Makefile), [ci.yaml L472–486](https://github.com/gleam-lang/gleam/blob/HEAD/.github/workflows/ci.yaml), `bit_array_test.gleam:161`
- **Kotlin Multiplatform.**
  - `commonTest` depends on `kotlin.test`. "A platform-specific framework will provide the test runner" for each target, and Gradle `allTests` runs every target "with the corresponding test runner". Platform differences use `expect`/`actual`. The page shown covers only Android/iOS, not the JS runner. — [Kotlin docs: run tests](https://kotlinlang.org/docs/multiplatform/multiplatform-run-tests.html)
  - Kotlin/JS can map `Long` to BigInt (ES2020) with the **experimental** `-Xes-long-as-bigint` flag. Exporting `Long` also needs `-XXLanguage:+JsAllowLongInExportedDeclarations`. — [Kotlin JS interop docs](https://kotlinlang.org/docs/js-to-kotlin-interop.html)
  - A community thread reports that Kotlin's own Long emulation is faster than BigInt for in-range values. This is opinion, not a benchmark. — [Kotlin Slack archive](https://slack-chats.kotlinlang.org/t/8192771/can-anyone-explain-why-long-math-is-extremely-slow-compared-)
- **GHC JS backend.**
  - A 2023 merge added "CI: JavaScript backend runs testsuite", built on the cross compiler. Many tests were marked broken or skipped for JS. — [ghc-commits, Feb 2023](https://mail.haskell.org/pipermail/ghc-commits/2023-February/051797.html), [ghc-commits Nov 2022](https://mail.haskell.org/pipermail/ghc-commits/2022-November/050019.html)
  - Perf tests had to be re-enabled separately, because the JS backend was treated as a stage1 compiler. — [IOG engineering update 2023-07-13](https://engineering.iog.io/2023-07-13-ghc-update)
- **js_of_ocaml.**
  - `compiler/tests-jsoo/test_ints.ml` checks `Int64.of_string`/`Int32.of_string` on boundary and overflow values as `let%expect_test` (ppx_expect) blocks. The same expect tests are the oracle on every backend. — [Debian sources, js-of-ocaml 6.2.0](https://sources.debian.org/src/js-of-ocaml/6.2.0-2/compiler/tests-jsoo/test_ints.ml)
  - wasm_of_ocaml runs the suite with `dune build @runtest-wasm` and forked ppx_inline_test/ppx_expect. — [js_of_ocaml 6.4.0 README_wasm_of_ocaml](https://ocaml.org/p/js_of_ocaml/6.4.0/README_wasm_of_ocaml.html)
- **wasm-bindgen-test (Rust → wasm).**
  - Tests run on Node by default. `wasm_bindgen_test_configure!(run_in_browser)` moves them into a browser.
  - `wasm-pack test --headless --chrome --firefox --safari` runs them under each browser it is given. — [wasm-bindgen guide: headless browsers](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/browsers.html), [wasm-pack test](https://rustwasm.github.io/docs/wasm-pack/commands/test.html)
  - Worker modes (`run_in_dedicated_worker`, etc.) were found only on a mirror. Unverified.
- **rust-js already does the same thing for itself.**
  - `--test` compiles a program's `#[test]`s for Node's test runner by default (ADR 0026/1076). `--cfg=browser` turns on tests that need a real browser.
  - The same compiled tests run in Playwright (Chromium, Firefox, WebKit) and Vitest browser mode, on Bun:
    `RUST_JS_TESTS=out/app.test.js bunx --bun playwright test -c browser/playwright.config.ts`
    — [rust-js ADR 0027](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0027-real-browser-tests.md), [rust-js test/browser.test.ts](https://github.com/rust-js-lang/rust-js/blob/HEAD/test/browser.test.ts)

### Inferences
- For rustpb, the closest precedent is Scala.js plus rust-js's own `--test` mode:
  1. Write rustpb's tests once as ordinary `#[test]`s.
  2. Run them with `cargo test` natively and as rust-js `--test` output on Node, Bun and Playwright browsers.
  3. Gate decided differences on a single named `cfg` or constant, Scala.js-`Platform` style, never on ad-hoc `#[cfg(target_arch)]`.
- Copy the Scala.js stale-list check and the conformance runner's "passing listed test fails" rule (next section) for any JS skip list. A JS skip list that cannot fail when it goes stale is the main way such projects lose trust.
- Scala.js's forced JVM locale, encoding and timezone is a reminder: hold native test runs to JS's fixed environment too, not just the other way round. For rustpb that probably matters little, since there is no locale-sensitive formatting, but float-to-string in JSON and map iteration order are similar "ambient" differences.

### Gaps
- I did not read Kotlin/JS's current JS test runner (Karma/Mocha/Node) from primary docs; the fetched page covered only Android and iOS.
- I found nothing current on GHC JS backend testsuite status beyond 2023.
- The wasm-bindgen worker test modes are unverified.

---

## JS protobuf libraries' test setups

### Takeaway
protobuf-es is the reference.
- It runs Google's `conformance_test_runner` against a Node testee, with an exact failure list per configuration (with BigInt: empty; without BigInt: 44 lines, all unquoted int64/uint64 JSON numbers).
- It runs the same conformance separately on Bun and Deno.
- Its unit tests use `node:test`, and its bundle-size table is published against google-protobuf.

The upstream runner's failure list is strict: a listed test that passes is reported as a failure that must be fixed in the list.

### Cited Findings
- **protobuf-es conformance scripts.**
  - `test:bigint` runs `conformance_test_runner --maximum_edition MAX --enforce_recommended --failure_list failing_tests_with_bigint.txt --text_format_failure_list failing_tests_text_format.txt bin/conformance.js`.
  - `test:string` runs the same with `failing_tests_without_bigint.txt`, chosen when `BUF_BIGINT_DISABLE=1`.
  - The runner comes from the npm package `protobuf-conformance` 36.2.0.
  - — [protobuf-es/packages/protobuf-conformance/package.json](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf-conformance/package.json)
- **The failure lists.**
  - With BigInt: 0 lines. Without BigInt: 44 lines. Text format: 3 lines.
  - The no-BigInt list explains: "a JavaScript runtime can only safely represent values in the range from Number.MIN_SAFE_INTEGER to Number.MAX_SAFE_INTEGER. That means number values in JSON will be unsafe as well."
  - Its entries are all `*.JsonInput.{Int64,Uint64}Field{Max,Min}ValueNotQuoted.*`.
  - — [failing_tests_without_bigint.txt](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf-conformance/failing_tests_without_bigint.txt)
- **The testee.** It reads all requests from `process.stdin` with `readMessages(…, ConformanceRequestSchema)` and writes responses to `process.stdout`. — [conformance.ts L73–86](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf-conformance/src/conformance.ts)
- **The wire protocol.** The tester sends a 4-byte little-endian length N, then N bytes of `ConformanceRequest`. The testee answers with a 4-byte LE length M and a `ConformanceResponse`. — [protobuf/conformance/conformance_test_runner.cc L13–30](https://github.com/protocolbuffers/protobuf/blob/HEAD/conformance/conformance_test_runner.cc)
- **Upstream failure-list semantics.**
  - A test on the list that succeeds is reported: "These tests succeeded, even though they were listed in the failure list … Remove their match from the failure list."
  - The runner also reports list entries that match no test, and wildcard entries that match too many.
  - — [protobuf/conformance/conformance_test.cc L369–735](https://github.com/protocolbuffers/protobuf/blob/HEAD/conformance/conformance_test.cc)
  - Upstream keeps a `failure_list_<lang>.txt` per implementation (cpp, csharp, java, java_lite, jruby, objc, php, python, dart_upb, …). It also ships `conformance_rust.rs`. — [protobuf/conformance/](https://github.com/protocolbuffers/protobuf/blob/HEAD/conformance)
- **protobuf-es 64-bit support.**
  - `protoInt64` is built once by `makeInt64Support()`. It uses BigInt only if `typeof BigInt === "function"` and `DataView` has `getBigInt64`/`getBigUint64`/`setBigInt64`/`setBigUint64` ("Safari 14 implements BigInt, but not the DataView methods"), and if `BUF_BIGINT_DISABLE !== "1"`.
  - Otherwise it falls back to strings, checked by `assertInt64String`/`assertUInt64String`.
  - — [protobuf/src/proto-int64.ts L121–235](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf/src/proto-int64.ts)
- **protobuf-es CI matrix.**
  - `ci.yaml` runs unit tests and conformance on Node 22.x, 24.x and 26.x, each × {bigint, fallback} via `BUF_BIGINT_DISABLE`.
  - `bun.yaml` (Bun 1.x) and `deno.yaml` (Deno LTS) each run `turbo run test -F './bun/conformance'` / `'./deno/conformance'`, plus a `gh-diffcheck` step that fails on uncommitted generated changes.
  - No Playwright, Puppeteer or Karma dependency was found in any package.json, so conformance does not run in browsers.
  - — [protobuf-es/.github/workflows](https://github.com/bufbuild/protobuf-es/blob/HEAD/.github/workflows)
- **protobuf-es unit tests.** They run as `tsx --test $(find src -type f -path '*.test.ts')` (Node's built-in runner). Fixtures cover proto2, proto3, editions 2023, maps, oneofs, JSON names, `jstype`, extensions and WKT wrappers (`packages/protobuf-test/extra/*.proto`). — [protobuf-test/package.json](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf-test/package.json)
- **protobuf.js.**
  - `toObject`'s `longs` option can be BigInt, String or Number, and Number "may be lossy".
  - Exact 64-bit needs the `long` package.
  - 64-bit map keys are stored as an 8-character hash string.
  - `fromObject` takes decimal strings, `Long` or `BigInt`.
  - — [protobufjs 8.7.x README (jsDelivr)](https://cdn.jsdelivr.net/npm/protobufjs@8.7.0/README.md)

### Inferences
- rustpb's JS build can be a conformance testee exactly as protobuf-es is. Native rustpb gets its own `failure_list_native.txt` and the JS build its own list. The strongest check is that **the two lists are identical** once the JS build is blocking: any line in one and not the other is a native/JS divergence. This can be a test of its own. (Design §6.5 later splits each list into a measured ScalaPB baseline and a temporary unsupported list, so native/JS equality holds while features are still missing.)
- Because rustpb maps u64/i64 to BigInt unconditionally, the protobuf-es "without BigInt" configuration does not apply. But its 44 failures show where a JS build is weakest: **int64 values as unquoted JSON numbers**. rustpb's JSON is read by rustpb's own reader (design §5), plain Rust compiled by rust-js, not `JSON.parse`. That is worth one targeted test row: parse `{"x": 18446744073709551615}` natively and as JS and compare.
- protobuf-es tests Node, Bun and Deno, but no browser, for conformance. The conformance runner needs stdin/stdout, so browsers are covered only by rustpb's own e2e (§6.11), not by conformance.

### Gaps
- I did not clone protobuf.js or protobuf-javascript (google-protobuf). Their test runners and conformance status (whether google-protobuf JS has a conformance testee) are unverified.
- I did not find whether the `protobuf-conformance` npm package is maintained by upstream or by Buf.

---

## Cross-runtime and browser testing; what actually differs between engines

### Takeaway
The behaviours that change results across JS engines are few and mostly known:
- maximum string length (V8 is far smaller than JavaScriptCore);
- the stack-overflow error type (`RangeError` vs Firefox's `InternalError`);
- `JSON.parse` precision above 2^53;
- `TextDecoder`'s default replacement vs `fatal`;
- historic gaps like Safari 14's missing `DataView` BigInt methods.

Playwright covers Chromium, Firefox and WebKit on one CI host. rust-js already wires this up (ADR 0027).

### Cited Findings
- **Maximum string length per engine:**

  | Engine | Max UTF-16 units |
  |---|---|
  | V8 (Chrome, Node, Deno) | 2^29 − 24 (2^28 − 16 on 32-bit) |
  | SpiderMonkey (Firefox) | 2^30 − 2 |
  | JavaScriptCore (Safari, Bun) | 2^31 − 1 |

  Exceeding it throws `RangeError`. — [rust-js ADR 0682](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0682-a-js-strings-length-is-at-most-int32-max.md) (cites V8 `v8-primitive.h`, SpiderMonkey `String.h`, WebKit `StringImpl.h`)
- **Stack overflow is engine-specific.** V8 and JavaScriptCore throw `RangeError: Maximum call stack size exceeded`; SpiderMonkey throws `InternalError: too much recursion`. rust-js had to classify both. — [rust-js ADR 0692](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0692-a-thrown-error-is-a-panic-and-what-js-threw.md)
- **`TextDecoder` defaults.**
  - `fatal` is false by default: invalid bytes become U+FFFD. `fatal: true` throws `TypeError`.
  - The BOM is stripped by default, unless `ignoreBOM: true`.
  - Baseline widely available since January 2020.
  - — [MDN TextDecoder()](https://developer.mozilla.org/en-US/docs/Web/API/TextDecoder/TextDecoder)
- **How rust-js decodes UTF-8.** It validates with a port of `core::str::validations` (so `valid_up_to`/`error_len` and messages are Rust's) and uses `TextDecoder` only for valid bytes. `from_utf8_lossy` relies on `TextDecoder` replacing "by the same maximal subpart" as Rust's `Utf8Chunks`; a mutation that replaced only the first bad sequence was caught. — [rust-js ADR 0172](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0172-utf8-decoding.md)
- **BigInt over DataView.** Safari 14 had BigInt but not `DataView.getBigInt64` etc.; protobuf-es feature-tests both. — [proto-int64.ts L129](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf/src/proto-int64.ts)
- **`JSON.parse` source text access.**
  - The reviver's `context.source` exposes a number's raw text, which is the standard way to read int64 JSON numbers exactly. It is Stage 4 (spec draft dated 2025-11-23).
  - The web-features explorer lists it as "Limited availability": Chrome/Edge 114 and Firefox 135, Safari not listed. — [TC39 proposal](https://tc39.es/proposal-json-parse-with-source/), [web-features: json-raw](https://web-platform-dx.github.io/web-features-explorer/features/json-raw)
  - A TypeScript PR claims all major browsers and Node 22+. This is secondary and conflicts with the explorer. — TypeScript PR #63248 (seen only through a mirror; unverified)
- **Playwright in CI.**
  - The docs recommend `workers: process.env.CI ? 1 : undefined` "to prioritize stability and reproducibility". Sharding across CI jobs is the way to parallelise.
  - Install with `npx playwright install --with-deps`, or use the image `mcr.microsoft.com/playwright:v1.64.1-noble`.
  - "Always set a global timeout in CI."
  - — [Playwright: CI](https://playwright.dev/docs/ci)
- **rust-js's browser runs.**
  - Playwright's Chromium, Firefox and WebKit, plus Vitest browser mode, both on Bun.
  - happy-dom is kept for fast tests, but "does no layout" and lacks some real-engine APIs.
  - The test harness currently launches `chromium.launch({ headless: true })` in 12 places.
  - — [ADR 0027](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0027-real-browser-tests.md), grep of rust-js `test/`
- **Bun and Node in rust-js.** The serde probes run each program "on Bun and Node", comparing each outcome with the recorded native answer. — [rust-js TESTS.md L28](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md)

### Inferences
- For rustpb the engine axis matters less than the Rust/JS axis. rust-js owns number, string and byte semantics and has already pinned them against native (ADR 0172, 0682, 0692). A rustpb JS build on V8 and on JavaScriptCore should agree unless:
  - a value approaches engine size limits (a 1 GB `string` field fails earlier on V8 than on Bun or Safari, which is relevant to rustpb's size limits in §4.6);
  - a depth bomb hits a different stack depth per engine. Design §6.4's "depth bombs … no stack overflow" must hold on the smallest stack. Firefox's `InternalError` is the odd one out.
- A practical matrix: corpus and conformance on Node (V8) and Bun (JSC) for every push; e2e on Chromium, Firefox and WebKit via Playwright. Firefox (SpiderMonkey) is otherwise reached only through the browser run.
- Pitfall, standard JS behaviour not sourced here: `JSON.stringify` throws `TypeError` on a BigInt, and `===` between a BigInt and a Number is always false. A harness that serialises JS results to compare with native must print BigInts as decimal strings itself. It must also never compare via `JSON.stringify` or `==`.

### Gaps
- I found no primary source cataloguing `TextDecoder` or typed-array behavioural differences *between* current engines. The documented differences are the limits above, not semantic divergences.
- Playwright's page did not state WebKit-on-Linux support or any retry policy.
- I have no data on headless CI minutes per browser.

---

## Differential native-vs-JS testing, fuzzing, and rust-js machinery RustPB can reuse

### Takeaway
rust-js's whole method is differential:
- every corpus program runs natively and as JS, and stdout must match;
- generated tables, metamorphic rewrites, neighbour shuffles, determinism checks, mutations and a reducer widen the inputs.

Its stated lessons apply directly to rustpb: generate the space, two oracles beat one, a batch whose cases are all refused must fail, and every accepted difference must be re-tested.

For JS-side fuzzing the main coverage-guided tool (Jazzer.js) is deprecated on npm. Generating inputs natively and *replaying* them through the JS build is the practical route.

### Cited Findings
- **The corpus.**
  - 1,580 Rust programs (`test/corpus/*.rs`); each one's JS must print what native Rust prints and match its snapshot.
  - Each is also built as a library: per push only the 120 marked `library-differs`, nightly every one.
  - The full suite is 3,849 tests in 172 files and takes 11–13 min on 2 machines; the corpus is about half of it.
  - — [rust-js TESTS.md L10–12](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md)
- **Which nets found the most bugs.**
  - Of 357 P0s (2026-11-01 to 11-10): 144 found by an audit beside another fix, 76 by generated tables, 57 by the metamorphic net, 23 by ports, 11 by the consumer net, 30 unlabelled.
  - Each P0 carries a `found-by:` label, counted by `bun scripts/found-by.ts`.
  - — [rust-js docs/finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md)
- **The numbers grid.**
  - 114 operations × 14 types × boundary values: 0, ±1, MIN, MAX, their neighbours, values near 2^31, 2^32, 2^53 and 2^63, and floats' ±0, ±inf, NaN and subnormals.
  - 83,017 cases against native in both builds; a slice runs per push of the numbers code.
  - First run found an `f32` `div_euclid` wrongly rounded as `f64`.
  - — [finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md)
- **The metamorphic net.** Each corpus case is rewritten 39 ways by a `syn` rewriter (boxing, `Option`, `Rc<RefCell>`, generic, items moved into a library crate, …), and native's run of the rewrite is the oracle. Round 2 found 147 rewrites differing, 97 of them from moving items into a library crate. — [finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md), [TESTS.md L26](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md)
- **The neighbour shuffle.** Cases are grouped about 13 to a crate; a group that differs is narrowed to its victim plus the fewest neighbours needed. It found bugs that appear only beside an unrelated function that sets a per-type fact. The stated lesson: "Context is part of the input." — [finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md)
- **Determinism.** Each case is compiled twice in separate processes and its JS, source map and manifest compared byte for byte. This found #866: a `HashSet` walk order made `n + 1` wrap in 5 compiles of 8. — [finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md)
- **serde/serde_json probes.**
  - 358 one-statement programs, each built natively with pinned serde and with rust-js three ways (program, library, producer/consumer split), run on Bun and Node.
  - Result on 2026-11-13: "281 same, 85 refused, 3 native-invalid, none different".
  - `test/serde-std.test.ts` checks std types' serde forms against native serde_json, to and from text and from a `Value`.
  - — [TESTS.md L28](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md); commit 610760bd9 in `git log`
- **The rest of the toolbox.** Generated programs run under Miri first, then as JS (ADR 0092). There is also Rustlantis MIR fuzzing, a small Crater (real crates' own `#[test]`s, natively and as JS), mutations (ADR 0093), a C-Reduce-style reducer for every net's failures (ADR 1107), and decision coverage, which lists compiler decisions no program reaches (ADR 1124). — [finding-wrong-answers.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md), [TESTS.md](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md)
- **rust-js's eight lessons.**
  1. Generate the space, don't sample it by hand.
  2. Context is part of the input.
  3. A refusal is safe; newly compiling shapes are where wrong answers start.
  4. Unwind paths are where orders differ.
  5. Equivalence needs the full rows.
  6. Every accepted difference is a claim to re-test.
  7. Two oracles beat one (a second lowering, `--std-iterators`).
  8. "A program all refused passes all": the harness fails a batch whose every case is refused.

  — [finding-wrong-answers.md "The lessons"](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/finding-wrong-answers.md)
- **rust-js's rule for Rust/JS disagreement.** The difference must be one of: none; rare and harmless, accepted only by explicit user decision; results change, so preserve Rust; or cannot preserve, so compile error. — [rust-js AGENTS.md / ADR 0262](https://github.com/rust-js-lang/rust-js/blob/HEAD/docs/decisions/0262-when-rust-and-js-disagree.md)
- **A batching protocol to copy.** The conformance protocol streams length-prefixed requests to one long-lived testee process over stdin/stdout, so process start-up is paid once per run, not per case. — [conformance_test_runner.cc](https://github.com/protocolbuffers/protobuf/blob/HEAD/conformance/conformance_test_runner.cc), [protobuf-es conformance.ts](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/protobuf-conformance/src/conformance.ts)
- **JS fuzzing tools.**
  - The `@jazzer.js/fuzzer` and `@jazzer.js/jest-runner` npm packages are listed as **deprecated**, with the latest release about two years ago. — [ecosyste.ms: @jazzer.js/fuzzer](https://packages.ecosyste.ms/registries/npmjs.org/packages/@jazzer.js%2Ffuzzer/versions), [Snyk advisor](https://snyk.io/advisor/npm-package/@jazzer.js/fuzzer)
  - OSS-Fuzz's JavaScript guide still says JS fuzzing there is powered by Jazzer.js. — [OSS-Fuzz JS guide](https://google.github.io/oss-fuzz/getting-started/new-project-guide/javascript-lang/)
  - Whether the GitHub repo is archived was not confirmed.

### Inferences
- **The rustpb differential harness.** The cheapest strong design reuses the conformance shape:
  1. Build one "testee" binary natively and once as rust-js JS.
  2. Each reads a length-prefixed stream of `(message type, mode, input bytes or JSON)`.
  3. Each writes a canonical result: re-encoded bytes in hex, a canonical `Debug` of the decoded value with 64-bit numbers as decimal, or the `DecodeError` text.
  4. Diff the two output streams.

  One process per side per batch keeps it fast; the corpus, the fixtures (§6.4) and fuzz outputs all go through the same pipe.
- **Fuzzing.** Since coverage-guided JS fuzzing is weak (Jazzer.js deprecated), fuzz natively with cargo-fuzz/libFuzzer, which design §6.7 already plans, and **replay the whole saved fuzz corpus plus every crash through the JS testee** on each change. The JS build then inherits native coverage without a JS fuzzer.
- **Reusing rust-js's machinery.** rustpb could add its generated code to rust-js's crate corpus or small Crater, so rust-js's nightly runs rustpb's own `#[test]`s natively and as JS. The 64-bit varint two-halves path (§7) is a candidate for a numbers-grid-style table: every varint length 1–10 × boundary values near 2^31, 2^32, 2^53, 2^63 and 2^64−1, natively vs JS.
- **"All refused passes all" applies to rustpb.** If rust-js refuses a generated module, every JS case in that batch is vacuous. The harness must count JS cases actually run and fail when that count is zero, or below native's.
- **Determinism applies to `rustpb-build`'s output too.** Generate twice and compare byte for byte, as rust-js's determinism net does. HashMap-ordered codegen is the documented failure mode.

### Gaps
- jsfuzz's current maintenance status was not checked.
- No source was found on differential native-vs-JS fuzzing of a protobuf implementation specifically; this appears to be novel.

---

## End-to-end browser tests against a real server

### Takeaway
connect-rust's model is a shell script:
1. build and start the real native server on a fixed port;
2. poll it with curl until ready (up to 30 × 0.1 s), failing fast if the process dies;
3. run `wasm-pack test --headless --firefox`;
4. kill the server in a `trap`.

The server enables permissive CORS. The tests cover a success path, an invalid URL and an unreachable server. rust-js's equivalent runner is Playwright (ADR 0027), not wasm-pack.

### Cited Findings
- **connect-rust's `test.sh`.**
  - `ADDR="${ADDR:-127.0.0.1:8080}"`; `cargo build ${CI:+--locked} -p eliza-example --bin eliza-server`; the server runs in the background.
  - A readiness loop `for i in $(seq 1 30)` sends `curl -X POST …/connectrpc.eliza.v1.ElizaService/Say -d '{"sentence":"healthcheck"}'`, checks `kill -0 "$SERVER_PID"`, and sleeps 0.1.
  - Then `wasm-pack test --headless --firefox examples/wasm-client`, with `trap cleanup EXIT` killing the server.
  - — [connect-rust/examples/wasm-client/test.sh](https://github.com/connectrpc/connect-rust/blob/HEAD/examples/wasm-client/test.sh)
- **The tests.** `tests/integration.rs` has three `#[wasm_bindgen_test]` async tests under `wasm_bindgen_test_configure!(run_in_browser)`:
  - `say_via_fetch_transport` (reply starts with "Hello");
  - `say_invalid_url` (error contains "invalid");
  - `say_unreachable_server` (`127.0.0.1:1`, error message non-empty).
  - — [integration.rs](https://github.com/connectrpc/connect-rust/blob/HEAD/examples/wasm-client/tests/integration.rs)
- **Server and CI.** The eliza server adds `tower_http::cors::CorsLayer::permissive()` (`examples/eliza/src/main.rs:247`). CI runs `./examples/wasm-client/test.sh` (`.github/workflows/ci.yml:293`). — [connect-rust ci.yml](https://github.com/connectrpc/connect-rust/blob/HEAD/.github/workflows/ci.yml)
- **The client.** It uses buffa and serde_json for messages, with web-sys `Request`/`Headers`/`RequestInit` for the fetch transport. — [wasm-client/Cargo.toml](https://github.com/connectrpc/connect-rust/blob/HEAD/examples/wasm-client/Cargo.toml)
- **Playwright flakiness guidance.** One CI worker, global timeout, sharding for scale. — [Playwright: CI](https://playwright.dev/docs/ci)

### Inferences
- connect-rust's example is **wasm**, not JS. It proves the transport pattern, not rust-js's output. rustpb's e2e (§6.11) should keep the shape (real native server, real browser, the readiness loop, the trap) but run rust-js output through rust-js's Playwright runner. That gives Chromium, Firefox and WebKit, where wasm-pack gives one browser per flag.
- Flakiness controls to copy and improve:
  - **Port.** Bind port 0 and have the server print its port, instead of hard-coding 8080: parallel CI jobs or a stale server collide on a fixed port.
  - **Readiness.** Keep the "process died" check in the readiness loop.
  - **Timeouts.** Give every step a hard timeout (which rust-js's AGENTS.md also demands).
  - **Retries.** Set Playwright `retries: 0`, so a flake is investigated rather than hidden. This is my inference; Playwright's own sample config uses `retries: 2` in CI.
- **What the e2e should assert.** Assert byte equality of what crossed the wire, not just "a reply arrived". The server echoes the raw body hex and its own decoded `Debug`, and the browser test compares both against its own encode and decode. That makes the e2e a differential test across the network for the §6.11 cases (mapped id, sealed oneof, `Unrecognized` enum, unknown fields), in binary and JSON.
- The connect-rust tests' negative cases (bad URL, unreachable server) are cheap and catch transport error-mapping. rustpb's equivalent negatives are a truncated body and a body with invalid UTF-8 in a `string`, decoded in the browser.

### Gaps
- No data on how flaky connect-rust's wasm job is in practice. Its CI history was not read.

---

## Performance and bundle-size regression testing for the JS output

### Takeaway
Bundle size is cheap to gate deterministically. protobuf-es publishes esbuild-minified-and-compressed sizes against google-protobuf, and size-limit fails CI past a per-bundle budget.

Runtime benchmarks are noisy on shared CI. rust-js deliberately keeps its JSON benchmark as a by-hand measurement "of a quiet machine, not a pass or a fail". CodSpeed's tinybench plugin is the common CI option for JS.

### Cited Findings
- **protobuf-es `bundle-size`.**
  - Method: generate googleapis with `protoc-gen-es` and with protobuf-javascript's `protoc-gen-js`, bundle an entry importing 1/4/8/16/32 files with esbuild, minify, and compress "like a web server would".
  - Example rows: Protobuf-ES with 1 file is 136,798 b bundled / 68,003 b minified / 16,041 b compressed; at 32 files, 24,934 b compressed. protobuf-javascript is 36,226 b compressed at 1 file and 75,983 b at 32.
  - — [protobuf-es/packages/bundle-size/README.md](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages/bundle-size/README.md)
  - protobuf-es also has a `protobuf-bench` package (not read). — [protobuf-es/packages](https://github.com/bufbuild/protobuf-es/blob/HEAD/packages)
- **size-limit.** It "checks every commit on CI, calculates the real cost of your JS for end-users and throws an error if the cost exceeds the limit". Budgets are per path with a limit (gzip optional), and its GitHub Action posts size changes as a PR comment. — [size-limit README v13.0.3 (jsDelivr)](https://cdn.jsdelivr.net/npm/size-limit@13.0.3/README.md)
- **CodSpeed for JS.** `withCodSpeed(new Bench())` from `@codspeed/tinybench-plugin` wraps a tinybench suite and falls back to plain tinybench locally. It requires tinybench ≥4, which is ESM-only, and runs in CI through `CodSpeedHQ/action` (docs show both v2 and v4). — [CodSpeed tinybench docs](https://codspeed.io/docs/benchmarks/nodejs/tinybench)
- **rust-js's JSON benchmark.**
  - `bun scripts/serde-bench.ts` writes and reads a document at 1, 10 and 96 MB in four shapes (deep, wide, long strings, maps), typed and as a `Value`, on Bun, Node and optimised native serde_json, beside each engine's `JSON`, plus bundle sizes of `size-programs.ts`' programs.
  - Its output is "a measurement of a quiet machine, not a pass or a fail"; run by hand, `--quick` takes about 4 min.
  - — [rust-js TESTS.md L29](https://github.com/rust-js-lang/rust-js/blob/HEAD/TESTS.md)
- **GHC's experience.** Performance tracking for the JS backend had to be added deliberately, as separate debug and release jobs. — [IOG engineering update 2023-07-13](https://engineering.iog.io/2023-07-13-ghc-update)

### Inferences
- **Gate size, measure speed.**
  - Size is deterministic, given the determinism check above. Commit a size table per representative module (one record type, the large sealed oneof from milestone 2, the whole API surface): bundled, minified and Brotli/gzip. Fail CI past a budget, as size-limit does, or simply diff a committed table.
  - Speed: run a fixed benchmark (decode and encode of the golden corpus, binary and JSON) natively and on Node and Bun, recorded as a trend. Gate only on gross regressions (e.g. >2×), or via an instruction-count tool such as CodSpeed. The 2× threshold is a judgement call, not sourced.
- **A benchmark for the §7 lowering decision.** Benchmark "compile the expansion" against "compiler-owned lowering", and Number-for-≤32-bit against BigInt for 64-bit fields. The Kotlin thread's claim that BigInt is slower than emulated longs is unverified and worth measuring in rustpb's own setting.
- **Compare against protobuf-es.** Publish a protobuf-es comparison on the same `.proto` set with the same esbuild-and-compress method. It is the obvious yardstick users will ask for, and the method is already documented.

### Gaps
- I did not read `protobuf-bench` or protobuf.js's benchmark suite.
- I did not confirm how CodSpeed measures JS: instrumented simulation vs wall time.
- No sources were found on CI-noise thresholds that projects actually use for JS microbenchmarks.
