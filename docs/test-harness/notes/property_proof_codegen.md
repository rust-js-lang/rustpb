# Property-based testing, formal verification and code-generator testing in Rust, for RustPB's harness

Scope: which Rust testing techniques fit which part of RustPB (runtime codec, TypeMapper, unknown fields, sealed oneofs, the default-construction graph, and the rustpb-build generator), with tools, versions, how serious projects use them, costs and pitfalls. State as of 2026-11-13.

Local clones read (commit, date): prost `894f4a7` (2026-11-12, release 0.14.6), buffa `1047169` (2026-10-20), connect-rust `bd2ab67` (2026-10-22), rust-protobuf `0118c79` (2025-09-21), ScalaPB (local clone, `proptest/` module). RustPB design read: `https://github.com/rust-js-lang/rustpb/blob/HEAD/docs/design.md` §3, §4, §6.

Correction the report writer must carry forward: design.md §3 decision 1 says prost's varint/zigzag arithmetic is "fuzzed and Kani-checked", and §6 item 10 says "Kani proofs, as prost's is". In prost at `894f4a7`, the only Kani harnesses are in `prost-types` (Duration/Timestamp conversions); the varint codec is covered by hand-written unit tests and proptest, not Kani. See question 2.

## 1. Property-based testing: proptest vs quickcheck vs bolero; generating values of generated types; round-trip, metamorphic and size properties; shrinking; seeds and regression files

### Takeaway
Use proptest (1.12.0) as the property engine, and have rustpb-build optionally emit generators for generated types (buffa already does this for `arbitrary`, routing custom types through their base type). Share one `arbitrary`-based generator between libFuzzer targets and in-`cargo test` properties (bolero's model) for the codec. Define "encode stability" over decoded-value equality wherever `HashMap` or unknown-field order is involved. A byte-for-byte check would be flaky.

### Cited Findings
- proptest's current version on docs.rs is 1.12.0. `cases` defaults to 256 (`PROPTEST_CASES`). `PROPTEST_RNG_SEED` sets the seed: "If the environment variable is undefined, a random seed is generated". Other knobs: `PROPTEST_RNG_ALGORITHM` (`xs` XorShift, `cc` ChaCha), `max_shrink_iters` (default `u32::MAX`; 0 disables shrinking), `max_shrink_time` (default 0, no limit), `fork` and `timeout` (per-case subprocess; both strategy and test must be deterministic), `max_global_rejects` (default 1024), and `PROPTEST_DISABLE_FAILURE_PERSISTENCE` — [proptest Config docs](https://docs.rs/proptest/latest/proptest/test_runner/struct.Config.html)
- Failing cases are persisted under a `proptest-regressions/` tree named after the test's source file and replayed before new cases. The book recommends committing them. proptest stores the *seed* that produced the failure, not the value, so a changed strategy may no longer reproduce the exact input — [proptest book: failure persistence](https://proptest-rs.github.io/proptest/proptest/failure-persistence.html)
- proptest describes itself as close to feature-complete, in "passive maintenance". It uses explicit `Strategy` objects with stateful (integrated) shrinking, while quickcheck shrinks statelessly from the output value. proptest can be up to roughly an order of magnitude slower at generating complex values. The quickcheck README itself says proptest "improves on the concept of shrinking". quickcheck's latest docs.rs version is 1.1.0 — [docs.rs quickcheck](https://docs.rs/crate/quickcheck/latest); [quickcheck README 1.0.3](https://docs.rs/crate/quickcheck/1.0.3/source/README.md); search summary of proptest docs via [docs.rs proptest](https://docs.rs/crate/proptest/^1)
- `proptest-state-machine`: you give a `ReferenceStateMachine` (abstract model: `init_state`, `transitions`, `apply`, `preconditions`) and a `StateMachineTest` (SUT: `init_test`, `apply` with postconditions, `check_invariants`). It shrinks in three passes: delete transitions from the back, shrink each transition, shrink the initial state. Caveats: preconditions are needed because shrinking removes enabling steps; sequential only; heavier tests — [proptest book: state machine testing](https://proptest-rs.github.io/proptest/proptest/state-machine.html)
- proptest-derive (latest 0.6.0) is marked experimental. Its source says self-recursive or mutually recursive types can't be derived. Recursion needs hand-written `prop_recursive(depth, desired_size, expected_branch_size, …)` strategies — [proptest-derive source](https://docs.rs/proptest-derive/latest/src/proptest_derive/lib.rs.html); [proptest recursive strategy](https://docs.rs/proptest/1.0.0/src/proptest/strategy/recursive.rs.html)
- The `arbitrary` crate's impls for recursive types (e.g. `struct Foo(Option<Box<Foo>>)`) avoid infinite recursion when input bytes run out — [arbitrary docs](https://docs.rs/arbitrary/1.3.1/arbitrary/trait.Arbitrary.html)
- bolero (0.13.4) is a fuzzing *and* property-testing front end. One harness runs under libFuzzer, AFL or honggfuzz, as a random test under `cargo test`, and under Kani. Sanitizers need nightly — [bolero README](https://docs.rs/crate/bolero/latest/source/README.md); [Kani blog: using Kani with bolero](https://model-checking.github.io/kani-verifier-blog/2022/10/27/using-kani-with-the-bolero-property-testing-framework.html); [Yoshua Wuyts, "bridging fuzzing and property testing"](https://blog.yoshuawuyts.com/bridging-fuzzing-and-property-testing)
- **buffa generates generator impls from codegen.** `CodeGenConfig::generate_arbitrary` emits `#[cfg_attr(feature = "arbitrary", derive(::arbitrary::Arbitrary))]` on generated types. For a field whose string or bytes type is substituted, it emits `#[arbitrary(with = ...)]` using `::buffa::__private` builders. These build the canonical `String`/`Vec<u8>` and convert through `From`, "so the substituted type needs no `Arbitrary` impl", and cover singular, optional, repeated, oneof and map key/value. CI lints and tests `buffa-test` with `--features arbitrary` — `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/src/lib.rs` (≈L1290–1306), `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/src/message.rs` (`arbitrary_builder`, ≈L1915), `https://github.com/anthropics/buffa/blob/HEAD/.github/workflows/ci.yml` L78–82
- **buffa's encode fuzz oracle** (`fuzz/fuzzers/encode_proto3.rs`) takes an `Arbitrary` `TestAllTypesProto3`. It skips inputs whose `compute_size` exceeds 64 KiB, because the serde_json `Value` tree ran a 14-hour fuzz run out of its 2 GB RSS (documented OOM, Mar 2026). It then asserts `compute_size == encode_to_vec().len()`, decodes, treats `RecursionLimitExceeded` as expected (Arbitrary builds deeper messages than the decode limit), re-encodes for stability, then JSON-round-trips — `https://github.com/anthropics/buffa/blob/HEAD/fuzz/fuzzers/encode_proto3.rs`
- **prost's round-trip oracle** (`tests/src/lib.rs::roundtrip`) runs: decode arbitrary bytes, check `encoded_len == buf1.len()`, decode again, then check `buf1 == buf2 == encode_to_vec()`. Its doc says the message "should be compiled with `BTreeMap` fields, otherwise the comparison may fail due to inconsistent `HashMap` entry encoding ordering". An `encoded_len <= data.len()` assertion is commented out ("TODO: Reenable this once sign-extension in negative int32s is figured out") — `https://github.com/tokio-rs/prost/blob/HEAD/tests/src/lib.rs` L153–214
- prost's runtime uses proptest for per-scalar-type `check_type`/`check_collection_type` properties (repeated and packed, every tag from `MIN_TAG..=MAX_TAG`) — `https://github.com/tokio-rs/prost/blob/HEAD/prost/src/encoding.rs` L319–775, L1156–1250
- **ScalaPB's own property suite generates random *schemas*, not only values.** `proptest/src/test/scala/SchemaGenerators.scala` and `GraphGen.scala` generate random proto files: namespaces, nested messages and enums, oneofs, cross-file references, and a reserved-word list avoided by construction. They compile them in-process with javac/scalac. `GeneratedCodeSpec` ("Java and Scala protos are equivalent", tagged `Slow`, `minSuccessful(20)`, `workers(1)`) then generates random values per schema. It checks that text→binary matches Java, that parse(serialize) is the identity, that Java parses Scala's bytes, and that the Java↔Scala conversions and text format round-trip — `https://github.com/scalapb/ScalaPB/blob/HEAD/proptest/src/test/scala/GeneratedCodeSpec.scala`, `SchemaGenerators.scala`, `GraphGen.scala`

### Inferences
- Engine choice: proptest for in-tree properties with good shrinking and committed regression files. Add the `arbitrary` crate plus cargo-fuzz (or bolero, to reuse one harness under `cargo test`, libFuzzer and Kani) for coverage-guided byte-level fuzzing. quickcheck adds nothing over these.
- Generators for generated types should come from rustpb-build behind a feature, as buffa does, not from proptest-derive. proptest-derive cannot do recursive messages, which proto schemas routinely have, and buffa's approach already solves custom-type substitution. For rustpb, a `TypeMapper` field should be generated as "arbitrary base value, then `to_custom`", discarding values that fail. A mapper that rejects most base values starves the generator, so it should be able to supply its own base strategy. This is the generator twin of design.md §6 item 9 (`to_base(to_custom(b)) == b`).
- Bound recursion in the generator by rustpb's own depth budget (design §4.6: encode checks depth too). Otherwise, as buffa found, generated values hit the limit and must be skipped, which wastes cases.
- Properties to state (each maps to a design rule):
  - `decode(encode(x)) == x`, with `PartialEq` including unknown fields.
  - `encoded_len(x) == encode(x).len()` (§4.6 "Size and write"). Include negative `int32` and `Unrecognized(-1)` packed, which buffa and prost single out.
  - Encode stability: `encode(decode(encode(x)))` equals `encode(x)` byte-for-byte only for messages without `HashMap` fields. Otherwise compare decoded values (prost's doc comment documents the `HashMap` ordering problem).
  - Merge metamorphic, over encoder output only: `decode(encode(x) ++ encode(y)) == merge(x, y)`. Over arbitrary bytes `decode(a ++ b) == merge(decode(a), decode(b))` is **false**: a decoded proto3 value forgets whether a default scalar was on the wire. For `int32 x = 1`, `b = <empty>` and `c = 08 00` both decode to 0, yet `08 01 ++ b` gives 1 and `08 01 ++ c` gives 0 (reproduced with protobuf-go's `wrapperspb.Int32Value`, review #2 R1). The encoder never writes `c`'s explicit default, so the restricted form is sound. For arbitrary bytes, state it on the decoder's builder: decoding `b` on top of the state left by `a` equals decoding `a ++ b`. It must hold *before* mapping for mapped fields, since §4.1 merges base values and maps once. So the property needs a base-level decode or a decoder hook, and cannot run over mapped values alone.
  - Unknown-field preservation: decode with a schema that lacks fields, re-encode, decode with the full schema, and get the same value.
- ScalaPB's random-schema generator is the strongest precedent for testing rustpb-build *and* the runtime together, and it can be ported in spirit. Note its limit: it avoids reserved names by construction, so it tests only valid, non-colliding schemas. rustpb also needs the opposite generator, one that seeks collisions and cycles to exercise diagnostics (see question 3).

### Gaps
- No primary source found on proptest-arbitrary interop crates or on a maintained Rust counterpart to Hypothesis's database. The `proptest-arbitrary-interop` crate exists, but I did not verify its status.
- quickcheck's maintenance status is not stated anywhere I found. Only its 1.1.0 release is visible.
- No benchmark numbers for proptest generation and shrinking cost on deep protobuf-like trees.

## 2. Formal methods: Kani for varint/zigzag/length arithmetic; how prost uses Kani; Miri; Creusot/Prusti/Verus

### Takeaway
Kani fits rustpb's small, loop-bounded arithmetic: varint (at most 10 bytes), zigzag, length-prefix bounds checks, and the 64-bit-as-two-32-bit-halves path for JS. It is a cheap single CI job. But prost does *not* Kani-check its varint code; it Kani-checks only Duration/Timestamp in prost-types. So rustpb must write its own proofs, not inherit "prost's proofs". Miri matters little for rustpb's own safe code, but is cheap insurance for the pinned prost primitives and `bytes`. Creusot, Prusti and Verus are not realistic for a codec of this size today.

### Cited Findings
- prost's Kani usage at `894f4a7`:
  - harnesses exist only in `prost-types/src/duration.rs` (`check_duration_std_roundtrip`, `..._nanos`, `check_duration_chrono_roundtrip`) and `prost-types/src/timestamp.rs` (with `#[kani::unwind(3)]`), under `#[cfg(kani)]`;
  - CI job `kani` runs `model-checking/kani-github-action@v1.1` with `-p prost-types --features chrono`;
  - `KANI.md` recommends the GitHub action "because of instability in Kani internals", and documents `--concrete-playback` to turn a counterexample into a `cargo test`;
  - `prost/src/encoding/varint.rs` has hand-written `check(value, encoded)` unit tests plus `varint_overflow` tests, and no Kani harness.

  Sources: `https://github.com/tokio-rs/prost/blob/HEAD/KANI.md`, `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/ci.yml` L178–186, `https://github.com/tokio-rs/prost/blob/HEAD/prost-types/src/duration.rs` L206+, `https://github.com/tokio-rs/prost/blob/HEAD/prost/src/encoding/varint.rs` L181–280
- prost fuzzing: OSS-Fuzz CIFuzz on every PR with `fuzz-seconds: 300`, uploading crash artifacts. libFuzzer targets are `proto2`, `proto3`, `parse_date` and `parse_duration`, where the proto3 target runs `roundtrip::<TestAllTypesProto3>`. There is also an AFL target — `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/cifuzz.yml`, `https://github.com/tokio-rs/prost/blob/HEAD/fuzz/fuzzers/proto3.rs`, `https://github.com/tokio-rs/prost/blob/HEAD/FUZZING.md`
- Kani versions:
  - latest is kani-0.68.0 (16 Sep, marked Latest): explicit `contract::` symbols, `#[kani::loop_decreases]`, and autoharness for generic functions, run in parallel by default;
  - 0.60.0 introduced `autoharness`; 0.63.0 added experimental quantifiers and began stabilizing `--jobs` and `list`; 0.64–0.66 added loop contracts (modifies, `while let`, `for`).

  The page omits years. The ordering suggests 0.66.0 = Nov 2025, 0.67.0 = Jan 2026, 0.68.0 = Sep 2026 (inferred) — [Kani releases](https://github.com/model-checking/kani/releases)
- A 2026 paper ("Kani: A Model Checker for Rust", arXiv 2607.01504, ASE 2026 Industry Showcase) reports Kani running in production CI. The Rust standard-library verification campaign verifies over 16,000 harnesses per code change — [arXiv 2607.01504](https://arxiv.org/abs/2607.01504); [ASE 2026 listing](https://conf.researchr.org/details/ase-2026/ase-2026-industry-showcase/37/Kani-A-Model-Checker-for-Rust)
- Kani cannot verify unbounded loops: it needs an unwind bound "one more than" the iteration count, and reports an unwinding-assertion failure if the bound is too low. Proofs over buffers hold only up to the chosen size ("no errors on input slices of size 10, but no higher") — [Kani book: loop unwinding](https://model-checking.github.io/kani/tutorial-loop-unwinding.html)
- Kani's limitations chapter splits limits into inherent ones (the technique, undecidability) and fixable ones (unsupported features), with sub-pages on UB, feature support and overrides — [Kani book: limitations](https://model-checking.github.io/kani/limitations.html)
- bolero harnesses can run under Kani, so one harness serves fuzzing and bounded proof — [Kani blog](https://model-checking.github.io/kani-verifier-blog/2022/10/27/using-kani-with-the-bolero-property-testing-framework.html)
- Miri detects:
  - out-of-bounds access and use-after-free;
  - uninitialized reads;
  - misalignment;
  - invalid values;
  - data races;
  - leaks;
  - experimental Stacked/Tree Borrows aliasing violations.

  Its limits:
  - it is slow ("The default of 64 different seeds can be quite slow" with `-Zmiri-many-seeds`);
  - it has no FFI or networking;
  - it explores one execution per run;
  - it "fundamentally cannot ensure that your code is sound".

  `cargo miri nextest run` gives per-test processes but recompiles per invocation and runs no doctests — [Miri README](https://github.com/rust-lang/miri)
- buffa runs Miri in CI only on the modules with `unsafe`: `size_cache`, `copy_into_spare` and the pre-sized sink, the table interpreter (`miri test -p buffa --lib -- 'table::'`), and `OwnedView`. It caches the Miri sysroot — `https://github.com/anthropics/buffa/blob/HEAD/.github/workflows/ci.yml` L311–350
- Deductive verifiers: Prusti (on Viper), Creusot (to Why3, with prophecies for `&mut`) and Verus (adds linear ghost permissions; used in "several significant systems verification efforts" per a PLDI 2026 paper). I found no 2026 head-to-head comparison and no verified Rust protobuf or parser codec — [VerusBelt, PLDI 2026](https://iris-project.org/pdfs/2026-pldi-verusbelt.pdf); [Prusti project paper](https://www.paperswithcode.com/paper/the-prusti-project-formal-verification-for); [Verus project goals](https://github.com/verus-lang/verus/blob/main/source/docs/project-goals.md)

### Inferences
- Concrete Kani targets for rustpb, all with small fixed loop bounds:
  - `decode_varint(encode_varint(x)) == x` for all `u64`;
  - `encoded_len_varint(x) == encode_varint(x).len()`;
  - zigzag round-trip and monotone size for `i32`/`i64`;
  - `decode_varint` on any ≤11-byte buffer never panics and rejects >10-byte or overflowing input;
  - the length-prefix check (§4.6 "never reserve from a declared length") never reads past its parent for any `(declared, remaining)` pair;
  - the two-32-bit-halves 64-bit arithmetic equals the native `u64` arithmetic. This proves the JS path's math *as Rust*, not the emitted JS.

  These are exhaustive over the integer domain, which proptest cannot be. Whole-message encode/decode is a poor Kani target: unbounded `Vec`s, `HashMap` and allocation-heavy code blow up the solver. Leave it to proptest and fuzzing.
- Because design.md §3 dec. 1 pins prost's arithmetic, the proofs should target `rustpb::wire`'s functions, which wrap prost. They then prove what rustpb actually calls, whatever prost's internals do.
- CI cost: prost runs Kani as one job on a few harnesses. A handful of integer harnesses should take minutes at most (inference; no timing found). Kani's own docs recommend pinning the GitHub action because of internal instability, so pin the version.
- Miri: rustpb has no `unsafe` (§3 dec. 5), so Miri's value is in exercising dependencies (prost's string merge uses `unsafe` per design §3 dec. 1; `bytes`). Running a small codec test subset nightly is cheap insurance. It is not a per-PR gate.
- Creusot, Prusti and Verus would need specifications for the codec, the merge semantics and the TypeMapper contract. No codec precedent exists and the tools change quickly. Not realistic for a v1 harness. A small Kani set plus differential testing gives more assurance per hour.

### Gaps
- No published wall-clock numbers for prost's Kani job or for Kani on varint-sized harnesses.
- The years of Kani release dates are not shown on the releases page; the year mapping above is inferred from ordering.
- I did not find whether any Rust protobuf crate Kani-checks its varint code. buffa has no Kani in its workspace manifests or CI that I found.

## 3. Testing the code generator: snapshots, compile tests, compile-fail/trybuild, API guarantees, determinism, never-panic, build.rs integration

### Takeaway
The Rust protobuf generators combine four layers:
1. unit tests that call the generator on in-memory `FileDescriptorProto`s and match on `Result` errors (buffa);
2. golden expected-output files (prost-build's `_expected_*.rs`; insta or expect-test are the off-the-shelf form);
3. ignored-by-default "generate, then `cargo check`" matrix tests over a fixture set (buffa);
4. regenerate-and-`git diff --exit-code` checks for checked-in bootstrap code (ScalaPB, and rustpb's own descriptor types).

What none of them do, and rustpb should add, is property-fuzz the generator with random and adversarial descriptor sets, asserting no panic, determinism, and "compiles, or refused with a diagnostic".

### Cited Findings
- **buffa codegen unit tests** build `DescriptorProto`/`FileDescriptorProto` values in Rust, call `generate(&[file], &[...], &CodeGenConfig::default())`, and assert on the error variant. Example: `naming.rs` checks absent or empty names → `CodeGenError::MissingField("message.name")` across four shapes. The tests module has 30 files (naming, keyword-escaped fields, map types, size arithmetic, feature gating, …), and most assertions are `contains(...)` string checks on output (163 in `generation.rs`) — `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/src/tests/naming.rs`, `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/src/tests/`
- **buffa compile matrix**: `feature_gating_compile.rs` runs `cargo check` on generated code for each `{json, views, text}` feature subset over 25 fixture protos (editions, oneofs, extensions, maps, groups, MessageSet, cross-package refs). Its doc says this catches "a `cfg`-gated item referenced from an ungated context … a cross-feature dependency" that string assertions cannot. It is `#[ignore]` so `cargo test --workspace` stays fast; CI runs it with `-- --ignored`; it reuses the workspace `target/` so `buffa` isn't rebuilt 8 times — `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/tests/feature_gating_compile.rs`; CI steps "Feature-gating compile matrix", "Shared descriptor pool compile" in `https://github.com/anthropics/buffa/blob/HEAD/.github/workflows/ci.yml` L100–109
- **prost-build golden files**: fixtures under `prost-build/src/fixtures/` (smoke_test, helloworld, field_attributes, packed_encoding, …). The smoke test compares generated output to `_expected_smoke_test.rs` / `_expected_smoke_test_formatted.rs`. Identifier sanitizing (`sanitize_identifier`, `to_snake`, `strip_enum_prefix`) has its own unit tests, including raw-keyword cases — `https://github.com/tokio-rs/prost/blob/HEAD/prost-build/src/lib.rs` L390–455, `https://github.com/tokio-rs/prost/blob/HEAD/prost-build/src/ident.rs`
- **rust-protobuf test crates**: `protobuf-codegen-identical-test` (pure-Rust parser vs protoc codegen output), `protobuf-codegen-pure-test`, `protobuf-codegen-protoc-test`, `protobuf-parse-error-test` with `test-data`, `interop`, `protobuf-fuzz` — `https://github.com/stepancheg/rust-protobuf/blob/HEAD/test-crates/`
- **ScalaPB checked-in generated code**: `test_generated_code_checked_in.sh` regenerates the plugin protos and fails on `git diff --exit-code` — `https://github.com/scalapb/ScalaPB/blob/HEAD/test_generated_code_checked_in.sh`
- **insta**:
  - `glob!` runs a closure per matching input file and sets a per-input snapshot suffix (needs the `glob` feature);
  - `INSTA_UPDATE=auto` (default) writes `.snap.new` locally and behaves like `no` in CI;
  - `INSTA_FORCE_PASS=1` collects all snapshots in one run;
  - `cargo insta test --review` is the recommended loop;
  - `--unreferenced=auto` deletes stale snapshots locally and rejects them in CI.

  Sources: [insta advanced](https://insta.rs/docs/advanced/); [cargo-insta CLI](https://insta.rs/docs/cli/)
- **expect-test 1.5.1**: inline `expect![...]` and `expect_file!`. `UPDATE_EXPECT=1` rewrites the literal in source using recorded `file!/line!/column!`. Its docs call insta "a more feature full snapshot testing library" — [expect-test docs](https://docs.rs/expect-test/latest/expect_test/)
- **trybuild (1.0.120)** compile-fail tests compare rustc stderr to `.stderr` snapshots. A compile_fail test that compiles is a failure. `TRYBUILD=overwrite` rewrites snapshots; new ones go to `wip/`. trybuild keeps old normalizations so old snapshots still pass. Drift across rustc versions is the main pitfall, and projects handle it by:
  - pinning one rustc version (darling);
  - keeping `.stderr.beta`/nightly variants (bitflags, num_enum);
  - running only on latest stable (zbus);
  - pinning a `rust-toolchain.toml` that includes `rust-src`.

  Sources: [docs.rs trybuild](https://docs.rs/crate/trybuild/latest); [trybuild normalize.rs](https://docs.rs/crate/trybuild/1.0.62/source/src/normalize.rs); [darling compiletests](https://android.kernel.org/toolchain/rustc/+/87880447cc5437c45a3562596a9766d9797e7318/vendor/darling/tests/compiletests.rs); [bitflags compile.rs](https://fuchsia.googlesource.com/fuchsia/+/HEAD/third_party/rust_crates/vendor/bitflags-1.3.2/tests/compile.rs); [zbus MR 187](https://gitlab.freedesktop.org/dbus/zbus/-/merge_requests/187)
- `static_assertions` 1.1.0 `assert_not_impl_any!(T: Trait, …)` fails compilation if the type implements any listed trait, a compile-time negative check with no stderr snapshot — [static_assertions docs](https://docs.rs/static_assertions/latest/static_assertions/macro.assert_not_impl_any.html)
- ScalaPB's random-schema generator (question 1) compiles random valid schemas and is tagged `Slow` with `minSuccessful(20)`, so the suite is small but runs every time — `https://github.com/scalapb/ScalaPB/blob/HEAD/proptest/src/test/scala/GeneratedCodeSpec.scala`

### Inferences
- Layer plan for rustpb-build:
  1. **Diagnostics as data**: unit tests build descriptor sets in Rust and match the error enum, as buffa does. One test per §3 dec. 7 collision: oneof vs nested type, `Error`/`Empty`/`Unrecognized` variants, `gen`/`self` keywords, a message named `Option`. One per default-construction refusal (cycle named).
  2. **Golden output** with insta + `glob!` over a `fixtures/*.proto` directory, one `.snap` per fixture. Snapshots are rustfmt'd output, so reviews read as code. CI must never auto-accept, which is insta's default under `CI`. expect-test suits small inline cases.
  3. **Compile matrix**: generate all fixtures into a temp crate and `cargo check` it, plus `clippy -D warnings` and rustdoc. Mark it ignored by default and run it in CI, reusing the workspace target as buffa does.
  4. **API guarantees**: "a non-defaultable message has no `impl Default`" is best a generated `static_assertions::assert_not_impl_any!(Parent: Default)` per non-defaultable fixture, plus the generated `try_default() == Ok(default())` test the design already specifies. This avoids trybuild stderr drift. Keep trybuild for the few cases where the *error message* is the contract (e.g. a derive misuse), pinned to one toolchain as darling does.
  5. **Bootstrap types**: regenerate the checked-in `descriptor.proto`/`rustpb.proto` types and `git diff --exit-code`, as ScalaPB does (already design §3 dec. 6).
- **Never-panic and determinism by fuzzing the generator.** Use proptest/arbitrary over `FileDescriptorSet`, from two generators:
  - a *valid-by-construction* one (as ScalaPB's), where the property is "generates; output compiles (sampled, since compiling is slow); decode/encode round-trips";
  - an *adversarial* one, biased toward collisions (keywords, names equal after snake/camel casing, `Option`/`Error`/`Empty`, deep nesting, default-edge cycles), where the property is "never panics (`catch_unwind`), and returns `Ok` or a diagnostic". Some properties need no compile at all: same input twice gives byte-identical output; the output does not depend on file order; it does not depend on `HashMap` iteration (run with different seeds).

  A cheap libFuzzer target can also feed raw bytes to the descriptor decoder plus generator, since rustpb-build decodes descriptors with its own runtime (design §3 dec. 6).
- **build.rs integration**: an integration test crate with a real `build.rs`. Assert that `cargo build -vv` output contains `cargo:rerun-if-changed=` for every input `.proto` and for imports. Then touch an imported proto and assert a rebuild via cargo's fingerprint "Dirty" lines in `-v` output. I found no tool that does this; it is a hand-written integration test (inference).

### Gaps
- No public Rust protobuf generator found that property-fuzzes its *generator* with random descriptor sets. The precedent is ScalaPB's (Scala) and buffa's raw descriptor unit tests.
- No source found on best practice for testing `rerun-if-changed` emission. The approach above is inferred.
- I did not read connect-rust's codegen tests in detail. Its workspace manifests and CI do not mention insta, expect-test, trybuild or proptest.

## 4. Model-based testing and oracles for the default-construction graph and fixed-point analyses

### Takeaway
The defaultability fixed point and the cycle refusal are pure functions over a small graph, an ideal fit for a property test against a *naive reference model*: brute-force recursion with memoization or an explicit DFS. Generate random message graphs with edge kinds {no_box, required, mapped-base, optional, oneof, repeated} and `default = false` flags, and assert that the generator's analysis equals the model's. ScalaPB remains the semantic oracle for the hand fixtures (design §4.1).

### Cited Findings
- proptest-state-machine supports a reference model vs SUT with invariants checked after every transition and shrinking of transition sequences — [proptest book: state machine](https://proptest-rs.github.io/proptest/proptest/state-machine.html)
- ScalaPB's `GraphGen.scala` generates random multi-file message and enum graphs with a namespace state (unique names, nesting, cross-file IDs). `GeneratedCodeSpec` checks structural invariants over generated graphs ("min and max id are consecutive over files") before compiling them — `https://github.com/scalapb/ScalaPB/blob/HEAD/proptest/src/test/scala/GraphGen.scala`, `GeneratedCodeSpec.scala` L19–33
- The design specifies the rules: a message is non-defaultable if it declares `default = false`, or a singular default edge (unmapped `no_box`/`required` child, or a mapped field's base message) reaches a non-defaultable message. A cycle made only of singular default edges is refused, naming the cycle. Optional, oneof and repeated edges break cycles — `https://github.com/rust-js-lang/rustpb/blob/HEAD/docs/design.md` §4.1 "Default construction composes through the message graph"

### Inferences
- Oracle design for `defaultability`:
  - **Model**: a recursive function `defaultable(m) = !declares_false(m) && all(defaultable(t) for singular default edges m→t)`, with explicit "on stack" detection returning `Cycle(path)`. Write it as simply as possible in test code; it is the specification.
  - **Properties**:
    - analysis == model for every node;
    - a node is refused-as-cycle iff it lies on a cycle of only singular-default edges (check against Tarjan's SCC computed independently, or brute force on graphs ≤ 8 nodes);
    - *monotonicity*: adding `default = false` anywhere never makes any message defaultable, and turning any singular edge into optional never makes a message non-defaultable;
    - *locality*: a change to a message unreachable via singular default edges changes nothing;
    - the diagnostic's named cycle is a real cycle in the input;
    - for every defaultable message the generated `impl Default` exists; for every non-defaultable one it does not (end-to-end on a sample, compile-checked).
  - **Exhaustive small-scope**: enumerate *all* graphs up to 3–4 messages × edge kinds. The space is small enough to run in seconds, and this "small scope hypothesis" style finds the corner cases a random sampler skips (inference; no source gathered).
- For runtime behavior of the graph (decode of absent fields, error paths `Grandparent.parent.child.field`), the generated message types are the SUT. The model predicts which decode inputs must fail and with which path. A proptest over "random subset of fields present" against that prediction covers design §4.1 rule 4 beyond its seven hand fixtures.
- Ordering properties: "required error reported before default error" (§4.1 rule 4a) is a model rule, so state it as a property over random messages with both a missing required field and a missing non-defaultable child.
- The ScalaPB differential corpus (design §6) is the outer oracle. The model above checks rustpb-build is internally consistent with its own spec; only ScalaPB checks the spec matches ScalaPB.

### Gaps
- No published Rust example found of model-based testing of a code generator's fixed-point analysis. The plan above is inference built from general techniques.
- No source gathered for small-scope exhaustive testing in Rust (e.g. a bounded-exhaustive enumerator crate). Kani could also be used for bounded graphs, but its cost on such code is unknown.

## 5. Coverage: cargo-llvm-cov, branch coverage, MC/DC status (2026), how trusted projects set and report targets

### Takeaway
Line and region coverage via `cargo-llvm-cov` is stable and standard. Branch coverage still requires nightly (`--branch`). rustc removed MC/DC entirely in August 2025 (PR #144999, 1.91 milestone), and as of 2026 re-implementation is only a project-goal proposal. So rustpb cannot claim MC/DC. It should treat coverage as a gap-finder, with mutation testing (cargo-mutants) as the stronger signal of test quality.

### Cited Findings
- rustc PR #144999 "coverage: Remove all unstable support for MC/DC instrumentation" removed the `-Zcoverage-options=mcdc` support added in 2024. It was merged 2025-08-08 (1.91.0 milestone). The author called the code a "major maintenance burden" and wrote "I made the wrong call in accepting the current implementation" — [rust-lang/rust#144999](https://github.com/rust-lang/rust/pull/144999)
- A 2026 Rust project goal proposes re-implementing decision and MC/DC coverage in rustc, possibly via Stable MIR, with a commercial contributor committing to maintenance. It was at proposal stage in the sources found — [Rust project goals 2026: MC/DC coverage support](https://rust-lang.github.io/rust-project-goals/2026/mcdc-coverage-support.html); [proposal notes](https://hackmd.io/JIntvBk_Ri63Sq2Tn63LQA)
- cargo-llvm-cov's README marks `--branch` as unstable and nightly-only (issue #8). Thresholds use flags such as `--fail-under-branches`; the threshold example is from a third-party guide — [cargo-llvm-cov README](https://github.com/taiki-e/cargo-llvm-cov); [changelog](https://docs.rs/crate/cargo-llvm-cov/latest/source/CHANGELOG.md); [third-party usage guide](https://playbooks.com/skills/laurigates/claude-plugins/cargo-llvm-cov)
- cargo-mutants (docs.rs latest 27.1.0):
  - `--in-diff FILE` tests only mutants overlapping changed lines, giving fast PR feedback that "can miss some problems that would be found by running mutants on the whole codebase";
  - 26.2.0 made an empty mutant set not fail, for in-diff CI use;
  - one project runs `cargo mutants --in-diff origin/main..HEAD` as a non-blocking PR job.

  Sources: [mutants.rs in-diff](https://mutants.rs/in-diff.html); [cargo-mutants NEWS 26.2.0](https://docs.rs/crate/cargo-mutants/latest/source/NEWS.md); [docs.rs cargo-mutants](https://docs.rs/cargo-mutants/); [example project's mutation-testing page](https://sites.ecmwf.int/docs/tensogram/main/dev/mutation-testing.html)
- None of prost's, buffa's or connect-rust's CI workflows (local clones) runs a coverage job or enforces a coverage threshold. A grep of all three repos' `.github/workflows/*.yml` for `llvm-cov`, `tarpaulin`, `codecov`, `coverage` and `nextest` matched only buffa's unrelated "publish coverage" script, so none uses nextest either — `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/ci.yml`, `https://github.com/anthropics/buffa/blob/HEAD/.github/workflows/ci.yml`, `https://github.com/connectrpc/connect-rust/blob/HEAD/.github/workflows/`

### Inferences
- Report region and line coverage per crate (runtime, derive, build) with `cargo llvm-cov nextest`, and add nightly branch coverage on a scheduled job. Gate only on "no uncovered *new* error-path region in `rustpb::wire` and the decoder" via review, not a global percentage, since no trusted protobuf crate in the sample publishes a threshold.
- Mutation testing is a better trust signal for a codec than a coverage number, and design.md §6 item 5 already calls for it. Run cargo-mutants `--in-diff` per PR (non-blocking at first), plus a full scheduled run over `rustpb` and `rustpb-derive`. Note a pitfall: mutants in *generated* code are not visited by cargo-mutants, which mutates source files. So the derive and build crates' templates are the mutation surface, plus hand-written mutations of generated output (design's "swap a tag, drop zigzag").
- Coverage of generated code: generated `.rs` files in `OUT_DIR` are instrumented like any code, but they are a build artifact. Map them back to fixtures when reporting (inference; not verified with cargo-llvm-cov's path filters).

### Gaps
- I found no authoritative source on how trusted Rust projects (rustls, wasmtime) set coverage targets. No local clones of those exist, and I did not search further within the tool budget.
- The exact status, in November 2026, of the MC/DC project goal (accepted? any nightly flag?) is not confirmed.

## 6. API stability checks: cargo-semver-checks, cargo-public-api, MSRV

### Takeaway
Use cargo-semver-checks on `rustpb` (runtime), `rustpb-json` and `rustpb-build`'s public API in release CI, and a cargo-public-api snapshot test of the runtime so any API change shows in review. The generated code's API is a second public surface that neither tool sees directly. Cover it by running semver-checks on a fixture crate of generated code between rustpb versions (inference). MSRV: declare `rust-version` and run a CI job that regenerates the lockfile and `cargo check`s on the MSRV toolchain, as buffa does.

### Cited Findings
- cargo-semver-checks had 242 lints at the end of 2025, up from 120 at the end of 2024; its 2025 releases were v0.39–v0.45. A 2026 project-goal page says 245, and v0.47.0 added 7 opt-in lints. The maintainer warns the lint count can become a vanity metric — [cargo-semver-checks 2025 year in review](https://predr.ag/blog/cargo-semver-checks-2025-year-in-review/); [Rust project goals 2026: cargo-semver-checks](https://rust-lang.github.io/rust-project-goals/2026/cargo-semver-checks.html); [0.47.0 release](https://newreleases.io/project/cargo/cargo-semver-checks/release/0.47.0)
- cargo-public-api lists a crate's public API from rustdoc JSON and diffs it between versions or git refs. Its test workflow uses the `public-api`, `rustdoc-json` and `rustup-toolchain` crates with `assert_eq_or_update("./tests/snapshots/public-api.txt")`, updated by `UPDATE_SNAPSHOTS=yes cargo test`. It needs a nightly (0.52.x needs nightly-2025-11-22 or later), installed automatically and not necessarily active — [cargo-public-api README](https://github.com/cargo-public-api/cargo-public-api)
- buffa's MSRV job declares workspace `rust-version = "1.75"`. CI regenerates an MSRV-compatible lockfile (`rm -f Cargo.lock && cargo generate-lockfile`), then runs `cargo check --workspace --all-targets --locked` on 1.75, plus a separate test on Rust 1.77 for the table codec. It also checks no_std on a bare-metal ARM target and a 32-bit (`i686`) workspace check — `https://github.com/anthropics/buffa/blob/HEAD/Cargo.toml` L37, `https://github.com/anthropics/buffa/blob/HEAD/.github/workflows/ci.yml` L238–275, L374–405
- prost's CI has a `semver-checks` job using `obi1kenobi/cargo-semver-checks-action@v2` — `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/ci.yml` L215–220
- prost's CI runs `cargo minimal-versions --no-private check --all-features` (cargo-hack + cargo-minimal-versions) to test lower dependency bounds — `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/ci.yml` L170–176

### Inferences
- The derive macro and generated code are the API most users touch: field names, `Option` vs `Box`, the presence of `Default`, the `try_default` signature, enum `Unrecognized`. A "generated API fixture crate" built from a fixed schema set gives cargo-semver-checks (or a cargo-public-api snapshot) something to diff across rustpb releases. It catches, e.g., a message silently losing `impl Default`. I believe cargo-semver-checks has lints for removed trait impls, including derived ones, but I did not verify the lint name (see Gaps).
- A 32-bit `cargo check` (as buffa does) is relevant to rustpb because rust-js lowers to a JS target where `usize` differs. It catches `usize`/`u64` length arithmetic assumptions early.
- Add `cargo minimal-versions` as prost does, since rustpb pins prost exactly (§3 dec. 1) and must not silently need newer dependency versions.

### Gaps
- I did not verify which cargo-semver-checks lints cover removal of a (derived) trait impl, or whether it handles proc-macro-generated items in a dependent crate's output.
- I did not check Cargo's MSRV-aware resolver behavior (`rust-version`-aware resolution) against primary docs.

## 7. Test-speed engineering: cargo-nextest, partitioning, fast inner loop vs heavy nightly jobs

### Takeaway
Split the harness into tiers by cost:
- **per edit**: unit tests, proptest at default 256 cases, golden snapshots, generator diagnostics;
- **per PR**: the compile matrix (ignored by default, as buffa does), Kani, a time-boxed fuzz run (prost's 300 s CIFuzz), cargo-mutants `--in-diff`;
- **nightly/scheduled**: high-`PROPTEST_CASES`, long fuzzing, random-schema generate+compile, full mutation run, Miri, branch coverage, the ScalaPB differential corpus.

Run tests with cargo-nextest, with hash partitioning across CI shards and archived builds.

### Cited Findings
- nextest partitioning:
  - `slice:m/n` assigns round-robin after filtering;
  - `hash:m/n` assigns deterministically by binary ID and test name, so adding tests doesn't move others, though small runs can be uneven;
  - `count:` is deprecated in favor of `slice:`.

  A build can be archived once and reused by partition jobs (GitLab example: `parallel: 3` with `--partition slice:${CI_NODE_INDEX}/${CI_NODE_TOTAL}`) — [nextest partitioning](https://nexte.st/docs/ci-features/partitioning/)
- `cargo miri nextest run` runs one Miri process per test, in parallel, but recompiles per invocation and skips doctests — [Miri README](https://github.com/rust-lang/miri)
- buffa keeps the default `cargo test --workspace` fast by marking compile-matrix tests `#[ignore]`, run explicitly in CI, and by sharing the workspace `target/`. Conformance runs locally "slow, ~10–20 min" via a separate task. Fuzzing is a separate nightly-only workspace (`fuzz/` excluded from the workspace) — `https://github.com/anthropics/buffa/blob/HEAD/buffa-codegen/tests/feature_gating_compile.rs` (module doc), `https://github.com/anthropics/buffa/blob/HEAD/Taskfile.yml` L73, `https://github.com/anthropics/buffa/blob/HEAD/fuzz/Cargo.toml` L10
- ScalaPB tags its random-schema property `Slow` with `minSuccessful(20)` and `workers(1)` — `https://github.com/scalapb/ScalaPB/blob/HEAD/proptest/src/test/scala/GeneratedCodeSpec.scala` L35–36
- prost time-boxes PR fuzzing to `fuzz-seconds: 300` through OSS-Fuzz CIFuzz — `https://github.com/tokio-rs/prost/blob/HEAD/.github/workflows/cifuzz.yml`
- proptest's `PROPTEST_CASES`, `PROPTEST_MAX_SHRINK_TIME` and `PROPTEST_TIMEOUT` env vars let one test suite run at different intensities per tier without code changes — [proptest Config](https://docs.rs/proptest/latest/proptest/test_runner/struct.Config.html)

### Inferences
- Fix `PROPTEST_RNG_SEED` per CI run and print it (or derive it from the commit hash), so a nightly failure is reproducible locally. Committed `proptest-regressions/` files then pin found failures forever. Also copy shrunk inputs into named hand fixtures, since regression seeds break when strategies change (proptest book).
- Compiling generated code dominates generator-test cost. Batch many fixtures into *one* generated crate per compile (buffa reuses one target dir), and sample only a few random schemas per PR for compile checks; check never-panic and determinism on thousands without compiling.
- Keep Kani in its own job: it is independent of the test build and needs its own toolchain via the pinned action.

### Gaps
- No measured timings were gathered for nextest vs `cargo test` on a protobuf-sized workspace, or for Kani per harness.
- I did not read nextest's "why process-per-test" page; the trade-off (isolation vs process start cost) is not sourced here.
