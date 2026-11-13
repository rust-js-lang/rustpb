# Protobuf-specific test oracles and suites for RustPB

Research date: 2026-11-13. Every local read names the clone and commit it came from.
Local clones used (all under `~/Documents/GitHub/`):

| Clone | Commit / date | Note |
|---|---|---|
| `protobuf` | `8e2636c`, 2026-11-13, `version.json` = protoc `38-dev` | sparse checkout (`conformance/`, `docs/`, `rust/`), shallow |
| `protobuf-conformance` (Buf) | `faec7c9`, 2026-07-27 | cloned for this note |
| `protobuf-es` | `76e269e`, 2026-11-09 | cloned for this note |
| `ScalaPB` | `d713ac9e`, 2026-03-16 (master = `v1.0.0-alpha.3-22`) | has branch `origin/0.11.x`; latest 0.11 tag seen `v0.11.9` |
| `scalapb-jsoniter` | `e7bf71f`, 2026-11-10; tags `v0.1.2`, `v0.3.0` | |
| `scalapb-json4s` | `b0a0aab`, 2026-11-01 | cloned for this note |
| `buffa` | `1047169`, 2026-10-20 | |
| `prost` | `894f4a7`, 2026-11-12 | |
| `prost-reflect` | `2d0e184`, 2026-07-09 | |

Citation form: a GitHub URL names the upstream file, but **every file was read in the local clone at the
commit above**, never on GitHub.

---

## Q1. Google's conformance suite: runner, protocol, categories, failure lists, counts, build, non-native testees

### Takeaway
The conformance suite is one C++ program, `conformance_test_runner`. It holds every test case and drives a
testee program over stdin/stdout. Each message on the pipe is a 4-byte little-endian length followed by a
serialized `ConformanceRequest` or `ConformanceResponse`. A run fails on any unexpected failure, on any test
that passes while it is listed as failing, and on a listed test whose failure message changed. **Skips are
silent and never fail a run.** A proto2 + proto3 run (`--maximum_edition PROTO3`) has **2,779 binary/JSON
tests** (1,476 of them proto3) and **434 text-format tests**. Adding editions gives **5,567** and **883**.
These counts are for runner v33.0.0. buffa, prost-reflect, protobuf-es (Node and Bun) and protobuf.js all run
it, and the JS projects get a prebuilt runner from the `protobuf-conformance` npm package.

### Cited Findings

**Protocol and testee contract**
- The runner forks the testee and talks over its stdin/stdout: "tester sends 4-byte length N (little endian)"
  plus N bytes of `ConformanceRequest`; testee replies with 4-byte length M plus M bytes of
  `ConformanceResponse` — [conformance_test_runner.cc header comment](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test_runner.cc)
- `ConformanceRequest` = `oneof payload { bytes protobuf_payload = 1; string json_payload = 2; string jspb_payload = 7; string text_payload = 8; }`,
  `WireFormat requested_output_format = 3`, `string message_type = 4`, `TestCategory test_category = 5`,
  `JspbEncodingConfig jspb_encoding_options = 6`, `bool print_unknown_fields = 9` — [conformance.proto](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance.proto)
- `ConformanceResponse.result` oneof: `parse_error = 1`, `serialize_error = 6`, `timeout_error = 9`,
  `runtime_error = 2`, `protobuf_payload = 3`, `json_payload = 4`, `skipped = 5`, `jspb_payload = 7`,
  `text_payload = 8`. The proto says a `parse_error` "does not necessarily mean the testee failed the test", and a `runtime_error` "will always indicate that the test failed" — [conformance.proto](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance.proto)
- The first request has `message_type == "conformance.FailureSet"`, and the testee must answer with a
  serialized (empty) `FailureSet` in `protobuf_payload` — [conformance.proto](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance.proto); also implemented this way in [ScalaPB ConformanceScala.scala](https://github.com/scalapb/ScalaPB/blob/master/conformance/src/main/scala/ConformanceScala.scala) and [protobuf-es conformance.ts](https://github.com/bufbuild/protobuf-es/blob/main/packages/protobuf-conformance/src/conformance.ts)
- The message types a testee may receive are `protobuf_test_messages.proto3.TestAllTypesProto3`,
  `...proto2.TestAllTypesProto2`, `...editions.proto2.TestAllTypesProto2`,
  `...editions.proto3.TestAllTypesProto3`, `...editions.TestAllTypesEdition2023`,
  `...editions.TestAllTypesEdition2026`, and `...edition_unstable.TestAllTypesEditionUnstable` (main branch) — [conformance.proto](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance.proto)
- Enums: `WireFormat {UNSPECIFIED, PROTOBUF, JSON, JSPB (inside Google only), TEXT_FORMAT}`;
  `TestCategory {UNSPECIFIED_TEST, BINARY_TEST, JSON_TEST, JSON_IGNORE_UNKNOWN_PARSING_TEST (optional
  feature), JSPB_TEST, TEXT_FORMAT_TEST}` — [conformance.proto](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance.proto)
- Crash and hang handling: if a read times out, hits EOF, or fails, the runner kills the testee. That test
  gets a `timeout_error` or `runtime_error`, and the next test **spawns a fresh testee**. The default read
  timeout is 30 s (`fork_pipe_runner.h: read_timeout = absl::Seconds(30)`) — [fork_pipe_runner.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/fork_pipe_runner.cc)

**How outputs are judged**
- Most tests parse the testee's output with the C++ reference implementation and compare it with
  `MessageDifferencer`. The field comparator sets `set_treat_nan_as_equal(true)`. So byte order, JSON key
  order and number spelling are **not** compared — [conformance_test.cc `VerifyResponse`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc)
- Some binary tests set `require_same_wire_format`. These require **byte-exact** output
  (`check = equivalent_wire_format == protobuf_payload`): `RunValidBinaryProtobufTest` cases such as
  `ValidDataScalarBinary`, `ValidDataOneofBinary`, and the `UnknownOrdering`-style round trips — [conformance_test.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc), [binary_json_conformance_suite.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/binary_json_conformance_suite.cc)
- Test names are `<Level>.<Syntax>.<InputFormat>.<TestName>.<OutputFormat>`. Level is
  `Required`/`Recommended`. Syntax is `Proto2`, `Proto3`, `Editions_Proto2`, `Editions_Proto3` or `Editions`.
  Input is `ProtobufInput`/`JsonInput`/`TextFormatInput`. Output is `ProtobufOutput`/`JsonOutput`/
  `TextFormatOutput`, and some tests end in `Validator` instead — [conformance_test.cc `GetTestName`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc)

**Failure lists (legacy `conformance_test_runner` CLI)**
- Flags: `--failure_list <file>`, `--text_format_failure_list <file>`, `--enforce_recommended`,
  `--maximum_edition <edition>`, `--output_dir <dir>`, `--test <name>` (repeatable; runs those tests in
  isolation), `--debug` (needs `--test`), `--performance`, `--verbose`, and `--record_requests <file>`
  (writes one line per request, `<test_name> <input_size> <crc32c_hex>`). The first non-flag argument is the
  testee program; **the arguments after it are passed to the testee** — [conformance_test_runner.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test_runner.cc)
- Failure-list format: one test name per line, with `#` starting a comment. Text after `#` on a test line
  is the **expected failure message**, and the actual (normalized) message must start with it. An empty
  message matches anything. Names can use `*` wildcards (Java's list uses `Recommended.*.JsonInput...`) — [conformance_test_runner.cc `ParseFailureList`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test_runner.cc), [conformance_test.cc `ReportFailure`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc), [failure_list_java.txt](https://github.com/protocolbuffers/protobuf/blob/main/conformance/failure_list_java.txt)
- A listed test that passes prints "is in the failure list, but test succeeded. Remove its match from the
  failure list" and fails the run. A listed test that fails with a different message is reported in
  `expected_failure_messages.txt` / `unexpected_failure_messages.txt`. A wildcard matching too many tests goes
  to `exceeded_max_matches.txt`. New failures go to `failing_tests.txt`. Each file is written to
  `--output_dir` as a ready-made patch — [conformance_test.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc)
- An unlisted `Recommended` failure is only a `WARNING` unless `--enforce_recommended` is set. An unlisted
  `Required` failure is an `ERROR` — [conformance_test.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc)
- **Skips never fail a run.** `ReportSkip` only records the test in `skipped_`, which is printed with
  `--verbose`. The summary line is `CONFORMANCE SUITE PASSED|FAILED: N successes, M skipped, K expected failures, U unexpected failures` — [conformance_test.cc `ReportSkip`, `RunSuite`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/conformance_test.cc)
- The runner runs two suites in each invocation: binary+JSON (`--failure_list`) and text format
  (`--text_format_failure_list`). buffa's script notes "The conformance_test_runner always runs two suites per invocation" — [buffa conformance/run-conformance.sh](https://github.com/anthropics/buffa/blob/main/conformance/run-conformance.sh)

**New on protobuf main (2026, not yet in a release as far as I could tell)**
- A gtest-based `test_environment` binary has flags `--testee_binary`, `--testee_args`, `--failure_list`
  (comma-separated), `--enforcement_level` (a priority, "0 enforces only P0, 1 enforces P0 and P1"),
  `--maximum_edition` (default `EDITION_PROTO3`), `--performance`, `--fix` (rewrites the failure list from
  the observed results; refused under `--gtest_filter`/sharding) and `--fix_output_file`. The file says these
  must never be linked into `conformance_test_runner`, "whose legacy command line parses flags of the same names itself" — [test_environment_flags.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/test_environment_flags.cc) (Copyright 2026)

**Test counts.** They are counted from Buf's "baseline" testees, which fail every test on purpose so the full
list is recorded. Runner `protobuf-conformance` npm **v33.0.0** per `package-lock.json`.
- `baseline_proto3` (`--maximum_edition PROTO3`): **2,779** binary+JSON tests, **434** text-format tests — [protobuf-conformance impl/baseline_proto3](https://github.com/bufbuild/protobuf-conformance/tree/main/impl/baseline_proto3)
  - Required.Proto3: 776 ProtobufInput + 357 JsonInput + 10 top-level (Duration/Timestamp proto-input range
    checks) = **1,143**
  - Recommended.Proto3: 226 ProtobufInput + 102 JsonInput + 5 top-level (FieldMask round trip, Value NaN/Inf) = **333**
  - Required.Proto2: 779 ProtobufInput + 206 JsonInput = **985**; Recommended.Proto2: 228 + 90 = **318**
  - Text format: Required.Proto3.TextFormatInput 384, Recommended.Proto3.TextFormatInput 24,
    Recommended.Proto3.ProtobufInput (text output) 8, Required.Proto2.TextFormatInput 18
- `baseline_edition2023` and `baseline_edition2024`: **5,567** binary+JSON, **883** text. Each adds
  Editions_Proto2 and Editions_Proto3 mirrors plus 14 `Required.Editions.ProtobufInput` tests. The two
  baselines have the same counts — same source
- Test families in `binary_json_conformance_suite.cc`: `PrematureEof*` (12 shapes per type),
  `ValidDataScalar[Binary]`, `ValidDataRepeated{,.PackedInput,.UnpackedInput}`, `RepeatedScalarSelectsLast`,
  `RepeatedScalarMessageMerge`, `ValidDataMap`, `ValidDataOneof[Binary]`, `MergeOneofMessage`,
  `IllegalZeroFieldNum_Case_0..3`, `BadTag_FieldNumberTooHigh`, `BadTag_FieldNumberSlightlyTooHigh`,
  `BadTag_OverlongVarint`, `BadTag_VarintMoreThanTenBytes`, `BadLength_Varint32BitOverflow`,
  `UnmatchedStartGroup*`/`UnmatchedEndGroup*`/`MismatchedGroupTags`, `UnknownWireType{6,7}_Field{0..3}_Version{0..3}`,
  `RejectInvalidUtf8.String.{Singular,Repeated,Oneof,MapKey,MapValue}` (RECOMMENDED), `UnknownVarint`,
  `UnknownOrdering`, `EnforceDepthLimit.{Map,MapStringKey,MessageSetExtension}` (RECOMMENDED; the comment says
  "The default recursion limit is 100 for most languages"), and many JSON families (number spellings,
  NaN/Infinity, wrappers, WKTs, field-name conventions, `json_name`, null handling, trailing commas,
  `*DeepNesting25/200`) — [binary_json_conformance_suite.cc](https://github.com/protocolbuffers/protobuf/blob/main/conformance/binary_json_conformance_suite.cc)

**Building or obtaining the runner**
- CMake: `cmake . -Dprotobuf_BUILD_CONFORMANCE=ON && cmake --build .` produces `conformance_test_runner`.
  Bazel: `bazel test //src:conformance_test` (C++), and per-language `conformance_test` targets. CMake builds
  only the runner, not other languages' testees. The README still says "the test runner currently does not work on Windows", although main has a `fork_pipe_runner_win32.cc` — [conformance/README.md](https://github.com/protocolbuffers/protobuf/blob/main/conformance/README.md)
- **Prebuilt binary:** npm package `protobuf-conformance`, latest **36.2.0**. Install it with
  `npm install --save-dev protobuf-conformance`, then run `npx conformance_test_runner` and
  `npx conformance_proto_eject <dir>`; the second copies the test `.proto` files. The page calls it "an unofficial distribution of the test runner". It is published from `timostamm/protobuf-npm`, and its sole listed collaborator is timostamm — [npm protobuf-conformance](https://www.npmjs.com/package/protobuf-conformance). Buf's README says that starting with protobuf v32.0 it stopped publishing runner releases and points to this npm package — [protobuf-conformance README](https://github.com/bufbuild/protobuf-conformance/blob/main/README.md)
- buffa builds a **statically linked runner** in Docker (Alpine/musl, `cmake g++ ...`). It pins
  `PROTOBUF_TAG=v33.5` by **commit** `b6f9284d...`, "not the tags: a tag can move without a review of this file", and pins jsoncpp the same way, built static, because the conformance CMake expects the `jsoncpp_lib` target — [buffa conformance/Dockerfile.tools](https://github.com/anthropics/buffa/blob/main/conformance/Dockerfile.tools)
- prost's `protobuf` helper crate builds protobuf **v25.8** with CMake in `build.rs`
  (`protobuf_BUILD_CONFORMANCE` ON except on Windows) and copies the runner out — [prost protobuf/build.rs](https://github.com/tokio-rs/prost/blob/master/protobuf/build.rs)
- prost-reflect does the same from a git **submodule** of protobuf (commit `e915ce24`), using the `cmake`
  crate in `build.rs` — [prost-reflect prost-reflect-conformance-tests/build.rs](https://github.com/andrewhickman/prost-reflect/blob/main/prost-reflect-conformance-tests/build.rs)
- ScalaPB downloads protobuf **3.19.6** and builds the runner with Bazel 3.7.2. CI caches the binary keyed on
  the script's hash — [ScalaPB conformance/build_test_runner.sh](https://github.com/scalapb/ScalaPB/blob/master/conformance/build_test_runner.sh), [.github/workflows/test.yml](https://github.com/scalapb/ScalaPB/blob/master/.github/workflows/test.yml)

**How Rust projects wire their testees**
- **buffa:** one testee binary, run under **seven modes** selected by environment variable:
  `std`, `nostd`, `BUFFA_VIA_VIEW=1`, `BUFFA_VIA_LAZY=1`, `BUFFA_VIEW_JSON=1`, `BUFFA_VIA_REFLECT=1`,
  `BUFFA_VIA_VTABLE=1`, and a table-codec build. Each mode has its own failure list
  (`known_failures_{,nostd,view,lazy,view_json,reflect,view_vtable,table,text}.txt`), and modes that don't
  cover an I/O pair answer `Skipped`. Flags: `--maximum_edition 2024`, **no** `--enforce_recommended` — [buffa conformance/run-conformance.sh](https://github.com/anthropics/buffa/blob/main/conformance/run-conformance.sh)
- buffa parses `ConformanceRequest` and builds `ConformanceResponse` **by hand** (`envelope.rs`): "we must not use buffa (or a generated codec) to parse the test harness messages themselves", which avoids circular trust. The loop is `read_exact` of 4 bytes, `u32::from_le_bytes`, `read_exact` of the body, `handle`, write the length and body, then `flush`. EOF on the length read ends the loop cleanly — [buffa conformance/src/envelope.rs, src/main.rs](https://github.com/anthropics/buffa/blob/main/conformance/src/main.rs)
- buffa's main `known_failures.txt` contains only comments, so it has zero listed failures. Its README: "buffa passes the protobuf binary and JSON conformance test suite (v33.5, editions up to 2024). Both `std` and `no_std` builds pass the full suite including JSON" — [buffa README.md](https://github.com/anthropics/buffa/blob/main/README.md)
- **prost-reflect:** a Cargo integration test (`tests/conformance.rs`) runs
  `conformance_test_runner --enforce_recommended --failure_list failure_list.txt --text_format_failure_list text_format_failure_list.txt <bin>`.
  It locates the testee binary next to `current_exe()`, and **both failure lists are empty**. It builds the
  test messages with `prost_build` using `.btree_map(["."])` "so that encode/decode roundtrips can use encoded output for comparison" — [prost-reflect conformance tests](https://github.com/andrewhickman/prost-reflect/tree/main/prost-reflect-conformance-tests)
- **prost:** the same integration-test pattern (`CARGO_BIN_EXE_conformance`, `--enforce_recommended`). It
  **skips all JSON** (returns `Skipped` for JSON input or output), and `failing_tests.txt` lists
  `Required.Proto{2,3}.ProtobufInput.UnknownVarint.ProtobufOutput` because "prost doesn't preserve unknown fields" — [prost conformance/](https://github.com/tokio-rs/prost/tree/master/conformance)

**Non-native (JavaScript) testees**
- **Node, protobuf-es:** the testee is `bin/conformance.js`, starting `#!/usr/bin/env node`. The command is
  `conformance_test_runner --maximum_edition MAX --enforce_recommended --failure_list failing_tests_with_bigint.txt --text_format_failure_list failing_tests_text_format.txt bin/conformance.js`.
  A second run sets `BUF_BIGINT_DISABLE=1` to cover the string fallback for 64-bit integers, with its own
  list `failing_tests_without_bigint.txt` — [protobuf-es packages/protobuf-conformance/package.json, README.md](https://github.com/bufbuild/protobuf-es/tree/main/packages/protobuf-conformance)
- protobuf-es: the BigInt failure list is empty. The no-BigInt list holds only
  `Required.Proto{2,3}.JsonInput.{Int64,Uint64}Field{Max,Min}ValueNotQuoted.{Protobuf,Json}Output`, the cases
  where unquoted JSON numbers lose precision without BigInt — [protobuf-es failing_tests_without_bigint.txt](https://github.com/bufbuild/protobuf-es/blob/main/packages/protobuf-conformance/failing_tests_without_bigint.txt)
- **Bun, protobuf-es:** the runner execs the TypeScript file directly, `src/conformance.ts`, whose shebang is
  `#!/usr/bin/env -S bun run`. It uses the same flags with **empty failure lists**. A Deno variant also
  exists (`deno/conformance`) — [protobuf-es bun/conformance](https://github.com/bufbuild/protobuf-es/tree/main/bun/conformance)
- Node framing: protobuf-es reads stdin as an async stream (`readMessages(process.stdin, ...)`). Buf's
  protobuf.js and baseline testees use synchronous `readSync`/`writeSync` on fds 0 and 1, run through
  `#!/usr/bin/env -S npx tsx` — [protobuf-es conformance.ts](https://github.com/bufbuild/protobuf-es/blob/main/packages/protobuf-conformance/src/conformance.ts), [protobuf-conformance impl/protobuf.js/runner.ts](https://github.com/bufbuild/protobuf-conformance/blob/main/impl/protobuf.js/runner.ts)
- Buf's comparison table: Protobuf-ES 0 required and 0 recommended failures (edition 2024). **protobuf.js 0/0
  (edition 2024)**, so the 1,847 figure in older mirrors is out of date. protobuf-ts 6/7, google-protobuf
  1169/389, ts-proto 751/613, protoscript 1151/541, protoc-gen-ts 1690/475. "The Required and Recommended columns cover Binary and ProtoJSON. Text Format conformance is tracked separately." — [protobuf-conformance README](https://github.com/bufbuild/protobuf-conformance/blob/main/README.md) (HEAD 2026-07-27)
- Buf's blog said Protobuf-ES passed 100% of required and 99.8% of recommended tests at the time — [Buf blog: Protobuf-ES is the only fully-compliant Protobuf library for JavaScript](https://buf.build/blog/protobuf-conformance) (vendor claim; its README now shows 0/0)

**Current upstream failure lists, for scale:** C++ 78 lines, Java 82, Python 65, Rust (upb kernel) 11 lines;
the text lists for both Rust kernels are empty — `wc -l` in the local [protobuf/conformance](https://github.com/protocolbuffers/protobuf/tree/main/conformance)

### Inferences
- **Predicted deliberate failures for rustpb.** Each one should be listed in the failure list with its
  expected message, so the deviation is documented and caught if it changes:
  1. `Required.Proto3.ProtobufInput.UnknownOrdering.ProtobufOutput`. The test requires that unknown fields
     with one field number but **different wire types keep their wire order** (LEN "abc", VARINT 123, LEN
     "def", VARINT 456). ScalaPB's `UnknownFieldSet` groups values by wire type (design §4.6). protobuf-java
     makes the same choice and lists this test as `Required.*.ProtobufInput.UnknownOrdering.ProtobufOutput # Unknown field mismatch`.
     rustpb, matching ScalaPB, will fail it the same way. This is the clearest "conformance vs ScalaPB"
     conflict and should be decided explicitly (design §4.6 already chooses ScalaPB) — [binary_json_conformance_suite.cc `TestUnknownOrdering`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/binary_json_conformance_suite.cc), [failure_list_java.txt](https://github.com/protocolbuffers/protobuf/blob/main/conformance/failure_list_java.txt)
  2. `Required.Proto3.JsonInput.OriginalProtoFieldName.{JsonOutput,ProtobufOutput}` (REQUIRED). It feeds
     `"field_name2"`-style keys, which scalapb-jsoniter 0.1.2 skips silently (design §5), so the output is
     "not equivalent".
  3. `FieldNameDuplicate*` (RECOMMENDED) and `OneofFieldDuplicate{,2}` (REQUIRED) use
     `RunValidJsonTestOrParseFailure`, so either a parse error or the expected value passes. rustpb passes if
     0.1.2's last-non-null-wins gives the expected value. Otherwise the test goes in the list.
  4. Strictness tests that 0.1.2 may accept, such as quoted bools, leading zeros, trailing commas and
     comments: unknown until measured. Run the runner once against the **ScalaPB+jsoniter-0.1.2 testee**
     (Q3) and **copy that failure list** as rustpb's expected list. Then the deviations are measured, not
     guessed.
- Run with `--maximum_edition PROTO3 --enforce_recommended`. rustpb is proto3-only (design §2), so the
  testee answers `skipped` for `TestAllTypesProto2`. **Skips never fail a run, so the harness must assert the
  skip set too.** Parse the `--verbose` output, or count requests on the testee side, and require exactly the
  1,476 proto3 binary+JSON tests to be answered (counts per runner version). Without that check, a testee that
  skips everything "passes".
- Text-format tests: rustpb has no text format, so answer `skipped`. They don't count, and need no failure list.
- Use the **same testee source twice**: native, and through rust-js under Bun (`#!/usr/bin/env -S bun run`
  or a `bun` wrapper script). Give each its own failure list, as buffa does per mode. Run the JS testee a
  second time with BigInt disabled only if rustpb's JS output ever has a non-BigInt path. protobuf-es's
  no-BigInt failures show exactly which tests catch 64-bit precision loss.
- Parse the envelope by hand, as buffa does, so a rustpb bug can't hide inside the harness. Or decode it
  with prost, a separate implementation.
- Pin the runner by version (npm `protobuf-conformance@36.2.0`, or a commit-pinned CMake build as buffa
  does), and record its test count in the repo, so a runner upgrade shows up as a reviewed count change.
  `--record_requests` lists exactly which requests ran; diffing it pins the corpus across upgrades.
- Wire `--output_dir` into CI artifacts: the runner writes ready-made `failing_tests.txt` and
  `succeeding_tests.txt`-style patches.

### Gaps
- I did not run the runner, so per-category counts for v36.x and main (v38-dev) are not measured. The
  counts above are for v33.0.0, from Buf's baseline lists.
- I could not find when `UnknownOrdering` was added; the local clone is shallow. It is not in ScalaPB's
  3.19.6-era failure list, which suggests it arrived later. That is an inference.
- Whether the new `test_environment`/`--enforcement_level`/`--fix` interface ships in a tagged release:
  not verified.
- The platforms the npm runner package ships binaries for are not stated on the npm page.

---

## Q2. Upstream test messages and golden files: what each covers, which are useful fixtures

### Takeaway
For a proto3-only implementation, the useful fixtures are `test_messages_proto3.proto` (the conformance
message: every scalar, repeated, packed/unpacked, map key/value combination, oneof, recursion and WKT) and
the proto3 unittest files. The classic binary golden files (`golden_message`,
`golden_packed_fields_message`) survive only under `objectivec/Tests/`. C++ now builds its golden bytes at
test time. Several unittest files on main have moved to `edition = "2026"`, which protox may not parse.

### Cited Findings
- `test_messages_proto3.proto` (syntax proto3) defines `TestAllTypesProto3` (nested `NestedMessage` with
  `corecursive`, `recursive_message = 27`, maps 56–74 covering every scalar key type and
  string→message/enum values, `oneof oneof_field` from 111), `ForeignMessage`, `NullHypothesisProto3` and
  `EnumOnlyProto3` — [test_messages_proto3.proto](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/test_messages_proto3.proto)
- Proto3 unittest files and their syntax on main: `unittest_proto3.proto` (proto3; `TestAllTypes`,
  `TestPackedTypes`, `TestUnpackedTypes`, `NestedTestAllTypes`, `TestEmptyMessage`, `TestOneof2`...),
  `unittest_proto3_optional.proto` (proto3; `TestProto3Optional`), `unittest_well_known_types.proto` (proto3;
  WKTs singular/repeated/oneof/map). These are **edition 2026**: `map_proto3_unittest.proto`,
  `unittest_drop_unknown_fields.proto` (`Foo`/`FooWithExtraFields`, an old/new schema pair) and
  `unittest_preserve_unknown_enum.proto` (`MyMessage`/`MyMessagePlusExtra`) — local `grep` of the first
  `syntax`/`edition` line, [src/google/protobuf/](https://github.com/protocolbuffers/protobuf/tree/main/src/google/protobuf)
- The golden binaries are tracked only at `objectivec/Tests/golden_message` and
  `objectivec/Tests/golden_packed_fields_message`. C++'s `message_unittest.inc` writes a temporary
  `golden_message` from `TestUtil::SetAllFields` at test time. `src/google/protobuf/testdata/` holds text data:
  `map_test_data.txt`, `text_format_unittest_data*.txt` and `bad_utf8_string` — local `git ls-files`,
  [message_unittest.inc](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/message_unittest.inc), [map_test.inc](https://github.com/protocolbuffers/protobuf/blob/main/src/google/protobuf/map_test.inc)
- The editions transforms of the test messages are kept as goldens:
  `editions/golden/test_messages_proto3_editions.proto` and `edition2024_transform/test_messages_proto3.proto` — local `git ls-files`
- prost-reflect compiles `test_messages_proto2.proto`, `test_messages_proto3.proto` and `unittest.proto`
  for its tests — [prost-reflect build.rs](https://github.com/andrewhickman/prost-reflect/blob/main/prost-reflect-conformance-tests/build.rs)
- `npx conformance_proto_eject <dir>` copies the conformance test protos out of the npm package, so they
  stay in step with the runner version — [npm protobuf-conformance](https://www.npmjs.com/package/protobuf-conformance)

### Inferences
- **Take the test protos from the runner package** (`conformance_proto_eject`), not from protobuf main. Then
  the `.proto` files and the runner always agree, and rustpb never meets edition-2026 syntax by accident.
- `unittest_drop_unknown_fields.proto` and `unittest_preserve_unknown_enum.proto` are ready-made
  **old/new schema pairs** for evolution tests (Q4). Port their content to proto3 syntax if protox can't
  parse edition 2026.
- `golden_message` and `golden_packed_fields_message` are proto2 `unittest.proto` messages, with groups,
  extensions and defaults. They don't help a proto3-only codec beyond fuzz seeds.
- The highest-value fixture for rustpb is **its own golden set from ScalaPB on the app's real messages**
  (design §6.1). The upstream messages cover protobuf semantics; they don't cover ScalaPB options
  (TypeMapper, `no_box`, sealed oneofs, `collection_type = Set`).

### Gaps
- I did not decode the objectivec golden files to check which message and protobuf version they match.
- Whether protox (the pure-Rust parser) accepts `edition = "2026"`: not checked.

---

## Q3. ScalaPB's own test suite, and ScalaPB as a conformance testee or differential oracle

### Takeaway
ScalaPB already ships a conformance testee, `conformance/ConformanceScala.scala`. It is identical on master
and `0.11.x`, about 160 lines, uses the standard 4-byte LE framing, and builds as a self-executing assembly
jar. But it uses **scalapb-json4s, not scalapb-jsoniter**, and runs a **protobuf 3.19.6** runner, so its
failure list doesn't describe our JSON contract. Its property test compiles **random schemas** and checks
ScalaPB against protobuf-java. scalapb-jsoniter has no conformance runner; its tests compare against Java's
`JsonFormat`. A long-running JVM oracle can reuse conformance.proto unchanged: put the app's message full
name in `message_type`, and use jsoniter 0.1.2 for JSON.

### Cited Findings
- `ConformanceScala.scala`: a `@tailrec runTests` loop. It reads a 4-byte LE length (`readLittleEndianIntFromStdin`),
  reads the request, and matches `messageType` against `TestAllTypesProto3`, `TestAllTypesProto2` and
  `conformance.FailureSet`. Payload handling: `parseFrom` for bytes, the json4s
  `JsonFormat.parser.withTypeRegistry(...)` for JSON (`.ignoringUnknownFields` when
  `testCategory.isJsonIgnoreUnknownParsingTest`), and `fromAscii` for text. Output is `toByteString`,
  `JsonPrinter.print` (with `serialize_error` on throw) or `toProtoString`. A parse exception becomes
  `parse_error(err.getMessage)`. EOF exits 0 — [ScalaPB conformance/src/main/scala/ConformanceScala.scala](https://github.com/scalapb/ScalaPB/blob/master/conformance/src/main/scala/ConformanceScala.scala)
- `git diff origin/0.11.x master -- conformance` is empty: the 0.11.x testee is the same — local diff
- build.sbt: `lazy val conformance` is a JVM-only, Scala 2.13 project depending on `scalapb-json4s`
  (master: `1.0.0-alpha.1`). Its main class is `scalapb.ConformanceScala`, and `assemblyPrependShellScript`
  makes the jar itself executable. CI runs `sbt conformanceJVM2_13/assembly`, then
  `conformance_test_runner --enforce_recommended --failure_list conformance/failure_list_scalapb.txt --text_format_failure_list conformance/text_format_failure_list_scalapb.txt conformance/target/jvm-2.13/conformance` — [ScalaPB build.sbt](https://github.com/scalapb/ScalaPB/blob/master/build.sbt), [.github/workflows/test.yml](https://github.com/scalapb/ScalaPB/blob/master/.github/workflows/test.yml)
- ScalaPB's failure list (13 entries, old naming from the 3.19.6 runner): FieldMask round trips,
  `BoolFieldDoubleQuoted{False,True}`, `BytesFieldBase64Url.*`, `FieldMaskInvalidCharacter`,
  `NullValueInOtherOneofNewFormat.Validator`, `StringFieldSurrogateInWrongOrder`,
  `StringFieldUnpaired{High,Low}Surrogate`, and `Proto2...FieldNameExtension.Validator`. The text list includes
  `*UnknownFields_{Drop,Print}.TextFormatOutput` — [failure_list_scalapb.txt](https://github.com/scalapb/ScalaPB/blob/master/conformance/failure_list_scalapb.txt), [text_format_failure_list_scalapb.txt](https://github.com/scalapb/ScalaPB/blob/master/conformance/text_format_failure_list_scalapb.txt)
- The property test `proptest/GeneratedCodeSpec.scala`, "Java and Scala protos are equivalent" (tagged `Slow`,
  `minSuccessful(20)` schemas). `SchemaGenerators.genCompiledSchema` makes random `.proto` schemas
  (`GraphGen`, `GenTypes`) and compiles them with both protoc-java and ScalaPB. `GenData.genMessageValueInstance`
  makes values as text format. The test then asserts: Scala bytes reparse to the same value; Java parses
  Scala's bytes to the same `javaProto`; `toJavaProto`/`fromJavaProto` round-trip; Java and Scala parse each
  other's text format; and descriptor full names match — [ScalaPB proptest/src/test/scala/GeneratedCodeSpec.scala](https://github.com/scalapb/ScalaPB/blob/master/proptest/src/test/scala/GeneratedCodeSpec.scala)
- The e2e suite holds per-feature specs (`CustomTypesSpec`, `NoBoxSpec`, `SealedOneofSpec`,
  `CollectionTypesSpec`, `RequiredFieldsSpec`, `UnknownFieldSetSpec`, `MapsSpec`, `EnumSpec`,
  `WellKnownTypesSpec`, ...). `e2e-withjava` builds with Java interop as well — [ScalaPB e2e/src/test/scala](https://github.com/scalapb/ScalaPB/tree/master/e2e/src/test/scala)
- ScalaPB's `UnknownFieldSet` keeps `fields: Map[Int, UnknownFieldSet.Field]`, filled by a
  `mutable.HashMap[Int, Field.Builder]` builder, with `parseField` matching `WIRETYPE_VARINT/FIXED64/LENGTH_DELIMITED/FIXED32` — [scalapb-runtime UnknownFieldSet.scala](https://github.com/scalapb/ScalaPB/blob/master/scalapb-runtime/src/main/scala/scalapb/UnknownFieldSet.scala)
- scalapb-jsoniter contains no conformance runner; grep for "conformance" finds nothing. Its tests
  (`JsonFormatSpec`, `OneOfSpec`, `WellKnownTypesSpec`, `StructFormatSpec`, `AnyFormatSpec`, ...) use a
  `JavaAssertions` trait that compares the Scala `Parser`/`Printer` with Java's `JsonFormat` on the JVM
  (`JavaAssertionsPlatform` is empty on JS/Native) — [scalapb-jsoniter core/shared/src/test](https://github.com/scalapb/scalapb-jsoniter/tree/master/core/shared/src/test/scala/scalapb_jsoniter)
- scalapb-json4s has `ParsingConformanceSpec` (base64 alphabets, NullValue oneof presence, quoted wrapper
  bools, enum JSON names) and a ScalaCheck `ArbitrarySpec` — [scalapb-json4s src/test](https://github.com/scalapb/scalapb-json4s/tree/master/src/test/scala/scalapb/json4s)

### Inferences
- **Oracle process design, reusing conformance.proto with no changes:**
  - Base it on `ConformanceScala.scala` and build it **inside the Scala app's build**, so it uses the app's
    ScalaPB 0.11.x generated code and its TypeMappers.
  - Replace json4s with **scalapb-jsoniter 0.1.2** `Parser`/`Printer`, configured exactly as the app's codec
    (`includingDefaultValueFields = false`; design §5).
  - Look up companions by `message_type` (any app message full name) in a map built from the generated
    `*Proto.messagesCompanions`.
  - Keep the 4-byte LE framing and the `parse_error`/`serialize_error`/`protobuf_payload`/`json_payload`
    results. The JVM stays warm for the whole session.
  - `test_category = JSON_IGNORE_UNKNOWN_PARSING_TEST` can stand for "lenient parse", if the app ever uses
    one.
  - The same process then serves three drivers:
    1. Google's runner, with `TestAllTypesProto3` compiled by ScalaPB 0.11.x. This **measures** the exact
       ScalaPB+jsoniter-0.1.2 failure list that rustpb's list must equal.
    2. rustpb's golden generator.
    3. rustpb's differential fuzzer, which sends mutated bytes or JSON to both the oracle and the rustpb
       testee and compares the results variant (success or failure) and the re-encoded payloads.
- Byte comparison has to account for the design's non-deterministic cases: sets, maps, and the order of
  unknown field numbers. Normalize before comparing: ask both sides for the binary output, decode with one
  canonicalizing reader, and compare. Or add a "canonical" output mode to the oracle. The conformance runner
  itself compares with `MessageDifferencer`, for the same reason.
- Panics map naturally onto the protocol. A Rust panic kills the testee, and the runner turns that into
  `runtime_error`, which always fails. So "never panics" is enforced for free on every malformed-input test,
  native and JS.
- ScalaPB's `proptest` approach (random schemas × random values, compared with a second implementation) can
  be copied directly. Generate random `.proto` files that use ScalaPB options, compile them with ScalaPB and
  rustpb, and compare bytes and JSON. This covers code-generation paths that a fixed corpus never reaches.

### Gaps
- I did not check whether scalapb-jsoniter 0.1.2 builds against ScalaPB 0.11.x for a standalone oracle
  outside the app; the app already does this, per design §5.
- I did not read `proptest/SchemaGenerators.scala` in detail, so I can't say which proto features it
  generates (maps, oneofs, enums are likely, given `GenTypes`).

---

## Q4. Schema evolution and compatibility checks

### Takeaway
`buf breaking` with category `WIRE_JSON` is the standard static check for "old bytes and old JSON stay
readable". Its rules catch deleted-but-unreserved numbers and names, incompatible type changes, json_name
changes and oneof moves. The dynamic check is to encode with the old schema, decode with the new one, then
re-encode and decode with the old schema. upstream's `Foo`/`FooWithExtraFields` pairs are models for this.

### Cited Findings
- Buf's breaking categories, strictest first: `FILE` (the default), `PACKAGE`, `WIRE_JSON` ("recommended
  minimum" because JSON is common) and `WIRE` (binary only) — [Buf docs: breaking rules](https://buf.build/docs/breaking/rules/)
- `WIRE` rules: `ENUM_VALUE_NO_DELETE_UNLESS_NUMBER_RESERVED`, `FIELD_NO_DELETE_UNLESS_NUMBER_RESERVED`,
  `FIELD_SAME_DEFAULT`, `FIELD_SAME_ONEOF`, `FILE_SAME_PACKAGE`, `FIELD_WIRE_COMPATIBLE_CARDINALITY`,
  `FIELD_WIRE_COMPATIBLE_TYPE`, `MESSAGE_SAME_REQUIRED_FIELDS`, `RESERVED_ENUM_NO_DELETE`,
  `RESERVED_MESSAGE_NO_DELETE`, and five `RPC_SAME_*` rules — [Buf docs](https://buf.build/docs/breaking/rules/)
- `WIRE_JSON` adds: `ENUM_SAME_JSON_FORMAT`, `ENUM_VALUE_NO_DELETE_UNLESS_NAME_RESERVED`,
  `ENUM_VALUE_SAME_NAME`, `FIELD_NO_DELETE_UNLESS_NAME_RESERVED`, `FIELD_SAME_JSON_NAME`, `FIELD_SAME_NAME`,
  `FIELD_WIRE_JSON_COMPATIBLE_CARDINALITY`, `FIELD_WIRE_JSON_COMPATIBLE_TYPE`, `MESSAGE_SAME_JSON_FORMAT` — [Buf docs](https://buf.build/docs/breaking/rules/)
- Wire-compatible changes from the proto3 guide:
  - `int32/uint32/int64/uint64/bool` are compatible, with out-of-range values "truncated, as in a C++ cast".
  - `sint32`↔`sint64` only.
  - `string`↔`bytes` if the bytes are valid UTF-8.
  - An embedded message ↔ `bytes`.
  - `fixed32`↔`sfixed32` and `fixed64`↔`sfixed64`.
  - Singular↔`repeated` only for string, bytes and message ("packed encoding won't parse correctly as a
    singular field").
  - `enum`↔int types.
  - `map<K,V>`↔ a repeated entry message (map clients "may reorder entries or drop duplicate keys").

  Moving fields into an *existing* oneof is unsafe. Reserve deleted names "so JSON and TextFormat encodings keep working". Unknown fields "are lost when you serialize to JSON" — [protobuf.dev proto3 guide, Updating a Message Type](https://protobuf.dev/programming-guides/proto3/)
- Upstream evolution fixtures: `unittest_drop_unknown_fields.proto` (`Foo` vs `FooWithExtraFields`) and
  `unittest_preserve_unknown_enum.proto` (`MyMessage` vs `MyMessagePlusExtra`) — local [src/google/protobuf](https://github.com/protocolbuffers/protobuf/tree/main/src/google/protobuf)

### Inferences
- For RustPB there are two separate evolution questions:
  - **(a) The schema's evolution**: the app's own `.proto` changes over time. Run `buf breaking --against <git ref>`
    with `WIRE_JSON` on every schema change. Add an exception rule for ScalaPB options: changing
    `(scalapb.field).type` or `(rustpb.field).type` is wire-neutral by definition.
  - **(b) The two implementations agree across versions**: ScalaPB writes with schema version N and rustpb
    reads with N+1, and the reverse. Build this as a fixture matrix: for each pair of an old and a new message
    (added field, removed+reserved field, enum value added, int32→int64, singular→repeated string, message→bytes),
    check ScalaPB-old-encode → rustpb-new-decode → rustpb-encode → ScalaPB-old-decode. The value must be equal
    and the **unknown fields preserved**. This is where `preserve_unknown_fields` (default true, design §2)
    is actually tested.
- The design's TypeMapper adds a hazard buf cannot see: a mapper whose `to_custom` rejects a value that an
  older writer produced (default-on-bad-id). Old stored records (design §6.3, "real data") are the oracle for
  this; `buf breaking` is not.
- An enum value added in N+1 and read by N must stay `Unrecognized(n)` and re-encode to the same number,
  in rustpb and ScalaPB alike. Pin this as a fixture.

### Gaps
- I didn't research Buf's handling of custom options in `buf breaking`, so I can't say whether
  `(scalapb.*)` option changes are ignored or flagged.
- Beyond these protobuf.dev rules I found no published cross-implementation evolution test suite: no
  standard "schema evolution conformance".

---

## Q5. Edge-case test vectors implementations most often get wrong

### Takeaway
The conformance suite already encodes most canonical edge cases: varint length and overflow, illegal tags,
truncation at every position, groups, unknown wire types 6 and 7, packed/unpacked mixing, last-wins and
merge semantics, oneof merging, UTF-8, depth limits, and JSON number and NaN spellings. RustPB-specific
vectors must be added where ScalaPB's policy differs from the conformance reference: unknown-field order,
`-0.0` omission, wrong-wire-type-as-unknown, and the JSON leniencies.

### Cited Findings (from `binary_json_conformance_suite.cc` on main unless noted)
- **Tags**:
  - `IllegalZeroFieldNum_Case_0..3`: field number 0.
  - `BadTag_FieldNumberTooHigh`: a varint tag far above the max, whose low 5 bytes look like field 1.
  - `BadTag_FieldNumberSlightlyTooHigh`: a 5-byte tag above UINT32_MAX, with bit 35 set.
  - `BadTag_OverlongVarint`: an overlong but in-range tag. **Must be accepted.**
  - `BadTag_VarintMoreThanTenBytes`.

  — [binary_json_conformance_suite.cc `TestIllegalTags`](https://github.com/protocolbuffers/protobuf/blob/main/conformance/binary_json_conformance_suite.cc)
- **Lengths**: `BadLength_Varint32BitOverflow`, a length varint `\x80\x80\x80\x80\x10` (REQUIRED) — same file, `TestIllegalLengths`
- **Truncation**: `PrematureEof{BeforeKnownNonRepeatedValue, BeforeKnownRepeatedValue, BeforeUnknownValue, InsideKnownNonRepeatedValue, InsideKnownRepeatedValue, InsideUnknownValue, InDelimitedDataForKnownNonRepeatedValue, InDelimitedDataForKnownRepeatedValue, InDelimitedDataForUnknownValue, InPackedField, InPackedFieldValue, InSubmessageValue}`, for each field type — `TestPrematureEOFForType`
- **Groups**: `UnmatchedStartGroup{,Nested,NestedLen,Unknown,WithData,WrongType}`,
  `UnmatchedEndGroup{...same}`, `MismatchedGroupTags`, `MismatchedNestedGroupTags` — `TestUnmatchedGroup`
- **Unknown wire types**: `UnknownWireType{6,7}_Field{0..3}_Version{0..3}` must fail to parse — `TestUnknownWireType`
- **Unknown fields**: `UnknownVarint` (round-trip preserved), and `UnknownOrdering` (same number, wire types
  interleaved, order kept; REQUIRED) — `TestUnknownOrdering`
- **Repeated and merge semantics**: `ValidDataRepeated.<TYPE>.{PackedInput,UnpackedInput}`,
  `RepeatedScalarSelectsLast<TYPE>` (a repeated scalar parsed as a singular field: last wins),
  `RepeatedScalarMessageMerge`, `ValidDataScalarBinary` (byte-exact), `ValidDataMap`, `ValidDataOneof[Binary]`,
  `MergeOneofMessage`, and `ValidLengthPrefixedField` — `TestValidDataForType`, `TestValidDataForMapType`, `TestValidDataForOneofType`
- **UTF-8**: `RejectInvalidUtf8.String.{Singular,Repeated,Oneof,MapKey,MapValue}`, all **RECOMMENDED**, not
  required — `TestInvalidUtf8String`
- **Depth**: `EnforceDepthLimit.{Map,MapStringKey}` (RECOMMENDED), built 10,000 levels deep; "The default recursion limit is 100 for most languages". JSON: `{Struct,ListValue,Value}DeepNesting{25,200}` (RECOMMENDED) — `RunRecursionLimitTests`
- **JSON numbers**:
  - `Int32Field{LeadingZero, NegativeWithLeadingZero, PlusSign, LeadingSpace, TrailingSpace, TooLarge, TooSmall, NotInteger, ExponentialFormat, FloatTrailingZero, MaxFloatValue, StringValue*}`,
    `Int64Field{MaxValueNotQuoted, MinValueNotQuoted, BeString, StringValueFullwidthDigits}`.
    `Int64FieldMaxValueNotQuoted` uses `9223372036854774784`, the largest double-representable value below
    2^63 (REQUIRED).
  - `Double/FloatField{Nan, Infinity, NegativeInfinity, ...NotQuoted, ...LeadingWhitespace, ...TrailingWhitespace, NormalizeQuietNan, NormalizeSignalingNan, StringValueArabicIndicDigits, StringValueFullwidthDigits, TooLarge, TooSmall, Min/MaxPositiveValue}`.
- **JSON structure**: `FieldNameInSnakeCase` and `OriginalProtoFieldName` (REQUIRED), `FieldNameDuplicate*`,
  `OneofFieldDuplicate{,2}` (valid-or-error), `OneofFieldNull{First,Second}`, `RejectTopLevelNull`,
  `TrailingComma*`, `JsonWithComments`, `StringFieldUnpaired{High,Low}Surrogate`,
  `StringFieldSurrogateInWrongOrder`, and `IgnoreUnknownJson{Number,String,True,False,Null,Object}` (in the
  ignore-unknown category, REQUIRED). Plus Duration/Timestamp range and format: `DurationProtoNanosWrongSign`,
  `TimestampJsonInputLowercaseZ`, and others — test names from local `grep`
- The upstream lists of the most mature implementations still carry JSON leniency failures. protobuf-java
  lists: quoted, capitalized and camel-case bools; unquoted NaN/Infinity; unquoted map keys; single quotes;
  comments; surrogate errors; and `Required...Int32FieldLeadingZero`, `...PlusSign`, `EnumFieldNotQuoted`,
  `DoubleFieldStringValueArabicIndicDigits` — [failure_list_java.txt](https://github.com/protocolbuffers/protobuf/blob/main/conformance/failure_list_java.txt)
- 64-bit integers in JS: without BigInt, protobuf-es fails exactly `{Int64,Uint64}Field{Max,Min}ValueNotQuoted` — [protobuf-es failing_tests_without_bigint.txt](https://github.com/bufbuild/protobuf-es/blob/main/packages/protobuf-conformance/failing_tests_without_bigint.txt)

### Inferences
These are the vectors to add beyond the suite, each recorded against the ScalaPB oracle (design §6.4
already lists most):
- **Varint boundaries**, for each 32/64-bit type: 0, 1, 127, 128, 16383, 16384, 2^31−1, −1 (int32 as
  10 bytes), −2^31, 2^63−1, 2^64−1. Also a 10-byte varint whose last byte is `0x02` or more: Java's
  `CodedInputStream` throws "malformed varint" only after 10 bytes and accepts high bits in byte 10. Record
  ScalaPB's answer, because Rust's arithmetic may differ.
- **zigzag**: `sint32` −1 → 1, `i32::MIN` → `0xFFFFFFFF`; `sint64` `i64::MIN`.
- **Floats**: `-0.0` singular (omitted by ScalaPB, design §4.6, where C++ conformance compares
  `MessageDifferencer`-equal anyway), `-0.0` in repeated (kept), NaN payloads (quiet or signaling; JSON
  normalizes them), subnormals, `f32::MAX` written to JSON, and float → JSON shortest round-trip digits (must
  match jsoniter digit-for-digit).
- **Packed field whose byte count runs past its parent**, and **a packed fixed32 length not divisible by 4**.
  Neither is a named conformance test. Inference from design §4.6; verify ScalaPB's behavior.
- **Wrong wire type for a known field** → unknown (ScalaPB) versus error (prost). Conformance doesn't pin
  this, so a ScalaPB fixture is the only oracle.
- **Map entries**: missing key, missing value, duplicate keys (last wins), an entry with unknown fields
  inside, and entry fields in reverse order (value before key).
- **Unknown fields across numbers** (rustpb writes ascending, design §4.6) and **within a number across wire
  types** (the UnknownOrdering conflict, Q1).
- **JS-only**: a 64-bit varint split across the 32-bit halves boundary (bit 31/32 carry), and `uint64`
  values above 2^53 through JSON both quoted and unquoted. protobuf-es's no-BigInt failures show the
  conformance suite only catches the second.

### Gaps
- I didn't read Java `CodedInputStream`'s exact 10th-byte rule for this note. The varint-overflow claim
  above is from memory and must be checked against ScalaPB 0.11.x's protobuf-java version before it becomes
  a fixture expectation.
- I didn't verify whether any conformance test feeds a packed length that isn't a multiple of the element
  width. The `PrematureEofInPackedField*` tests cover truncation only.
