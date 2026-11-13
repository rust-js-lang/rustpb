# Rust protobuf failure modes and testing practices (prior art for RustPB)

Research date: 2026-11-13. Local shallow clones read: prost at 894f4a7 (2026-11-12, v0.14.6), rust-protobuf at 0118c79 (2025-09-21, master = 4.0.0-alpha.0), quick-protobuf at 54e7d6c (2024-02-14, v0.8.1), all under `~/Documents/GitHub/`. "Local:" sources below are files in those clones. GitHub issue facts were read with `gh issue view`/`gh search` (issue text, not source). The GitHub search API rate-limited partway through, so quick-protobuf and some rust-protobuf issue searches were not done (see Gaps).

## Security advisories (RustSec / CVE / GHSA) for Rust protobuf crates

### Takeaway
Every Rust protobuf advisory on record is a denial-of-service from untrusted input (RUSTSEC-2020-0002 also notes possible memory corruption on targets without stack probes). Two shapes cover all of them: (1) recursion with no bound, which overflows the stack (prost 2020, rust-protobuf 2024/2025, through unknown groups), and (2) allocation sized by a length prefix the attacker controls (rust-protobuf 2019). A third, smaller class is panics on arithmetic overflow in well-known-type conversions (prost-types 2021). RustPB needs a recursion budget on every recursive path, including unknown-field and group skipping. It must never reserve memory from a length prefix before checking it against the bytes that remain. Its WKT conversions must be fallible.

### Cited Findings
- **RUSTSEC-2020-0002 / CVE-2020-35858 / GHSA-gv73-9mwv-fwgq (prost)**: "decoding untrusted input could overflow the stack". On architectures without stack probes (e.g. ARM), the advisory says the overflow "is unsound and can result in potential memory corruption (or even RCE)". GHSA titles it "Out of bounds write in prost". Fixed in **prost 0.6.1**; published 2020-01-16; upstream issue tokio-rs/prost#267. — [OSV RUSTSEC-2020-0002](https://api.osv.dev/v1/vulns/RUSTSEC-2020-0002); [RustSec](https://rustsec.org/advisories/RUSTSEC-2020-0002.html); [issue #267](https://github.com/tokio-rs/prost/issues/267)
- The prost fix was a decode recursion limit (issue "Add a recursion limit for decoding", tokio-rs/prost#176, closed 2019-04-09). An afl fuzzing issue (#270) was closed the same day as #267 (2020-01-16). — [prost #176](https://github.com/tokio-rs/prost/issues/176); [prost #270](https://github.com/tokio-rs/prost/issues/270)
- Current prost: `const RECURSION_LIMIT: u32 = 100;` in `prost/src/lib.rs`, carried in `DecodeContext.recurse_count`. Code comments say it "cannot be" changed at runtime (it can only be turned off with the `no-recursion-limit` feature, per issue #736). — Local: `prost/prost/src/lib.rs:30`, `prost/prost/src/encoding.rs:41-86`; [prost #736](https://github.com/tokio-rs/prost/issues/736)
- **RUSTSEC-2019-0003 / CVE-2019-15544 / GHSA-mh6h-f25p-98f8 (protobuf / rust-protobuf)**: "Affected versions of this crate called Vec::reserve() on user-supplied input", which allows out-of-memory through `stream::read_raw_bytes_into()`. Affected: <1.7.5 and 2.0.0 to <2.6.0. Fixed in **1.7.5 and 2.6.0**. Published 2019-06-08; issue stepancheg/rust-protobuf#411 asked for the OOM fix commit 66a22c8 to be backported into 2.6.0. — [OSV RUSTSEC-2019-0003](https://api.osv.dev/v1/vulns/RUSTSEC-2019-0003); [rust-protobuf #411](https://github.com/stepancheg/rust-protobuf/issues/411)
- How rust-protobuf mitigates this today: `READ_RAW_BYTES_MAX_ALLOC: usize = 10_000_000`. If a declared length is at or above that, it reserves only the cap and grows as bytes actually arrive. The same rule caps `reserve` for packed repeated fields (`len_bytes <= READ_RAW_BYTES_MAX_ALLOC` otherwise `READ_RAW_BYTES_MAX_ALLOC / ENCODED_SIZE`). — Local: `rust-protobuf/protobuf/src/coded_input_stream/mod.rs:54,386-411`, `buf_read_iter.rs:211-218,312-315`
- **RUSTSEC-2024-0437 / CVE-2025-53605 / GHSA-2gh3-rmm4-6rq5 (protobuf / rust-protobuf)**: "did not properly parse unknown fields when parsing a user-supplied input", which lets an attacker cause a stack overflow. Fixed in **protobuf 3.7.2** (all earlier versions affected). RustSec publish date 2024-12-12; the rustsec.org listing date is 2025-03-07. Upstream issue #749 ("Security Issue", opened 2024-12-09) says the reporter had tried to reach the maintainer privately "over the last months (August, October)". The changelog entry is "Do not stack overflow on groups" (PR #756), so the bug was recursion through unknown **groups** (wire type 3/4), which bypassed the recursion limit. — [OSV RUSTSEC-2024-0437](https://api.osv.dev/v1/vulns/RUSTSEC-2024-0437); [rust-protobuf #749](https://github.com/stepancheg/rust-protobuf/issues/749); Local: `rust-protobuf/CHANGELOG.md:7`; [rustsec.org protobuf package page](https://rustsec.org/packages/protobuf.html)
- rust-protobuf today: `DEFAULT_RECURSION_LIMIT: u32 = 100`, which can be changed at runtime with `CodedInputStream::set_recursion_limit`. — Local: `rust-protobuf/protobuf/src/coded_input_stream/mod.rs:51,106-112`
- **RUSTSEC-2021-0073 / CVE-2021-38192 / GHSA-x4qm-mcjq-v2gf (prost-types)**: converting a `prost_types::Timestamp` to `SystemTime` "can cause an overflow and panic" on untrusted input. Fixed in **prost-types 0.8.0** by switching the infallible `From<Timestamp> for SystemTime` to `TryFrom`. Published 2021-07-08; issue #438. — [OSV RUSTSEC-2021-0073](https://api.osv.dev/v1/vulns/RUSTSEC-2021-0073)
- OSV has **no advisories** for `quick-protobuf`, `protobuf-codegen`, `protobuf-parse`, `pbjson`, or `protobuf-json-mapping` (queried 2026-11-13). — [OSV query API](https://api.osv.dev/v1/query) (queried by package name, crates.io ecosystem)
- Advisories for libraries around protobuf that are not protobuf itself: a deps.rs page flags `quick-protobuf-codec 0.2.0` because of its `bytes` dependency (RUSTSEC-2026-0007 in `bytes`). I did not verify this further. — [deps.rs quick-protobuf-codec](https://deps.rs/crate/quick-protobuf-codec/0.2.0)
- Soundness bug outside the advisory DB: prost #193 "Unsound use of String::as_mut_vec" (2019). Merging string wire data that is not valid UTF-8 broke the `String` invariant in place. Closed 2019-06-11. — [prost #193](https://github.com/tokio-rs/prost/issues/193)

### Inferences
- The 2024 rust-protobuf bug and prost's open #924 (below) show that recursion limits tend to be attached to "decode a known sub-message" and miss the other recursive paths: skipping unknown groups, default construction of recursive required fields, Debug/Clone/Drop/encode of deep values. RustPB should list every recursive function in its runtime and in generated code, and give each one either a budget or a proof that it terminates.
- When the stack overflows in JS, it throws `RangeError` rather than aborting the process. A runtime that catches errors and keeps running may then continue with partially merged state, so a depth limit matters as much for the JS target as for native.
- quick-protobuf: `grep` over `quick-protobuf/src/reader.rs` finds no recursion-depth or limit field. I did not check whether its generated code bounds depth, so treat "quick-protobuf has no recursion limit" as unverified but likely. Its `Reader::from_reader` does `Vec::with_capacity(capacity); unsafe { buf.set_len(capacity) }` and then `read_exact` into memory that was never initialized (behind `#[allow(clippy::uninit_vec)]`). In Rust this is a soundness problem, not just a style issue. — Local: `quick-protobuf/quick-protobuf/src/reader.rs:645-653`
- quick-protobuf's Changelog has "added overflow checking in `read_unknown()`", so the length arithmetic in its unknown-field skip path could overflow before that fix. — Local: `quick-protobuf/Changelog.md:22`

### Gaps
- I could not get exact dates or versions for quick-protobuf's `read_unknown()` overflow fix (changelog line only).
- I did not search the issue tracker for advisories or bugs in Google's official Rust protobuf (`protobuf` v4 crate on upb/C++ kernels). The C/C++ kernels have their own upstream CVE history, which I did not cover.
- RustSec has two dates for RUSTSEC-2024-0437: OSV "published 2024-12-12" and the rustsec.org listing "March 7, 2025". The CVE ID is from 2025.

## Decoding correctness bug classes (issue trackers)

### Takeaway
Besides DoS, the main correctness gaps in Rust protobuf crates are: unknown fields not preserved (prost has had this open since 2017), wrong packed defaults in proto3 (both prost and rust-protobuf), map fields accepting the wrong wire type (fixed in prost 0.14.6), UTF-8 handled differently on encode and decode, recursion bugs in required/recursive proto2 messages, and float sign/presence (-0.0). The upstream wire-format rules every decoder must follow are: last value wins for scalars, singular sub-messages are merged, packed and unpacked are both accepted, fields can arrive in any order, and varints are at most 10 bytes.

### Cited Findings
- Upstream wire rules (protobuf.dev, "Encoding"):
  - For scalars and strings, "the parser accepts the last value it sees". Singular embedded messages are **merged**; repeated fields are concatenated.
  - "Protobuf parsers must be able to parse repeated fields that were compiled as packed as if they were not packed" (and the reverse). Several packed records for one field must be concatenated.
  - "Protobuf parsers must be able to parse fields in any order." Serialization order is an implementation detail.
  - Varints take "anywhere between one and ten bytes". Negative `intN` uses two's complement, so "all ten bytes must be used" (a negative int32 is 10 bytes on the wire).
  - "Map fields are just a shorthand for a special kind of repeated field". Map order is not guaranteed.
  - Groups (wire types 3/4) are "a deprecated feature that should not be used, but they remain in the wire format". A mismatched end-group field number makes the message malformed.
  - "Oneof fields are encoded the same as if the fields were not in a oneof."
  - "Protos must be smaller than 2 GiB when serialized."
  — [protobuf.dev Encoding guide](https://protobuf.dev/programming-guides/encoding/)
- **Unknown fields not preserved (prost)**: tokio-rs/prost#2 "Add support for unknown fields", open since 2017-06-26 and still open. prost's conformance failure list today is exactly the two tests caused by this: `Required.Proto2.ProtobufInput.UnknownVarint.ProtobufOutput` and `Required.Proto3.ProtobufInput.UnknownVarint.ProtobufOutput`, under the comment "TODO(tokio-rs/prost#2): prost doesn't preserve unknown fields." — [prost #2](https://github.com/tokio-rs/prost/issues/2); Local: `prost/conformance/failing_tests.txt`
- **Map wire type not validated (prost)**: the 0.14.6 changelog has "Reject map fields with a non length-delimited wire type (#1473)" (late 2026). Before that, a map field arriving with a varint or fixed wire type was not rejected. — Local: `prost/CHANGELOG.md:1-7`
- **proto3 packed default wrong**: prost #976 "`proto3` doesn't pack repeated fields by default". `options.packed()` returns false unless `[packed=true]` is set, so the proto3 default was ignored. Opened 2024-01-31, closed 2026-07-03. A related fix in 0.14.5: "Correct packed encoding default for proto3 fields with options (#1399)". rust-protobuf had the same class of bug: "Default to packed for repeated primitives in proto3" (PR #707). — [prost #976](https://github.com/tokio-rs/prost/issues/976); Local: `prost/CHANGELOG.md:16`, `rust-protobuf/CHANGELOG.md:26`
- **Recursive proto2 `required` message → stack overflow on empty input (prost)**: #924 (open since 2023-09-21). `message Foo { required Foo x = 1; }` generates `Box<Foo>`, and `Foo::decode("")` overflows the stack, because default construction recurses forever. — [prost #924](https://github.com/tokio-rs/prost/issues/924)
- **Other recursive functions without a bound**: prost #652 "I found two recursive functions which may cause stack overflow?" (open, 2022-05-21). — [prost #652](https://github.com/tokio-rs/prost/issues/652)
- **UTF-8 asymmetry**: prost #342 "Strings that are not valid UTF-8 can be encoded but error on decoding" (closed 2020-06-15): "encoding should raise an error if decoding will." See also #193 (unsound in-place merge of invalid UTF-8). — [prost #342](https://github.com/tokio-rs/prost/issues/342); [prost #193](https://github.com/tokio-rs/prost/issues/193)
- **Derive panics on non-ASCII**: prost #1417, "`prost::Message` derive panics on non-ASCII invalid `map` type strings" (closed 2026-03-17). The 0.14.5 changelog also has "Avoid UTF-8 panic for enumeration names (#1425)" and "Prevent panic for service generator in empty module (#1442)". These are panics in codegen and derive, not in the runtime. — [prost #1417](https://github.com/tokio-rs/prost/issues/1417); Local: `prost/CHANGELOG.md:15-17`
- **Negative zero / presence**: prost #1177 "Negative zero float is encoded equal to zero float (i.e. NULL)" (open, 2024-10-23). Comparing `-0.0 == 0.0` to decide "is this the default" drops the field, so -0.0 does not round-trip in proto3 implicit presence. — [prost #1177](https://github.com/tokio-rs/prost/issues/1177)
- **Group support history**: prost #16 "Support for `groups`" (closed 2017-07-04). The rust-protobuf fix for CVE-2025-53605 was about groups too (above). — [prost #16](https://github.com/tokio-rs/prost/issues/16)
- **Unknown enum values**: prost generates `TryFrom<i32>` that returns `UnknownEnumValue` (carrying the raw integer) instead of `DecodeError`. Open enums are stored as `i32` in message fields. — Local: `prost/CHANGELOG.md:475`; [prost #276](https://github.com/tokio-rs/prost/issues/276) ("A more ergonomic enumeration representation", open)
- **Error-returning instead of panicking on broken I/O**: rust-protobuf #714 "CodedOutputStream panics on broken pipe" (closed 2024-02-21). — [rust-protobuf #714](https://github.com/stepancheg/rust-protobuf/issues/714)
- **WKT contract breaks**: rust-protobuf #690 "`From` implementations for some WKTs break their contracts" (open, 2023-11-23). This is the same class as prost-types' Timestamp panic. — [rust-protobuf #690](https://github.com/stepancheg/rust-protobuf/issues/690)

### Inferences
- Conformance coverage for the edge cases listed in the brief: the upstream binary conformance tests cover many of them (packed/unpacked, last value wins, merging sub-messages, overlong varints, unknown fields, groups, oneof merge, map entries with a missing key or value). prost passing everything except unknown-field round-trip suggests that tests for the remaining cases come from upstream, not from prost's own unit tests. I did not open the upstream test list to confirm each item (see Gaps).
- In proto3 implicit presence, "is default?" must compare **bit patterns** for floats (so -0.0 is emitted), not `==`. ScalaPB on the JVM compares with `!= 0.0`, which also treats -0.0 as default. Byte compatibility with ScalaPB may therefore mean matching ScalaPB's choice rather than upstream C++'s. This needs checking against the ScalaPB generator.
- Map entries: since maps are "repeated entry messages", a decoder must accept entries with a missing key or value (default them), duplicate keys (last wins), and entry fields in any order. prost's #1473 shows that even the outer wire type was not checked until 2026.

### Gaps
- I did not find specific Rust issues (search was rate-limited) for: tag 0 / field numbers > 2^29-1, varints longer than 10 bytes, NaN payload preservation, unknown-field ordering on re-encode, or oneof "last member wins" in Rust crates. The upstream conformance suite is the best source for these; I did not enumerate its test names.
- The protobuf.dev Encoding page fetch gave no explicit statement on maximum field number or tag 0.

## Encoding correctness: determinism, field order, size computation, proto3 optional

### Takeaway
Upstream says outright that default serialization "is not deterministic". prost has an open request (#965) to guarantee field order and map order. prost's fuzz harness checks that `encoded_len()` equals the bytes actually written, and that re-encoding the decoded result gives identical bytes. There is still no recursion limit on encode (#736), so a message can be encoded that the same stack then refuses to decode.

### Cited Findings
- "the default serialization is not deterministic"; the byte output should not be assumed stable. — [protobuf.dev Encoding guide](https://protobuf.dev/programming-guides/encoding/)
- prost #965 "feature: Deterministic Serialization" (open, 2023-12-21) asks for a semver guarantee on field order (by tag) and map order when using the `BTreeMap` option. — [prost #965](https://github.com/tokio-rs/prost/issues/965)
- prost #736 "Recursion limit during encode" (open, 2022-10-26): "it is possible to encode a message that cannot be decoded anymore". Context: the Apache Arrow DataFusion project hit it. — [prost #736](https://github.com/tokio-rs/prost/issues/736)
- prost #1112: `encoded_len` "gets computed twice when encoding a message" (open, 2024-07-30). This is a performance issue. The issue itself reports the top-level double count; separately, `message::encode` calls `encoded_len()` at every level, so nested sizes are recomputed per level. — [prost #1112](https://github.com/tokio-rs/prost/issues/1112)
- prost's `tests::roundtrip` (used by its fuzzers):
  1. Decode the input.
  2. Compute `encoded_len()`, encode, and fail if `encoded_len != buf1.len()`.
  3. Decode the output again, re-encode, and fail with "roundtripped encoded buffers do not match" if the bytes differ.
  An assertion that `encoded_len <= data.len()` is commented out. — Local: `prost/tests/src/lib.rs:156-214`
- proto3 `optional` presence has a dedicated test: `proto3_presence.proto/.rs`. — Local: `prost/tests/src/proto3_presence.rs`

### Inferences
- Disabling `encoded_len <= input len` suggests that a canonical re-encoding can be longer than the input (e.g. a packed field received unpacked and re-emitted, or a field repeated in the input). Size can therefore only be checked against the actual output, not the input.
- For byte compatibility with ScalaPB, RustPB should adopt ScalaPB's actual output order as its contract: fields by declaration or number order, unknown fields appended at the end, and map order whatever ScalaPB emits. It should test this with golden bytes produced by ScalaPB rather than relying on any upstream determinism guarantee.

### Gaps
- I did not compare whether prost emits fields in tag order or declaration order, or where rust-protobuf places unknown fields on re-encode.

## Testing practices: conformance, fuzzing, differential, formal

### Takeaway
The strongest test pattern in the Rust ecosystem combines four things: (1) the upstream conformance runner with an exact-match failure list, (2) round-trip fuzzing on `TestAllTypesProto2/3` with an `encoded_len` check, (3) continuous fuzzing on OSS-Fuzz run on every PR through CIFuzz, and (4) interop tests against the C++ implementation. prost runs conformance for **binary only**: JSON and text-format requests are reported as Skipped. Buf's conformance results for JS libraries showed how much is missed without the suite: protobufjs was about 31% compliant at launch (zero failures today).

### Cited Findings
- **Conformance protocol**: the runner (`conformance_test_runner`) sends a `ConformanceRequest` to a "testee" program and expects a `ConformanceResponse`. `--failure_list <file>` must match **exactly**: a listed test that passes is also an error. Lists use wildcards (a trie over dot-separated names). Upstream keeps per-language lists, including `failure_list_rust_upb.txt`. — [conformance_test_runner.cc (mirror)](https://flutter.googlesource.com/third_party/protobuf/+/refs/heads/36.x/conformance/conformance_test_runner.cc); [failure_list_cpp.txt (mirror)](https://flutter.googlesource.com/third_party/protobuf/+/b4262af656bd3971cd2b2c9e51797982a8e9d91c/conformance/failure_list_cpp.txt); search-snippet level only, pages not opened.
- **prost conformance**:
  - Builds upstream protobuf **v25.8** in a build script (`PROTOBUF_VERSION = "25.8"`).
  - Its testee returns `Skipped("JSON output is not supported")`, `Skipped("TEXT_FORMAT output is not supported")` and `Skipped("JSON input is not supported")`.
  - Its failure list has 2 entries, both unknown-field round-trip.
  — Local: `prost/protobuf/build.rs:7-9`, `prost/conformance/src/main.rs:58-90`, `prost/conformance/failing_tests.txt`
- **prost fuzzing**:
  - libFuzzer targets `proto2`, `proto3` (round-trip `TestAllTypesProto3` via `tests::roundtrip`), `parse_date` and `parse_duration` (PRs #1127, #1129), plus afl targets.
  - CI workflow `cifuzz.yml` uses `google/oss-fuzz/infra/cifuzz` with `oss-fuzz-project-name: 'prost'` and `fuzz-seconds: 300` on every pull request, and uploads crash artifacts. This shows prost is an OSS-Fuzz project.
  - FUZZING.md documents afl (`cargo afl fuzz`) and a reproduce binary; its libFuzzer section says "TODO".
  — Local: `prost/fuzz/fuzzers/*.rs`, `prost/.github/workflows/cifuzz.yml`, `prost/FUZZING.md`, `prost/CHANGELOG.md:421-426`
- **prost formal verification**: the repo has `KANI.md`, which describes Kani model-checking harnesses (`cargo kani`) "proving the absence of certain classes of bugs like unwrap exceptions, overflows, and assertion failures". It recommends running them through the GitHub action. — Local: `prost/KANI.md`
- **prost unit tests for decoding edge cases**: `decode_error.rs`, `groups.rs`, `recursive_oneof.rs`, `message_encoding.rs`, `unittest.rs`, and varint split-buffer tests ("Add test for split buffer varint decoding (#1321)", "test `decode_varint_slice` with roundtrips (#1325)"). — Local: `prost/tests/src/`, `prost/CHANGELOG.md:154-156`
- **prost no_std and editions**: separate `tests-no-std`, `tests-2015/2018/2024` crates check generated code across Rust editions. — Local: `prost/` top-level listing
- **rust-protobuf**:
  - cargo-fuzz targets `all`, `empty_message(_read)`, `map(_read)`, `repeated(_read)`, `singular(_read)`.
  - A C++ interop crate (`test-crates/interop`, compiled in CI by `interop/cxx/compile.sh` on every matrix job).
  - `protobuf-codegen-identical-test`, which checks that the pure-Rust parser and protoc produce identical codegen.
  - `protobuf-parse-error-test`.
  - I found no conformance failure list in the repo.
  — Local: `rust-protobuf/test-crates/`, `rust-protobuf/test-crates/protobuf-fuzz/fuzz/fuzz_targets/`, `rust-protobuf/.github/workflows/ci.yml:37-170`
- **quick-protobuf**: tests are `write_read.rs`, `rust_proto.rs`, `packed_primitives`, and a `rust_protobuf` comparison directory. I found no fuzz or conformance directories in the clone. — Local: `quick-protobuf/quick-protobuf/tests/`
- **Buf's JS conformance survey (2023-03-21)**:
  - Protobuf-ES passes 100% of required and 99.8% of recommended tests (later quoted as 99.95%; its one failure was proto2 extensions).
  - protobufjs was then about 31% compliant (Buf's README now shows zero failures for it).
  - The new Duration tests caught a Protobuf-ES JSON bug where negative durations lost their sign ("-0.1s" became "0.1s").
  - The suite has over 2,000 tests.
  — [Buf blog: Protobuf-ES conformance](https://buf.build/blog/protobuf-conformance)

### Inferences
- RustPB has to be JSON-compatible with ScalaPB, and JSON is the half prost skips entirely. RustPB should therefore run the **full** upstream suite (binary + ProtoJSON, both `Required.*` and `Recommended.*`) natively **and** as compiled JS, with exact failure lists for each target. The JS run is where u64/BigInt, float formatting and Duration/Timestamp sign bugs will appear (the Protobuf-ES Duration sign bug is a direct precedent).
- Differential testing against ScalaPB itself is more useful to RustPB than against C++, because the contract is "byte- and JSON-equal to the ScalaPB deployment". A ScalaPB-based conformance testee exists upstream for the JVM (ScalaPB is cloned locally at `~/Documents/GitHub/ScalaPB`; not inspected here). It can generate golden corpora and serve as a differential oracle on fuzzer-generated messages.
- The prost round-trip oracle (len == written; decode∘encode∘decode stable) is cheap and catches size-computation drift. It should be the first fuzz target. Recursion-limit, length-prefix-allocation and group-skip inputs should be checked into the seed corpus as regression cases for each advisory above.
- Kani-style bounded proofs fit the varint/zigzag/length arithmetic well: small pure functions where "no overflow, no panic" can be proved exhaustively.

### Gaps
- I did not open protobuf.dev's conformance documentation or upstream `conformance/README.md` (the instruction not to read source via GitHub web applied, and no local upstream protobuf clone exists). The exact upstream guidance "for new implementations" (e.g. `--enforce_recommended`, `--maximum_edition`) is unverified here.
- I could not confirm whether `rust-protobuf` or Google's v4 Rust crate is on OSS-Fuzz (only prost is confirmed, via its workflow).
- I found no published conformance pass rates for rust-protobuf or quick-protobuf.

## WebAssembly / JavaScript target lessons (64-bit ints, unknown fields, size)

### Takeaway
The leading JS implementation (Protobuf-ES) uses `bigint` for 64-bit fields by default and offers `[jstype = JS_STRING]` to opt into strings, falling back to strings where BigInt is unavailable. protobufjs needs the separate `long` package for exact 64-bit values and otherwise loses precision above 2^53. Its unknown-field handling has changed between releases (preserved vs discarded by default), a cautionary example: a compatibility layer must preserve unknown fields by default and never change that silently.

### Cited Findings
- Protobuf-ES: `bigint` is the default for 64-bit scalars. `jstype = JS_STRING` generates `string` fields. Where BigInt is unavailable, 64-bit fields still round-trip without loss but hold strings. — search-result summary of the Protobuf-ES manual (`MANUAL.md`, seen only through a mirror) and [@bufbuild/protobuf proto-int64.d.ts](https://cdn.jsdelivr.net/npm/@bufbuild/protobuf@2.12.0/dist/cjs/proto-int64.d.ts) (the direct GitHub MANUAL.md fetch returned 404; treat as secondary).
- protobufjs:
  - For exact 64-bit support, install `long`. Otherwise 64-bit values fall back to JS numbers, with precision loss above 2^53.
  - The docs conflict across versions on unknown fields: some say they are kept by default and re-emitted (`message.$unknowns`). Later releases say fields "present on the wire are discarded by default" as "the safer default".
  - The setting is `Reader.discardUnknown`.
  — [protobufjs 8.8.0 README](https://cdn.jsdelivr.net/npm/protobufjs@8.8.0/README.md); [protobufjs 8.6.5 README](https://cdn.jsdelivr.net/npm/protobufjs@8.6.5/README.md) (search-snippet level; not opened)
- The ProtoJSON spec (relevant to JSON compatibility on JS): [protobuf.dev ProtoJSON format](https://protobuf.dev/programming-guides/json/) (not fetched; cited as the authority).
- prost has no wasm-specific issues in the search results. The closest is #555 "deno/typescript usage" (closed 2021-11-02). — [prost #555](https://github.com/tokio-rs/prost/issues/555)

### Inferences
- For a Rust→JS compiler where `u64`/`i64` become `BigInt`:
  - ProtoJSON writes int64/uint64 as decimal **strings**, so JSON is easy.
  - The hot path is varint decode/encode. Doing it in BigInt arithmetic is slow. The usual JS technique (protobuf-es, protobufjs) splits the value into two 32-bit halves (lo/hi numbers) and builds a BigInt only at the field boundary.
  - RustPB's varint routines should be written so that the rust-js compiler can lower them without per-byte BigInt operations. For example, a two-u32 decode path that is semantically identical to the u64 one, proved equal by fuzzing or Kani.
- The unknown-field example from protobufjs supports making preservation the default, and storing unknown fields as raw byte slices (cheap in JS as `Uint8Array` subarrays).
- Code size: protobufjs's reflection-based design and Protobuf-ES's generated descriptors plus shared runtime show the tradeoff. Per-message generated encode/decode is fast but large, while a table-driven runtime is small. I found no Rust-specific wasm size measurements (Gap).

### Gaps
- I found no reliable data on generated-code size or bundle size for prost/rust-protobuf in wasm, or on Protobuf-ES bundle-size claims.
- I did not verify protobuf-es's internal int64 split (lo/hi) from its source; that point is inferred from general knowledge and should be confirmed in a local clone.

## Compile time and generated code size for large schemas

### Takeaway
I found little hard evidence. The one concrete prost data point is a discussion asking for one module per `.proto`, because the single generated file pushed compile times over 25 s and slowed IDE indexing. The maintainer replied it was not supported but could become opt-in. prost-build also includes every transitively imported `.proto`.

### Cited Findings
- prost discussion #755: a user asks prost_build to emit one Rust module per `.proto` file because the single generated file makes compile times exceed 25 seconds and slows IDE indexing. The maintainer: "Prost doesn't support this though this could be something we add as an opt-in behavior." — [prost discussion #755](https://github.com/tokio-rs/prost/discussions/755) (search-snippet level)
- prost-build docs: transitively imported `.proto` files are included automatically. — [prost-build docs](https://docs.rs/prost-build/0.14.4/src/prost_build/lib.rs.html) (search-snippet level)
- prost #1021 "Consider placing `Config::format` behind a feature flag" (closed 2024-03-30). This is a build dependency cost (prettyplease) of formatting generated code. — [prost #1021](https://github.com/tokio-rs/prost/issues/1021)
- A non-Rust example of the same failure mode: a Perfetto schema produced about 532k lines of Zig, and the build was killed (suspected OOM). — [Ziggit thread](https://ziggit.dev/t/suggestions-for-huge-amount-of-generated-code/14558)

### Inferences
- Proc-macro derives (`#[derive(prost::Message)]`) move codegen cost into every downstream compile. Emitting plain Rust from the generator (as rust-protobuf does) avoids repeated macro expansion. For RustPB, generated plain code also gives the Rust→JS compiler readable input.
- One module per `.proto` file (or per package), with no single giant `include!`, helps incremental compilation and IDE indexing. It also lets the JS output be tree-shaken per file.

### Gaps
- I found no benchmarks of rustc time or binary size for thousands of messages in prost, rust-protobuf or Google's v4 crate. I found no reported remedies beyond splitting modules and feature-gating formatting.
- I did not research Google's official Rust protobuf (v4, upb kernel) compile-time or size characteristics.
