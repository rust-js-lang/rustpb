# Testing and verification practices that make low-level libraries trusted, and how they show their evidence

Research date: 2026-11-13. Web pages fetched that day; local clones under
`~/Documents/GitHub/` read at their checked-out commit (dates given per repo).
Scope: input to RustPB's test-harness design (docs/design.md §1, §6).

## 1. Exemplars and their concrete methods

### Takeaway
The most trusted projects have three things in common: several independent oracles
(their own suites, a shared public corpus, and differential runs against other
implementations), a hard rule that the suite itself is measured (coverage,
mutation testing), and an exact, versioned list of known failures. Of these,
SQLite is the extreme case and the template; for protobuf, the
conformance runner plus an exact failure list is the trust standard in the ecosystem.

### Cited Findings

**SQLite** (page last updated 2026-04-21)
- Library is ~155.8 KSLOC of C; test code is ~92,053 KSLOC, "590 times as much test code" — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Four independent harnesses: TCL tests (51,445 distinct cases, "millions" after parameterization); TH3 (50,362 cases, ~2.4 million instances for full coverage, ~248.5 million in the pre-release soak); SQL Logic Test (7.2 million queries, run differentially against PostgreSQL, MySQL, SQL Server and Oracle); dbsqlfuzz (libFuzzer with a custom mutator, 336 seed files, ~500 million to 1 billion mutations a day) — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- 100% branch coverage and 100% MC/DC under TH3, measured with gcov. They describe the measurement as "a test of the test suite". The release build is then re-run, and its output must match — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- `testcase()` macro (1,184 uses) marks boundary conditions that must be exercised both ways. `ALWAYS()/NEVER()` macros mark defensive code; the suite runs three times (release, debug-assert, coverage builds), and "All three test runs should yield exactly the same result". There are 6,754 `assert()`s; asserts make the engine about 3x slower, so they are debug-only — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Mutation testing at the assembly level: each branch is turned into an unconditional jump or a no-op, and the suite must catch it. Equivalent mutants are suppressed with `/*OPTIMIZATION-IF-TRUE*/` comments — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Fuzzing history: AFL from 3.8.10 (2015) to 3.29.0 (2019); OSS-Fuzz since 2016; dbsqlfuzz since late 2018, after which "bug reports from external fuzzers such as OSSFuzz have all but stopped"; jfuzz (corrupt JSONB) since Jan 2024. `fuzzcheck` replays "a few thousand 'interesting' cases out of the billions" in `make test`. "most testing CPU cycles are now devoted to fuzzing" — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Noted tension: "code tested to 100% MC/DC will tend to be more vulnerable to problems found by fuzzing" (defensive branches get removed to reach coverage) — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- The full suite also runs with optimizations disabled, and results must match (a differential run against itself) — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Release checklist: ~200 items, verified manually per release — [How SQLite Is Tested](https://www.sqlite.org/testing.html)

**JSONTestSuite / "Parsing JSON is a Minefield"** (first published 2016-10-26; last revised 2018-03-30 for RFC 8259)
- More than 300 hand-written files, mostly written from the spec text. The file name prefix carries the expected result: `y_` must accept, `n_` must reject, `i_` implementation-defined (either is fine). `test_transform/` holds odd-but-valid inputs (huge numbers, duplicate-like keys, NUL) — [JSONTestSuite](https://github.com/nst/JSONTestSuite); [seriot.ch article](https://seriot.ch/security/parsing_json.html)
- The runner treats exit code 0 as accept, 1 as reject, >1 as a crash, and a 5 s limit as a timeout. It writes an HTML matrix, one row per file and one column per parser. Colors: red = crash, brown = should-have-passed, yellow = should-have-failed. A "pruned" view keeps one row per group of identical results — [seriot.ch article](https://seriot.ch/security/parsing_json.html)
- More than 30 parsers were tested. Deep nesting (long runs of `[`) crashed JSONKit, SBJSON, ccan, cJSON and JSON.sh; unclosed structures crashed Freddy. Of the Rust parsers, json-rust wrongly accepted `[1.]`. serde_json was not in the article — [seriot.ch article](https://seriot.ch/security/parsing_json.html)
- A follow-up on another project's parser found improper acceptance of a non-minimal UTF-8 sequence and partial parsing at a NUL byte — [bitcoin #9028 mirror](https://mirror.b10c.me/bitcoin-bitcoin/9028/)

**Protobuf conformance runner** (local clone `protobuf`, commit 2026-11-13)
- Every official implementation commits a `failure_list_<lang>.txt`. Current sizes: cpp 78 lines, java 82, python 65, php 95, jruby 187, objc 2, rust_upb 11, rust_cc 12, csharp 41 — `~/Documents/GitHub/protobuf/conformance/failure_list_*.txt`
- The list is strict in both directions. A listed test that passes fails the run ("These tests succeeded, even though they were listed in the failure list ... Remove their match"). Stale entries, and wildcard entries that match too many tests, are also errors — `~/Documents/GitHub/protobuf/conformance/conformance_test.cc` lines 369, 691–735
- Lists record their own comments, such as an entry a test cannot yet express ("TODO: Uncomment once conformance tests can express failures that are not expected to be fixed") — `failure_list_rust_upb.txt`

**Buf's cross-implementation conformance table** (fetched 2026-11-13; no last-updated date shown)
- A README table of failure counts per implementation, split into Required and Recommended, covering binary and ProtoJSON (text format tracked separately), each tested at proto2, proto3 and its maximum edition. Protobuf-ES and protobuf.js: 0/0; protobuf-ts 6/7; google-protobuf 1169/389; ts-proto 751/613; protoscript 1151/541; protoc-gen-ts 1690/475. Each count is an SVG badge. Only TypeScript/JavaScript implementations are covered — [bufbuild/protobuf-conformance](https://github.com/bufbuild/protobuf-conformance)

**Rust protobuf peers** (local clones)
- buffa (commit 2026-10-20): its README states that it "passes the full protobuf conformance suite — binary, JSON, and text — with zero expected failures" (v33.5, editions up to 2024; `std` and `no_std`). The repo nevertheless keeps one known-failure file per codegen mode (`known_failures.txt` is comments only; view/table/lazy/reflect/text variants hold 3–14 lines). It is tested against protoc v21.12, 22.5, 25.5, 27.3, 29.5 and 33.5. CI runs Miri on its `unsafe` `set_len` path. Fuzz targets: decode_proto2, decode_proto3, decode_wkt, encode_proto3, json_roundtrip, wkt_json_strings — `~/Documents/GitHub/buffa/README.md` §Conformance, `conformance/`, `fuzz/fuzzers/`, `.github/workflows/ci.yml`
- prost (commit 2026-11-12): CIFuzz on every PR via the OSS-Fuzz project `prost`, `fuzz-seconds: 300`; fuzz targets proto2, proto3, parse_date, parse_duration, plus an AFL setup; CI jobs include `cargo-semver-checks`, `cargo minimal-versions`, no-std and Kani — `~/Documents/GitHub/prost/.github/workflows/cifuzz.yml`, `ci.yml`
- **Correction for design.md §6 item 10:** prost's Kani harnesses exist only in `prost-types/src/timestamp.rs` and `duration.rs`, and the CI job runs `-p prost-types --features chrono`. prost has no Kani proof of varint or zigzag arithmetic. The claim "as prost's is" does not hold — `~/Documents/GitHub/prost/.github/workflows/ci.yml` lines 178–186; `grep kani` over prost
- protobuf-go: fuzzers `wirefuzz`, `jsonfuzz`, `textfuzz`, run continuously by OSS-Fuzz — `~/Documents/GitHub/protobuf-go/internal/fuzz/README.md` (commit 2025-09-09)

**rustls**
- The README shows CI, Codecov coverage and an OpenSSF Best Practices badge (project 9034). The repo has `fuzz/`, `rustls-fuzzing-provider/`, `bogo/`, `openssl-tests/`, `connect-tests/`, `ci-bench/`, `audit/` and `SECURITY.md` — [rustls/rustls](https://github.com/rustls/rustls)
- BoGo (BoringSSL's TLS test suite) runs through a shim. In 0.23.x it runs as ignored tests, once per crypto provider (ring, aws-lc-rs, aws-lc-rs-fips) — [rustls tests/bogo.rs (docs.rs, 0.23.12)](https://docs.rs/crate/rustls/0.23.12/source/tests/bogo.rs)
- BoringSSL seeds its TLS fuzzers from BoGo transcripts, and keeps a suppressions file for tests that fail in fuzzer mode — [BoringSSL FUZZING.md](https://boringssl.googlesource.com/boringssl/+/refs/tags/0.20250701.0/FUZZING.md)

**Crypto test vectors: Wycheproof**
- JSON vector files with JSON Schemas, covering ~40 algorithm families. It is now maintained by the community under C2SP. Bundled harnesses were removed; consumers write their own loader and run the vectors in CI. It lists 20 consumers, among them OpenSSL, BoringSSL, aws-lc, NSS, Go crypto, RustCrypto and pyca/cryptography — [C2SP/wycheproof](https://github.com/C2SP/wycheproof)
- AWS-LC pairs Wycheproof, Cryptofuzz fuzzing, sanitizers and Valgrind with machine-checked proofs (SAW/Coq, HOL Light for s2n-bignum) that run in its CI. The proofs rest on stated, admitted assumptions — [aws-lc-verification](https://github.com/awslabs/aws-lc-verification); [NIST talk PDF](https://www.nist.gov/document/formal-verification-cryptographic-software-aws-current-practices-and-future-trends)

### Inferences
- The pattern RustPB already follows in §6 (ScalaPB as differential oracle, Google's runner as secondary oracle) maps onto SQLite's SLT (differential against other engines) plus TH3 (own suite). What it lacks next to SQLite: (a) a measure of the suite itself (coverage plus mutation score), and (b) a committed replay corpus of interesting fuzz finds run on every `cargo test` (SQLite's `fuzzcheck`).
- The `y_/n_/i_` prefix convention is directly reusable for §6.4's decoding fixtures: `y_` (must decode to the recorded value), `n_` (must be a `DecodeError`), `i_` (ScalaPB-defined; recorded). The file name then documents the expectation, and a results matrix (RustPB native / RustPB-on-JS / ScalaPB / prost / buffa) shows the evidence.
- A two-way strict failure list, as in Google's runner and buffa, is the ecosystem's trust currency. buffa's README claim of "zero expected failures", next to its per-mode lists, shows why each list should be published per mode (native, rust-js, JSON) with its count.
- Peers set the bar: a new protobuf library is now judged against "0 required failures" (Protobuf-ES, protobuf.js, buffa).

### Gaps
- serde_json's own test organization and any published JSONTestSuite matrix were not fetched. I found no source showing serde_json results in the original article.
- Hypothesis-style and Jepsen-style public reports were not researched in this pass.
- I could not find rustls audit reports, fuzz hours or coverage percentages from primary pages. The README shows only the badges.
- I found no primary source on how aws-lc-rs (the Rust wrapper) itself consumes Wycheproof.
- No CPU-hour figures for SQLite fuzzing ("the page gives no CPU-hour totals").

## 2. Anomaly and failure-injection testing applicable to a codec

### Takeaway
SQLite's anomaly method is exhaustive and simple: fail the Nth operation, for N = 1, 2, 3, … until the operation succeeds, run each N twice (fail once, then fail forever after), and check invariants after each run. For a codec the same loop becomes "every prefix", "every single-bit flip" and "every allocation". These are cheap natively. The real constraint is that the ScalaPB differential oracle runs on the JVM.

### Cited Findings
- OOM: a malloc wrapper that "can be rigged to fail after a certain number of allocations". The loop starts by failing the first allocation and increments until the operation completes. Each point runs twice: fail once, and fail continuously after the first failure — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- I/O errors: a VFS that fails after N operations, using the same loop, followed by `PRAGMA integrity_check`. Crash tests reorder and corrupt unsynced writes, then check atomicity. Compound faults are also tested (OOM during crash recovery) — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Malformed input: bytes of database files are modified, and the engine must report `SQLITE_CORRUPT` "without overflowing buffers, dereferencing NULL pointers, or performing other unwholesome actions" — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Leak detection on every test run, including after OOM and I/O errors — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Boundary values: tests push "right to the edge of its defined limits", and tests just beyond the limits check that errors are returned — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Depth bombs are a recurring real failure: JSON parsers crashed on deep `[` nesting — [seriot.ch](https://seriot.ch/security/parsing_json.html). The Rust `protobuf` crate's advisory RUSTSEC-2024-0437 / CVE-2025-53605 was uncontrolled recursion in `CodedInputStream::skip_group` on unknown groups, fixed in 3.7.2 with a depth check — [RustSec](https://rustsec.org/advisories/RUSTSEC-2024-0437)
- prost's fuzz targets are a single-sided round-trip; buffa adds encode and JSON round-trip targets — local `prost/fuzz/fuzzers`, `buffa/fuzz/fuzzers`

### Inferences
- Exhaustive mutations per corpus message of n bytes: n truncations (every prefix), 8n single-bit flips, and 256·n byte substitutions if wanted. Example (an assumption, not measured): 10,000 golden messages averaging 200 bytes give 2 M prefixes and 16 M bit flips. Native Rust decoding takes microseconds per message, so this is seconds to minutes and fits a nightly tier, or a per-PR tier on a sampled subset. Expected results come from cheap invariants: never panic, `Err` or `Ok`, `Ok` re-encodes stably, and `encoded_len` equals the bytes written.
- Comparing against ScalaPB for every mutant is the expensive part (JVM process, but batchable). Use a batch oracle server: send all mutants in one request and get back recorded outcomes, then commit the outcomes as goldens so the per-PR run needs no JVM. Re-record nightly.
- Allocation failure: Rust's `Vec` aborts on OOM rather than returning an error, so SQLite-style OOM loops apply only if RustPB promises fallible allocation. The codec-relevant analogue is length-prefix amplification: a huge declared length must not pre-allocate. Test it with a counting global allocator that asserts peak allocation ≤ k × input length.
- Depth limits should be tested at exactly limit−1, limit and limit+1 for each recursive shape (message, group, map value, JSON array/object, Any/Struct/Value), following SQLite's boundary rule. Each Rust protobuf advisory seeds a fixture (design §6.7 already says this).
- Run each anomaly case in both the native and rust-js builds, since JS has a different stack depth and BigInt paths.

### Gaps
- No published cost numbers for exhaustive prefix or bit-flip testing in a protobuf library were found. The estimates above are illustrative only.
- Not researched: whether ScalaPB's JVM parser has its own recursion limit value that RustPB must match exactly (design §4.6 says 100; to be confirmed against ScalaPB/protobuf-java source).

## 3. Making evidence visible

### Takeaway
Trust grows from evidence a reader can check: a "How X is tested" page with numbers (SQLite), a public results matrix (JSONTestSuite, Buf), committed failure lists, and machine-scored hygiene (OpenSSF Scorecard and Best Practices badge). Badges alone are weak signals, since Scorecard only checks that files exist.

### Cited Findings
- SQLite's single page lists harness sizes, test counts, coverage method, anomaly methods, fuzzing history and the release checklist, with a last-updated stamp — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Buf publishes per-implementation failure counts as SVG badges in a README table — [bufbuild/protobuf-conformance](https://github.com/bufbuild/protobuf-conformance)
- OpenSSF Scorecard checks include CI-Tests, Fuzzing, SAST, Signed-Releases, Pinned-Dependencies, Branch-Protection, Code-Review, Vulnerabilities (OSV), Security-Policy, Dangerous-Workflow, Token-Permissions, SBOM, Dependency-Update-Tool, Binary-Artifacts, Maintained and others. Fuzzing counts OSS-Fuzz listing, ClusterFuzzLite, Go native fuzzing, and some property-testing libraries in other languages. The fetched docs do not list cargo-fuzz or Rust property testing. Signed-Releases gives 8 points for a signature per release and 10 for SLSA provenance (`*.intoto.jsonl`), and "does not verify the signatures" — [Scorecard checks](https://github.com/ossf/scorecard/blob/main/docs/checks.md)
- OpenSSF Best Practices (passing level) MUSTs include: a FLOSS automated suite with documented invocation; a policy that major new functionality gets tests, plus evidence it was followed (`tests_are_added`); warnings enabled and fixed; static analysis before major releases; a published private vulnerability-reporting process; a first response within 14 days; no medium-or-higher public vulnerability unpatched for more than 60 days; known vulnerabilities named in release notes — [bestpractices.dev criteria](https://www.bestpractices.dev/en/criteria/0)
- rustls displays CI, Codecov and Best Practices badges and has a SECURITY.md — [rustls/rustls](https://github.com/rustls/rustls)
- cargo-vet checks every third-party dependency against recorded audits. It supports sharing audits across projects, relative (diff) audits between versions, and recorded exemptions to work down — [cargo-vet book](https://mozilla.github.io/cargo-vet/)
- RustSec advisories are machine-readable (OSV) and consumed by tooling. RUSTSEC-2024-0437 is the protobuf-specific precedent — [RustSec](https://rustsec.org/advisories/RUSTSEC-2024-0437)

### Inferences
- A `docs/TESTING.md` ("How RustPB is tested"), modelled on SQLite's page: each oracle with its counts, corpus sizes, fuzz targets and time per tier, the anomaly loops, the mutation score, a coverage figure, and a last-updated date. Generate the numbers from CI artifacts, never by hand, so the page cannot drift.
- A published results matrix: rows = fixture or conformance test, columns = RustPB native, RustPB via rust-js, ScalaPB, prost, buffa. It is generated by the harness and published per release. This applies the JSONTestSuite/Buf approach to RustPB's specific claim (ScalaPB equivalence).
- Committed, strict, per-target failure lists (native binary, native JSON, rust-js binary, rust-js JSON), each line commented with its reason or ADR, including the §5 scalapb-jsoniter deviations. README badges show the counts.
- Hygiene that Scorecard and Best Practices can verify: SECURITY.md, pinned actions, least-privilege tokens, release provenance, `cargo deny` (licenses, advisories, bans), `cargo audit`, and possibly `cargo vet`. Since Scorecard's Fuzzing check may not detect cargo-fuzz, an OSS-Fuzz or ClusterFuzzLite listing is the visible route (prost uses OSS-Fuzz CIFuzz).

### Gaps
- Not verified whether Scorecard's current code (not its docs) detects Rust cargo-fuzz targets.
- Silver and gold Best Practices criteria (coverage thresholds) were not on the fetched page.
- I did not research release-provenance tooling for crates.io specifically (crates.io trusted publishing, attestation status as of 2026-11).

## 4. Versioning and compatibility promises that hold tests to account

### Takeaway
The strongest promises are written down precisely: what may break and why, notice periods, and generator–runtime skew windows. Each is enforced by a CI check: semver checks, MSRV build, minimal-versions, and a protoc version matrix. For RustPB the central promise is wire and JSON equivalence with a pinned ScalaPB version, so the golden corpus is the contract's test.

### Cited Findings
- protobuf-go's README lists the exact allowed breaks: security, unspecified behavior, spec changes, bug fixes, additions to generated code, internal packages. Any other breaking change is announced "6 months in advance". The runtime stays compatible with generated code from a generator "no older than 1 year", enforced statically by `protoimpl.EnforceVersion` references in generated code — `~/Documents/GitHub/protobuf-go/README.md` §Compatibility (commit 2025-09-09)
- APIv2 (2020-03-02) took a new module path because it was not backward compatible; APIv1 is supported "indefinitely", and v1.4 is reimplemented on top of APIv2 — [go.dev blog](https://go.dev/blog/protobuf-apiv2)
- buffa's MSRV is set to the lowest toolchain the code actually compiles on, never more than ~12 months behind stable. Bumps are allowed in minor releases and recorded in the CHANGELOG. CI builds at the MSRV and at stable on every change — `~/Documents/GitHub/buffa/README.md` §Minimum supported Rust version
- prost CI runs `cargo-semver-checks` and `cargo minimal-versions` — `~/Documents/GitHub/prost/.github/workflows/ci.yml` lines 175–220
- In a 2023 study of the top 1000 crates, 172 crates (17.2%) had at least one release that broke semver, out of 464 of 14,389 releases (3.22%) — [predr.ag](https://predr.ag/blog/semver-violations-are-common-better-tooling-is-the-answer/)

### Inferences
- RustPB should write a protobuf-go-style compatibility section with three separate promises, each tied to a test: (1) wire/JSON equivalence with a named ScalaPB and scalapb-jsoniter version (test: golden corpus plus differential fuzzing; bumping the pinned ScalaPB version is itself a reviewed change that re-records goldens); (2) Rust API semver (test: cargo-semver-checks); (3) generated-code/runtime skew window (test: a compile check that pairs old generated code with the new runtime, plus an `EnforceVersion`-like const assertion in generated code).
- Since peers build against a protoc matrix (buffa: six protoc versions), RustPB, with its pure-Rust parser, should test descriptor equivalence against protoc (or protox) over the same .proto corpus.

### Gaps
- I did not fetch ScalaPB's own compatibility promise. It is needed to define what "ScalaPB-compatible" pins to.

## 5. Test-suite organization lessons

### Takeaway
Tier by cost, and make each tier a strict subset of a bigger one: SQLite's "veryquick" (~304.7k cases, minutes) before each check-in, the full suite (hours) and a soak run (~248.5 M) before release, and continuous fuzzing all the time. Every bug gets a test before it counts as fixed, and fuzz finds are kept as a replay corpus.

### Cited Findings
- Pre-check-in: "veryquick", ~304.7k cases, "a few minutes instead of a few hours". Pre-release: all tests on multiple platforms and compile configurations, plus the TH3 soak and Valgrind over veryquick and TH3 — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Regression rule: a bug "is not considered fixed until new test cases that would exhibit the bug have been added" — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- Fuzz-find replay: `fuzzcheck` replays a few thousand interesting cases during `make test` — [How SQLite Is Tested](https://www.sqlite.org/testing.html)
- prost runs a 300-second CIFuzz job per PR; OSS-Fuzz fuzzes continuously in the background — local `prost/.github/workflows/cifuzz.yml`; `protobuf-go/internal/fuzz/README.md`
- Traceability through names: conformance test names encode the spec area (e.g. `Recommended.Editions_Proto2.ProtobufInput.RejectInvalidUtf8.String.MapKey`), and failure-list entries carry the observed failure as a comment — `~/Documents/GitHub/protobuf/conformance/failure_list_rust_upb.txt`
- Expected changes in an upstream suite (BoGo) are absorbed through a shim config that maps error strings or marks options unimplemented, so they stay explicit and reviewable — [rustls bogo (docs.rs)](https://docs.rs/crate/rustls/0.23.12/source/tests/bogo.rs)
- The Best Practices badge requires a written test policy and evidence it was followed — [bestpractices.dev](https://www.bestpractices.dev/en/criteria/0)

### Inferences
- Suggested RustPB tiers:
  - **Fast loop** (seconds): unit tests, `y_/n_/i_` fixtures, the replay corpus, per-mapper properties at low case counts.
  - **Per PR** (minutes): the full golden corpus native and rust-js, conformance with strict lists, the 300 s CIFuzz, semver-checks, MSRV, Miri on `unsafe`, mutation testing of changed code (`cargo-mutants`).
  - **Nightly**: exhaustive prefix and bit-flip sweeps, re-recorded ScalaPB outcomes, long differential fuzzing, the full mutation run, coverage report.
  - **Release**: a generated TESTING.md refresh, the published results matrix, provenance.
- Traceability: give each design rule (§4.x, §5, §6.4 table rows) a stable ID and use it in fixture file names (`n_4.6_depth_bomb_group_101.bin`), so a script can list rules with no tests.
- Flaky-test policy: no primary source covered one in this pass. A strict failure list naturally turns flakiness into a visible red. The fallback is quarantine with an issue link and an expiry date (an inference, not sourced).

### Gaps
- No primary source on flaky-test policies of the exemplars (SQLite's page has none).
- Mutation-testing practice in Rust codec crates (cargo-mutants adoption, scores) was not researched.
