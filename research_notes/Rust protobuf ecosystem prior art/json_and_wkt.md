# Rust protobuf ecosystem: ProtoJSON and well-known types (prior art for RustPB)

Scope and method. Source was read from local shallow clones (`~/Documents/GitHub/<repo>`) at these commits (November 2026):
pbjson `3d254a2` (workspace 0.9.0; crates.io pbjson 0.9.0, 2025-12-09), prost-wkt `2f30fc6` (0.7.2, 2026-08-23), prost-reflect `2d0e184` (0.16.5, 2026-07-09), rust-protobuf `0118c79` (master = 4.0.0-alpha; crates.io protobuf-json-mapping 3.7.2, 2025-03-10), prost `894f4a7` (0.14.6, 2026-11-12), buffa `1047169` (0.9.2), prost-canonical-serde `f0cdf72` (0.1.3, 2026-02-06). Versions/dates/downloads come from the crates.io API (queried 2026-11-13). Line numbers refer to those commits; links point to the same paths upstream. Spec quotes are from [protobuf.dev ProtoJSON Format](https://protobuf.dev/programming-guides/json/). Items marked "inference" are my reading of the code, not confirmed by an issue, test or conformance run.

Crates at a glance:

| Crate | Mechanism | JSON conformance runner | Any | Notes |
|---|---|---|---|---|
| prost (0.14.6) | none: no JSON at all | runs binary only; testee answers "JSON input/output is not supported" | – | [issue #75 "JSON Mapping"](https://github.com/tokio-rs/prost/issues/75) open since 2018 |
| pbjson / pbjson-build / pbjson-types (0.9.0) | build-time codegen of hand-written-style `Serialize`/`Deserialize` impls from a `FileDescriptorSet`, over prost structs | none | not special-cased ([#2](https://github.com/influxdata/pbjson/issues/2) open since 2021) | ~17.8M recent downloads |
| prost-wkt (0.7.2) | `serde` **derive** via prost-build `type_attribute` + hand-written WKT types + `typetag`/`inventory` registry for Any | none | yes, via link-time registry | int64/bytes/enums/oneof not spec-shaped |
| prost-reflect (0.16.5) | runtime reflection: `DynamicMessage` + `DescriptorPool`, serde `Serialize`/`DeserializeSeed` | yes, empty failure list, `--enforce_recommended`, protobuf v25.4 | yes, via descriptor pool | ~27M recent downloads |
| protobuf-json-mapping (rust-protobuf 3.x) | runtime reflection over rust-protobuf `MessageDyn`, own tokenizer (no serde) | none found | not implemented (errors) | rust-protobuf README says "approaching end of life" |
| prost-canonical-serde (0.1.3) | proc-macro derive on prost types + build helper that injects `json_name` attributes | yes; 35 listed failures, 20 of them JSON (Any + enum aliases) | not implemented | new (2026), ~3.5k downloads |
| buffa (0.9.2) | own runtime + codegen; serde derive with `#[serde(with=...)]` helpers, hand-written visitors for oneof messages; also reflective `DynamicMessage` | yes, every known_failures list empty (protobuf v33.5) | yes, explicit `TypeRegistry` | ~5.3M recent downloads; used by `connectrpc` |

---

## How each crate generates or implements Serialize/Deserialize (field names, unknown fields, null, duplicates, scalars, bytes, enums, maps, oneof, defaults)

### Takeaway
There are three designs: (1) generated per-message serde impls driven by descriptors (pbjson, buffa, prost-canonical-serde), (2) reflection over descriptors at runtime (prost-reflect, protobuf-json-mapping, buffa's `DynamicMessage`), and (3) plain `serde` derive bolted onto prost structs (prost-wkt, ad-hoc `type_attribute`). Plain derive does not give ProtoJSON. Only the crates that run the upstream conformance suite (prost-reflect, buffa, prost-canonical-serde) handle the edge cases reliably. pbjson is the most widely used codegen approach, but it has long-open spec bugs: NaN/Infinity become `null`, Timestamp is written with `+00:00`, and Any/FieldMask are not special-cased.

### Cited Findings

**ProtoJSON spec baseline (what "correct" means)**
- Parsers must accept both the lowerCamelCase/json_name key and the original proto field name. Serializers should not emit `null`. A `null` parses as "field unset", except for `NullValue`. `null` is not allowed inside repeated fields. — [ProtoJSON spec](https://protobuf.dev/programming-guides/json/)
- Duplicate keys: serializers "MUST NOT emit duplicate keys (including multiple fields within the same `oneof`)". For parsers, "Last wins" behavior "should be used in all cases where possible", and otherwise "the parse must fail rather than selecting one to win arbitrarily". — [spec](https://protobuf.dev/programming-guides/json/)
- Unknown fields: parsers "should reject unknown fields by default but may provide an option to ignore unknown fields". — [spec](https://protobuf.dev/programming-guides/json/)
- Integers: int64/uint64 are emitted as strings. Parsers accept numbers or strings, and exponent notation (`1e2`) quoted or unquoted. `1.0` is valid for int32 but `1.5` is not. Out-of-range input fails. — [spec](https://protobuf.dev/programming-guides/json/)
- Floats: `"NaN"`, `"Infinity"` and `"-Infinity"` as strings. Bytes: emit standard base64 with padding, and accept standard or URL-safe, padded or not. Enums: emit the name (the first-declared name for aliases), and accept names or integers, with all alias names accepted. Map keys are always strings. — [spec](https://protobuf.dev/programming-guides/json/)
- Options the spec allows: always emit defaults, ignore unknown fields, use proto field names, and enums as integers. The spec notes that as of v25.x C++, Java and Python are nonconformant for proto2 `optional` when emitting defaults. — [spec](https://protobuf.dev/programming-guides/json/)

**prost (baseline)**
- prost has no JSON support. Its conformance testee returns "JSON input is not supported" and "JSON output is not supported" (`prost/conformance/src/main.rs:58-90`), and its failure list has only 2 binary unknown-field entries (`prost/conformance/failing_tests.txt`). — [prost conformance](https://github.com/tokio-rs/prost/tree/master/conformance)
- The JSON-mapping request has been open since 2018. Its thread points users to third-party crates, and a 2026 comment announces prost-canonical-serde. — [tokio-rs/prost#75](https://github.com/tokio-rs/prost/issues/75); related open requests: [#852 serde for prost-types](https://github.com/tokio-rs/prost/issues/852), [#1087 better serde for enumerations](https://github.com/tokio-rs/prost/issues/1087)

**pbjson-build (generated impls)**
- Inputs and outputs: it reads the `FileDescriptorSet` that prost-build writes and emits a separate `*.serde.rs` per package to `include!` beside prost's output. Users map `.google.protobuf` to `::pbjson_types` with `extern_path`. — `pbjson-build/src/lib.rs:1-70`, [docs](https://docs.rs/pbjson-build)
- Builder options, which are the complete configurability surface (`pbjson-build/src/lib.rs:96-210`): `exclude`, `retain_enum_prefix`, `extern_path`, `ignore_unknown_fields`, `btree_map`, `emit_fields` (emit defaults), `use_integers_for_enums`, `ignore_unknown_enum_variants` (unknown → enum default), and `preserve_proto_field_names`. — [pbjson-build lib.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-build/src/lib.rs)
- Field names: it serializes with `field.json_name()`, or the proto name if `preserve_proto_field_names` is set (`generator/message.rs:240-245`). Deserialization uses a generated `GeneratedField` enum whose `visit_str` matches `"jsonName" | "proto_name"` when they differ (`message.rs:673-760`). So both spellings are accepted. — [message.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-build/src/generator/message.rs)
- Unknown fields are an error by default (`serde::de::Error::unknown_field(value, FIELDS)`). With `ignore_unknown_fields` they map to `__SkipField__` and the value is consumed as `IgnoredAny` (`message.rs:588-612, 746-769`). The module doc says "unrecognised fields error on decode" (`message.rs:15`). — [message.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-build/src/generator/message.rs)
- Duplicate keys: every field is guarded by `if field__.is_some() { return Err(duplicate_field(..)) }`. The source comments that "this will report duplicate field if multiple value are specified for a one of" (`message.rs:837-846`). Repeating a key is therefore an error, not last-wins. The spec allows this but prefers last-wins. — [message.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-build/src/generator/message.rs)
- Null handling: explicit `null` is accepted for optional scalars, optional enums, messages, oneof members and wrapper types. For plain proto3 scalars, repeated fields and maps it is **rejected** (`"data did not match any variant"`, `"invalid type: null"`). The test suite asserts this deliberately: "Test primitives are not nullable" (`pbjson-test/src/lib.rs:714-781`). This deviates from the spec, which says null on any field means unset. — [pbjson-test](https://github.com/influxdata/pbjson/blob/main/pbjson-test/src/lib.rs)
- Numbers: numeric fields deserialize through `pbjson::private::NumberDeserialize<T>`, a `#[serde(untagged)]` enum of `Str(Cow<str>)` or `Number(T)`. Strings go through `T::from_str` (`pbjson/src/lib.rs:31-58`). int64/uint64 serialize via `ToString::to_string` as strings (`message.rs:350-356`). — [pbjson/src/lib.rs](https://github.com/influxdata/pbjson/blob/main/pbjson/src/lib.rs)
- Floats: there is no special-value handling on output. Generated code calls `serialize_field(.., &f64)`, and serde_json writes non-finite floats as `null` (`serde_json-1.0.154/src/ser.rs:169-175`). — [influxdata/pbjson#106 "float/double NaN/inf gets encoded to null in json"](https://github.com/influxdata/pbjson/issues/106), open since 2023-08
- Bytes: output uses standard base64 with padding. Input first tries STANDARD with "indifferent" padding, then retries URL_SAFE if the error is on `-` or `_` (`pbjson/src/lib.rs:73-100`). This was added for [#22 "Accept url-safe base64"](https://github.com/influxdata/pbjson/issues/22). — [pbjson/src/lib.rs](https://github.com/influxdata/pbjson/blob/main/pbjson/src/lib.rs)
- Enums: output is the name by default, or the number with `use_integers_for_enums`. Input accepts names or integers ([#46](https://github.com/influxdata/pbjson/issues/46)). Unknown names or numbers are an error unless `ignore_unknown_enum_variants` is set, which maps them to the enum default. Requests to preserve unknown values remain open ([#92](https://github.com/influxdata/pbjson/issues/92)). Alias names cannot be parsed ([#123](https://github.com/influxdata/pbjson/issues/123), open). — [pbjson issues](https://github.com/influxdata/pbjson/issues)
- Maps: integer keys deserialize via `NumberDeserialize`. Bool-keyed maps compile but fail at encode with `"key must be a string"` ([#27](https://github.com/influxdata/pbjson/issues/27), open since 2022). The generator `panic!`s on float or bytes keys (`message.rs:933-936`). — [message.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-build/src/generator/message.rs)
- Default omission: the predicate is `!is_empty()`, `!= 0`, or `!= 0.` for floats (`message.rs:78-100`). Because `-0.0 != 0.0` is false in Rust, -0.0 is treated as default and omitted (inference; buffa explicitly writes -0.0, see below).
- Conformance: pbjson has no conformance runner. Its tests are hand-written round-trips in `pbjson-test`. (Verified by listing the repo: there is no conformance directory.)

**prost-wkt (serde derive)**
- Usage is prost-build `.type_attribute(".", "#[derive(serde::Serialize,serde::Deserialize)]")`, plus `extern_path` for Any/Timestamp/Value to `prost_wkt_types`, plus `prost_wkt_build::add_serde(out, descriptor)`. Users must hand-add `#[serde(default)]` and `use serde::...` inside oneof modules. — [prost-wkt README](https://github.com/fdeantoni/prost-wkt)
- Consequences of plain derive, all open issues: int64 parses only from numbers ([#4](https://github.com/fdeantoni/prost-wkt/issues/4)); bytes serialize as an int array ([#10](https://github.com/fdeantoni/prost-wkt/issues/10)); enums serialize as i32 because prost stores them as `i32` ([#79](https://github.com/fdeantoni/prost-wkt/issues/79), 2025); missing fields fail without `#[serde(default)]` ([#5](https://github.com/fdeantoni/prost-wkt/issues/5)); oneofs serialize as externally tagged enums, with a community workaround using `#[serde(untagged)]`, strum and `flatten` ([#13](https://github.com/fdeantoni/prost-wkt/issues/13)). Field names are Rust snake_case unless the user adds `rename_all`. — [prost-wkt issues](https://github.com/fdeantoni/prost-wkt/issues)

**prost-reflect (reflection)**
- Options: `SerializeOptions { stringify_64_bit_integers (default true), use_enum_numbers, use_proto_field_name, skip_default_fields (default true) }` and `DeserializeOptions { deny_unknown_fields (default true) }`. They are passed explicitly via `serialize_with_options` and `deserialize_with_options`, which avoids ambient state (`prost-reflect/src/dynamic/serde/mod.rs:14-31, 180-245`). — [prost-reflect serde/mod.rs](https://github.com/andrewhickman/prost-reflect/blob/main/prost-reflect/src/dynamic/serde/mod.rs)
- Field lookup: `get_field_by_json_name(key).or_else(|| get_field_by_name(key))`, then extensions by json name. Unknown keys error with `unrecognized field name` or are skipped with `IgnoredAny` (`de/kind.rs:567-600`). — [de/kind.rs](https://github.com/andrewhickman/prost-reflect/blob/main/prost-reflect/src/dynamic/serde/de/kind.rs)
- Oneofs: an error is raised only if a second oneof member is **non-null** (`de/kind.rs:572-585`). The 0.15.1 changelog says this was done "to better conform to the specification". For non-oneof fields, a repeated key overwrites (last wins) per the `set_field` call (inference from code). — [CHANGELOG 0.15.1](https://github.com/andrewhickman/prost-reflect/blob/main/CHANGELOG.md)
- `deny_unknown_fields=false` also makes unknown enum names be ignored, which "matches the behaviour of this flag in other implementations" (CHANGELOG 0.15.1; `de/kind.rs:611-625`). Since [#157](https://github.com/andrewhickman/prost-reflect/issues/183), an explicit `null` leaves `has_field` false. — [prost-reflect#183](https://github.com/andrewhickman/prost-reflect/issues/183)
- Floats: output is `"Infinity"`, `"-Infinity"` or `"NaN"` (`ser/mod.rs:120-142`). Input accepts those strings, range-checks f32, and range-checks integral targets parsed from floats (`de/kind.rs:185-450`). Exact round-trip of doubles needs serde_json's `float_roundtrip` feature ([#129](https://github.com/andrewhickman/prost-reflect/issues/129)).
- Bytes: output is `BASE64_STANDARD`. Input tries standard, then URL_SAFE (`ser/mod.rs:3`, `de/kind.rs:505-532`).
- json_name: when the descriptor's `json_name` is unset it is filled with the camel-cased name (CHANGELOG 0.10.2). Arbitrary casing such as kebab-case is not offered; users are pointed to `use_proto_field_name` or setting `json_name` in the .proto ([#166](https://github.com/andrewhickman/prost-reflect/issues/166)).

**protobuf-json-mapping (rust-protobuf, reflection, no serde)**
- `PrintOptions { enum_values_int, proto_field_name, always_output_default_values }` and `ParseOptions { ignore_unknown_fields }`, with `_future_options: ()` to block exhaustive construction (`protobuf-json-mapping/src/print.rs:556-568`, `parse.rs:852-860`). — [rust-protobuf](https://github.com/stepancheg/rust-protobuf/tree/master/protobuf-json-mapping)
- Parse accepts both names via `field_by_name_or_json_name`, with a comment citing the spec (`parse.rs:668-684`). It has its own tokenizer rather than serde.
- int64 and uint64 are printed quoted (`print.rs:130-142`). Floats print `"NaN"`/`"Infinity"`/`"-Infinity"`, and finite values use Rust `{:?}` (`print.rs:72-88`). The base64 decode table maps both `+`/`-` and `/`/`_`, so it accepts mixed alphabets, and ignores `=` and newlines (`base64.rs:145-152`).
- rust-protobuf's README states it "is approaching end of life" in favor of the official protobuf v4 crate. — [rust-protobuf README](https://github.com/stepancheg/rust-protobuf)

**prost-canonical-serde (derive, 2026)**
- A build helper `add_json_name_attributes(config, &fds)` injects `#[prost_canonical_serde(proto_name = "...", json_name = "...")]` field attributes through prost-build, because a derive cannot see descriptors. The derive then matches `json | proto` on parse (`prost-canonical-serde-build/src/lib.rs:22-68`, `prost-canonical-serde-derive/src/lib.rs:400-431`). — [prost-canonical-serde](https://github.com/xyzzyz/prost-canonical-serde)

**buffa (codegen + reflection, 2025-2026)**
- Uses `#[derive(::serde::Serialize, ::serde::Deserialize)]` with `#[serde(with=...)]` helper modules for int64, float and bytes, plus `skip_serializing_if` predicates. Messages with a oneof or with extension ranges get hand-written `Deserialize`. The guide states that "the canonical protobuf JSON mapping is non-trivial and cannot be satisfied by plain derive alone". — [buffa DESIGN.md](https://github.com/anthropics/buffa/blob/main/DESIGN.md); [guide JSON section](https://github.com/anthropics/buffa/blob/main/docs/guide.md)
- Unknown JSON keys are **ignored by default**, unlike the spec default. `deny_unknown_json_fields(true)` and the path-scoped `deny_unknown_json_fields_in` are codegen-time switches that emit `#[serde(deny_unknown_fields)]`. The error lists both spellings: `expected one of maxItems, max_items, ...`. The guide explains that a codegen switch was chosen "because serde's derive has no runtime hook". It acknowledges that this lets a library crate fix strictness for its consumers, with a runtime override tracked in [buffa#444](https://github.com/anthropics/buffa/issues/444). — [guide "Unknown fields in JSON"](https://github.com/anthropics/buffa/blob/main/docs/guide.md)
- Runtime parse options (`ignore_unknown_enum_values`, `strict_extension_keys`, `strict_any_type_urls`) travel through **ambient state**: a thread-local `with_json_parse_options(&opts, || serde_json::from_str(..))` with a restore-on-drop guard under `std`, and a process-wide set-once `AtomicPtr` under `no_std` (`buffa/src/json.rs:18-62, 200-240`). The module docs include a per-field-kind table (implicit, optional, repeated/map, oneof, extension) giving exactly what "ignore unknown enum value" does in each. — [buffa json.rs](https://github.com/anthropics/buffa/blob/main/buffa/src/json.rs)
- Defaults: -0.0 is written, because only +0.0 is treated as the default. Floats accept `"NaN"`, `"Infinity"`, `"-Infinity"` or null (`json_helpers.rs:1218-1275`). Base64 decoding is lenient, accepting trailing bits and indifferent padding on both alphabets (`json_helpers.rs:40-55`). Size hints are clamped to 4096 against hostile deserializers (`json_helpers.rs:57-70`). — [json_helpers.rs](https://github.com/anthropics/buffa/blob/main/buffa/src/json_helpers.rs)
- Documented caveats: JSON parsing is not bounded by the binary decode limits ([#330](https://github.com/anthropics/buffa/issues/330)). serde_json's `arbitrary_precision` feature, if enabled anywhere in the build, breaks number parsing ([#482](https://github.com/anthropics/buffa/issues/482)). Unquoted f32 values are double-rounded (f64 then f32), as in C++, whereas Go rounds once ([#647](https://github.com/anthropics/buffa/issues/647)). Untrusted `serde_json::Value` meets `raw_value`-feature key smuggling, which buffa avoids with a `buffered` deserializer. — [guide](https://github.com/anthropics/buffa/blob/main/docs/guide.md)

### Inferences
- pbjson's "error on duplicate key" and "reject null for scalars" choices come from serde-derive conventions, not the spec. A lenient-parse target such as scalapb-jsoniter probably needs last-wins and null-as-unset. RustPB should decide each explicitly rather than inherit serde defaults.
- pbjson's `NumberDeserialize` uses `#[serde(untagged)]`, which buffers into serde's private `Content` before trying each variant. That costs speed and turns a type error into "data did not match any variant". In pbjson, a JSON number written with an exponent or as `1.0`, for an int64 field, likely fails, because `Number(i64)` cannot take an f64. The spec says such input should be accepted. I could not verify this without a conformance run.
- Because serde's `Deserialize` has no context parameter, each serde-based design must either bake options into codegen (pbjson's Builder, buffa's `deny_unknown_json_fields`) or use ambient state (buffa's thread-local). Reflection designs (prost-reflect, protobuf-json-mapping) pass options explicitly. For RustPB, whose generated code must also compile to JS, codegen-time options are the simpler fit. Thread-local or global ambient state needs an equivalent in the JS target.
- The serde ecosystem's float formatter matters for byte-identical output. serde_json 1.0.154 now formats f64 with `zmij` (`ser.rs:1716-1722`), where older versions used `ryu`. It emits `1.0` for 1.0, which differs from Go-style `1`. An "identical to the Scala codec" requirement should pin RustPB's own float formatting rather than delegate to whatever serde_json version is in the build.

### Gaps
- I did not run pbjson against the conformance suite: building was not permitted in this session, and no published pbjson conformance numbers were found. The spec deviations above come from code and issues, not a failure list.
- I did not find how protobuf-json-mapping 3.7.2 (crates.io) differs from master (4.0.0-alpha) in the JSON module. The analysis is of master.

---

## Well-known types: who gets Timestamp, Duration, FieldMask, Struct/Value/ListValue, wrappers, Any, Empty right

### Takeaway
prost-reflect and buffa implement every WKT special form, with range checks and Any via a registry or descriptor pool. pbjson-types implements Timestamp, Duration, Struct/Value/ListValue and the wrappers. It has spec bugs in Timestamp (`+00:00` instead of `Z`) and Value (`null`), and does not special-case Any or FieldMask. prost-wkt's Duration has sign and precision bugs. protobuf-json-mapping refuses Any entirely.

### Cited Findings
- **Spec**: Timestamp output is "Z-normalized with 0, 3, 6 or 9 fractional digits", and other offsets are accepted on input. Duration output has 0/3/6/9 fractional digits plus "s". FieldMask is written like `"f.fooBar,h"`. Any adds `"@type"`, and special WKTs inside an Any go under `"value"`. Wrappers use the primitive's form, with null allowed. Empty is `{}`. — [spec](https://protobuf.dev/programming-guides/json/)
- **pbjson-types**: its build excludes Duration, Timestamp, Value, Struct, ListValue, NullValue and the 9 wrappers from codegen, and hand-writes those (`pbjson-types/build.rs:24-45`). Everything else, including **Any and FieldMask**, is generated as an ordinary message. The crate doc warns: "Coverage of all types is currently incomplete, some may have non-compliant implementations" (`pbjson-types/src/lib.rs:5-6`). — [pbjson-types](https://github.com/influxdata/pbjson/tree/main/pbjson-types)
  - Timestamp serializes with chrono `to_rfc3339()` (`timestamp.rs:30-37`). The crate's own test asserts output `"2016-11-08T16:07:09+00:00"` (`timestamp.rs:77-80`): not Z-normalized. There is no year 0001-9999 range check. Pre-epoch conversion issues are discussed in [#95](https://github.com/influxdata/pbjson/issues/95). — [timestamp.rs](https://github.com/influxdata/pbjson/blob/main/pbjson-types/src/timestamp.rs)
  - Duration: output is correct (0/3/6/9 digits, `-0.xxx` for negative sub-second, sign-consistency check). Parsing has no ±315,576,000,000 s range check (`duration.rs:24-110`).
  - Any: it serializes as `{"typeUrl": "...", "value": "<base64>"}`, as shown in a user report in [#2](https://github.com/influxdata/pbjson/issues/2) (open since 2021-09). A fork with "Envoy fixes" for Any was offered in [#119](https://github.com/influxdata/pbjson/issues/119) and is not merged.
  - FieldMask: [#4 "Correct Serialization of FieldMask"](https://github.com/influxdata/pbjson/issues/4) is closed, but no FieldMask special case exists in pbjson-types/src or pbjson-build at HEAD (grep found none), so it is generated as `{"paths":[...]}` (inference).
  - Value: JSON `null` deserializes as `Value { kind: None }` rather than `NullValue` ([#100](https://github.com/influxdata/pbjson/issues/100), open). Integers over 16 digits in a `Value` fail with "out of range integral type conversion" ([#139](https://github.com/influxdata/pbjson/issues/139), open).
- **prost-types** (no serde) has `Display`/`FromStr` for Timestamp and Duration. Output is `Z` with 0/3/6/9 digits. Years above 9999 are written `+YYYYY`. The parser accepts lowercase `t`/`z` and a space separator (`prost-types/src/datetime.rs:67-96, 279-290, 508-509`; `duration.rs:108-128`). — [prost-types](https://github.com/tokio-rs/prost/tree/master/prost-types/src)
- **prost-reflect**: it special-cases all 17 WKTs, including Empty and FieldMask (`serde/mod.rs:258-279`). Timestamp is checked against seconds in [-62135596800, 253402300799], and Duration against |s| ≤ 315,576,000,000 with |nanos| ≤ 999,999,999 (`mod.rs:251-300`). Formatting delegates to prost-types `Display`. FieldMask uses snake↔camel conversion that **errors if the round-trip is impossible** (`serde/case.rs:1-40`). Any is resolved through the message's `DescriptorPool`. Well-known payloads go under `"value"`, and others are flattened next to `"@type"` (`ser/wkt.rs:55-90`). The Any parser buffers entries until `@type` is found, so `@type` can appear anywhere (`de/wkt.rs:45-104`). — [prost-reflect ser/wkt.rs](https://github.com/andrewhickman/prost-reflect/blob/main/prost-reflect/src/dynamic/serde/ser/wkt.rs)
- **prost-wkt**: Any uses `#[typetag::serde(tag = "@type")]` over a `MessageSerde` trait object. Every generated message is registered with `typetag` plus an `inventory::submit!` decoder entry (`wkt/src/lib.rs:1-95`; `wkt-build/src/lib.rs:54-105`). An Any that cannot be unpacked falls back to `{"@type", "value": <bytes>}` (`wkt-types/src/pbany.rs:149-164`).
  - Duration serializes as `format!("{:.9}s", seconds as f64 + nanos as f64 / 1e9)` (`wkt-types/src/pbtime/duration.rs:204-215`). That always gives 9 digits (the code admits it "*always* serializes with 9 fractional digits") and goes through f64, which loses nanosecond precision above about 2^53 ns, roughly 104 days (inference from the arithmetic). It previously serialized as an object ([#60](https://github.com/fdeantoni/prost-wkt/issues/60), closed).
  - Duration parse: for input `-1.5s`, seconds are parsed as `-1` and nanos as `+500000000` (`duration.rs:236-262`), giving -0.5 s. `-0.5s` parses as +0.5 s. Signs are never reconciled (inference from code; no issue found).
  - Missing WKTs are reported in [#76](https://github.com/fdeantoni/prost-wkt/issues/76). Empty and FieldMask use plain serde derive (`wkt-types/build.rs:36-44`), so FieldMask is `{"paths":[...]}`.
- **protobuf-json-mapping**: Any is unsupported in both directions (`AnyPrintingIsNotImplemented`, `AnyParsingIsNotImplemented`; `print.rs:42, 247-250`; `parse.rs:75, 818-820`). Duration always prints 9 digits, `"{sign}{secs}.{nanos:09}s"`, with the sign taken only from `seconds >= 0` (`print.rs:218-230`). So `{seconds:0, nanos:-500000000}` would print `"0.500000000s"` (inference from code). Timestamp prints 9 fractional digits by default (`rfc_3339.rs:439-447`).
- **buffa-types**: WKTs are generated structs with hand-written `*_ext.rs` for the JSON forms. Shared formatting primitives in `buffa::json_helpers::wkt` are used by both the typed serde impls and the reflective codec, "so the two paths can't drift" (`json_helpers.rs` module docs). Any needs an explicit `TypeRegistry`: register each package with `register_types` and the WKTs with `register_wkt_types`, then install it with `set_type_registry`. Lookup is by exact URL first, then by the name after the last `/`. An unregistered type is written as `{"@type": "...", "value": "<base64>"}`, which the guide calls "specific to buffa". Bare `@type` names without `/` are accepted unless `strict_any_type_urls` is set; the guide notes Go and Python accept them while C++ and Java reject them. — [buffa guide "Any"](https://github.com/anthropics/buffa/blob/main/docs/guide.md); [any_registry.rs](https://github.com/anthropics/buffa/blob/main/buffa/src/any_registry.rs)
- **prost-canonical-serde**: handles prost-types WKTs except Any. Every `Required.Proto3.JsonInput.Any*` conformance case is in its failure list. — [failure_list_rust_cc.txt](https://github.com/xyzzyz/prost-canonical-serde/blob/main/prost-canonical-serde-conformance/conformance/failure_list_rust_cc.txt)

### Inferences
- Any is the hardest WKT for a code generator: JSON Any requires a type registry. There are three approaches:
  - prost-wkt's link-time `inventory` + `typetag` relies on platform linker sections or constructors. It is not portable to a Rust-to-JS compiler.
  - prost-reflect's descriptor pool requires shipping descriptors.
  - buffa's explicit, user-installed registry populated by generated `register_types` functions works anywhere.
  The explicit registry suits RustPB best (no linker magic; the same code compiles to JS).
- Sharing one WKT formatting core between the typed and reflective paths, as buffa does, is a good pattern against drift. RustPB could likewise share one core between the Rust and JS outputs, since both come from the same Rust source.

### Gaps
- No issue or test was found confirming the prost-wkt Duration sign bug or the rust-protobuf zero-seconds negative-nanos bug. Both come from reading the code.

---

## Conformance suite results for JSON

### Takeaway
Only three Rust crates run the upstream conformance runner with JSON: buffa (0 failures, protobuf v33.5), prost-reflect (0 failures with `--enforce_recommended`, protobuf v25.4) and prost-canonical-serde (20 JSON failures: Any and enum aliases). pbjson, prost-wkt and protobuf-json-mapping publish no JSON conformance results.

### Cited Findings
- buffa: "passes the protobuf binary and JSON conformance test suite (v33.5, editions up to 2024). Both std and no_std builds pass the full suite including JSON." Every `conformance/known_failures*.txt` has 0 entries at HEAD: std, nostd, reflect, table, view, view_json, view_vtable, lazy and text. Several of these are alternate routes, e.g. view → JSON and DynamicMessage → JSON, so each codec path is checked separately. — [buffa README](https://github.com/anthropics/buffa/blob/main/README.md); [conformance/](https://github.com/anthropics/buffa/tree/main/conformance)
- The buffa author's post cautions that a green conformance run is "necessary but not sufficient": it does not exercise adversarial resource bounds, and four issues were found in a security review despite passing. — [Zero-copy protobuf and ConnectRPC for Rust (DEV)](https://dev.to/iainmcgin/zero-copy-protobuf-and-connectrpc-for-rust-1m3e)
- prost-reflect: `failure_list.txt` and `text_format_failure_list.txt` are both 0 bytes. The test runs `conformance_test_runner --enforce_recommended --failure_list failure_list.txt` (`prost-reflect-conformance-tests/tests/conformance.rs:25-29`). The protobuf submodule is pinned to `e915ce24`, which the GitHub API identifies as "Updating version.json and repo version numbers to: 25.4" (2024-07-24). The `JsonIgnoreUnknownParsingTest` category is handled by setting `deny_unknown_fields(false)` (`src/main.rs:76-80`). — [prost-reflect conformance tests](https://github.com/andrewhickman/prost-reflect/tree/main/prost-reflect-conformance-tests)
- prost-canonical-serde: 35 listed failures. 15 are binary (prost's unknown-field and MessageSet limits). 20 are JSON: 14 `Any*` cases (Any, AnyNested, AnyUnorderedTypeTag, AnyWithDuration/FieldMask/Int32ValueWrapper/NoType/Struct/Timestamp/ValueForInteger/ValueForJsonObject, each in JsonOutput and/or ProtobufOutput form) and 6 `EnumFieldWithAlias{DifferentCase,LowerCase,UseAlias}` cases. The README attributes these to prost's enum parsing not accepting aliases. — [CONFORMANCE.md and failure list](https://github.com/xyzzyz/prost-canonical-serde)
- prost: binary only; JSON requests are answered "not supported" (`prost/conformance/src/main.rs:58-90`). — [prost conformance](https://github.com/tokio-rs/prost/tree/master/conformance)

### Inferences
- The enum-alias conformance cases trip derive and codegen designs that reuse a Rust enum's `FromStr` (pbjson [#123](https://github.com/influxdata/pbjson/issues/123), prost-canonical-serde). RustPB should generate a name → value table that includes every alias, and serialize the first declared name.
- Conformance covers spec behavior, not "match implementation X". RustPB's goal of byte-identical output to the Scala codec needs its own golden-file and differential tests against that codec, in addition to the upstream suite.

### Gaps
- No published conformance numbers were found for pbjson, prost-wkt or protobuf-json-mapping.

---

## Known bugs, edge cases and design mistakes (from issue trackers)

### Takeaway
The recurring failures are: plain serde derive (wrong shapes everywhere); non-finite floats becoming `null` through serde_json; Timestamp offsets that are not Z-normalized; Any needing a registry; unknown or alias enum values; forward compatibility vs strict unknown-field rejection; and options that cannot be changed at runtime.

### Cited Findings
- pbjson open issues: NaN/Inf → null [#106](https://github.com/influxdata/pbjson/issues/106); Any [#2](https://github.com/influxdata/pbjson/issues/2); bool map keys [#27](https://github.com/influxdata/pbjson/issues/27); enum aliases [#123](https://github.com/influxdata/pbjson/issues/123); unknown enum values ("It really destroys the forward-compatibility of protobufs", comment on [#92](https://github.com/influxdata/pbjson/issues/92)); Value null [#100](https://github.com/influxdata/pbjson/issues/100); large integers in Value [#139](https://github.com/influxdata/pbjson/issues/139); bytes from an int array requested [#121](https://github.com/influxdata/pbjson/issues/121); no_std [#135](https://github.com/influxdata/pbjson/issues/135); generated code triggering clippy lints [#157](https://github.com/influxdata/pbjson/issues/157), [#137](https://github.com/influxdata/pbjson/issues/137). Closed fixes include explicit null support [#73](https://github.com/influxdata/pbjson/issues/73), integer enums [#46](https://github.com/influxdata/pbjson/issues/46), URL-safe base64 [#22](https://github.com/influxdata/pbjson/issues/22), an emit-defaults option [#82](https://github.com/influxdata/pbjson/issues/82), and the enum prefix-stripping case [#116](https://github.com/influxdata/pbjson/issues/116). — [pbjson issues](https://github.com/influxdata/pbjson/issues)
- prost-wkt open issues: int64 as string [#4](https://github.com/fdeantoni/prost-wkt/issues/4), missing fields [#5](https://github.com/fdeantoni/prost-wkt/issues/5), bytes [#10](https://github.com/fdeantoni/prost-wkt/issues/10), oneof [#13](https://github.com/fdeantoni/prost-wkt/issues/13), enums as strings [#79](https://github.com/fdeantoni/prost-wkt/issues/79), missing WKTs [#76](https://github.com/fdeantoni/prost-wkt/issues/76), generated `google.protobuf.rs` growing on each build [#63](https://github.com/fdeantoni/prost-wkt/issues/63). — [prost-wkt issues](https://github.com/fdeantoni/prost-wkt/issues)
- prost-reflect: explicit null previously set `has_field` true, fixed in #157 ([#183](https://github.com/andrewhickman/prost-reflect/issues/183)); float round-trip depends on serde_json `float_roundtrip` ([#129](https://github.com/andrewhickman/prost-reflect/issues/129)); custom key casing such as kebab-case is requested ([#166](https://github.com/andrewhickman/prost-reflect/issues/166)). — [prost-reflect issues](https://github.com/andrewhickman/prost-reflect/issues)
- buffa's own documented edge cases: `arbitrary_precision` incompatibility [#482](https://github.com/anthropics/buffa/issues/482), f32 double rounding [#647](https://github.com/anthropics/buffa/issues/647), JSON input not bounded [#330](https://github.com/anthropics/buffa/issues/330), no runtime unknown-field override [#444](https://github.com/anthropics/buffa/issues/444). — [buffa guide](https://github.com/anthropics/buffa/blob/main/docs/guide.md)

### Inferences
- Design mistake to avoid: using serde_json's default `f64` path for proto floats. NaN/Inf silently become `null` and output does not fail. Any serde-based RustPB must route floats through its own serializer.
- Design mistake to avoid: delegating Timestamp formatting to a general date library (chrono `to_rfc3339`). Its output conventions (`+00:00`, digit counts) are not ProtoJSON's.

### Gaps
- Bug reports for buffa beyond its own guide were not surveyed in depth.

---

## Performance, code size and compile time

### Takeaway
There is little independent data. buffa publishes cross-implementation JSON benchmarks against prost+pbjson: pbjson encode is 6-33% faster than buffa, while buffa decode is up to about 2.4× faster than pbjson on some messages. Both are about 5-10× faster than Go's protojson. No code-size or compile-time measurements for generated JSON impls were found.

### Cited Findings
- buffa README JSON encode, MiB/s, buffa / prost+pbjson 0.7 / Go protojson v1.36.6: ApiResponse 521 / 589 / 70; LogRecord 697 / 882 / 85; AnalyticsEvent 505 / 533 / 33; GoogleMessage1 571 / 674 / 73; MediaFrame 702 / 937 / 235. JSON decode: ApiResponse 492 / 205 / 40; LogRecord 530 / 424 / 63; AnalyticsEvent 169 / 152 / 25; GoogleMessage1 413 / 171 / 41; MediaFrame 1,235 / 1,218 / 215. The README states that the Google protobuf-v4 Rust crate "does not provide a JSON codec". — [buffa README benchmarks](https://github.com/anthropics/buffa/blob/main/README.md)
- pbjson emits a full hand-rolled `Serialize` (with `serialize_struct` and a `len` count of non-default fields) and a full `Deserialize` per message. The `Deserialize` includes a `GeneratedField` enum, a field-name `Visitor`, a message `Visitor` with one `Option` local per field, and per-field duplicate checks (`pbjson-build/src/generator/message.rs`, `generator.rs:37-81`). — [pbjson-build](https://github.com/influxdata/pbjson/tree/main/pbjson-build/src)

### Inferences
- pbjson's slower decode is plausibly due to `#[serde(untagged)]` buffering in `NumberDeserialize`, which applies to every numeric field. Its faster encode may come from its simple `serialize_struct` path; buffa supports more features (extensions, unknown fields). Neither explanation is benchmarked.
- Code size: a pbjson-style generator emits roughly two impls plus two visitors per message. Measuring this was blocked, since building pbjson-types was not permitted in this session. For a Rust-to-JS target, generic serde monomorphization per message is the main size risk, which favors small shared runtime helpers (buffa's `#[serde(with)]` modules) over inline per-field logic.

### Gaps
- No measurements of generated-code lines, binary size or compile time for pbjson, buffa or prost-canonical-serde were found or produced.

---

## Good ideas worth copying, especially for configurable JSON behavior to match another implementation exactly

### Takeaway
Copy these: descriptor-driven codegen of explicit serde impls (pbjson, buffa); a single named options struct with explicit documented defaults per spec option (prost-reflect, protobuf-json-mapping); buffa's per-field-kind semantics table for leniency options; an explicit Any registry; shared WKT primitives; and running the upstream conformance suite through every code path. To match a specific other implementation, make each divergence point a named, documented option rather than an emergent serde behavior.

### Cited Findings
- Option surfaces to model on:
  - pbjson Builder: `ignore_unknown_fields`, `emit_fields`, `use_integers_for_enums`, `ignore_unknown_enum_variants`, `preserve_proto_field_names`, `retain_enum_prefix`, `btree_map`.
  - prost-reflect: `stringify_64_bit_integers`, `use_enum_numbers`, `use_proto_field_name`, `skip_default_fields`, `deny_unknown_fields`.
  - protobuf-json-mapping: `_future_options: ()` for forward-compatible option structs.
  - buffa: `ignore_unknown_enum_values`, `strict_extension_keys`, `strict_any_type_urls` (runtime), plus `deny_unknown_json_fields[_in]` (codegen, path-scoped).
  — sources as cited above
- buffa documents each leniency choice against other implementations by file and line, e.g. "protobuf-go (`protojson/decode.go:175`) and protobuf-es (`from-json.ts:251`) both error on unregistered extension keys". It also notes where Go, Python, C++ and Java disagree, for bare `@type` names and f32 rounding. — [buffa json.rs](https://github.com/anthropics/buffa/blob/main/buffa/src/json.rs); [guide](https://github.com/anthropics/buffa/blob/main/docs/guide.md)
- prost-canonical-serde injects descriptor facts (`json_name`) into derive attributes through the build step, so a derive can stay descriptor-accurate. — [prost-canonical-serde-build](https://github.com/xyzzyz/prost-canonical-serde/tree/main/prost-canonical-serde-build)
- connectrpc, nominated as the canonical Rust ConnectRPC implementation, has a default `json` feature. Its early 0.1.0 depended on prost 0.14 + pbjson 0.8; later versions use buffa's `json` feature (connectrpc 0.9.1, 2026-09-21). — [connectrpc docs.rs](https://docs.rs/connectrpc); [connectrpc 0.1.0 README](https://docs.rs/crate/connectrpc/0.1.0/source/README.md); [ConnectRPC RFC 007](https://connectrpc.com/docs/governance/rfc/007-rust-implementation/)

### Inferences
Recommendations for RustPB (inference):
1. Generate explicit `Serialize`/`Deserialize` impls from descriptors; do not rely on derive. Every serde default (`null` = error, duplicate = error, NaN = null, deny/allow unknown) conflicts with some ProtoJSON or Scala-codec rule.
2. Make every divergence point an explicit option, defaulting to the Scala codec's behavior:
   - key spelling (json_name or proto name);
   - accept-both on parse;
   - unknown fields (ignore or error);
   - null on scalars, repeated and maps (unset or error);
   - duplicate keys (last-wins or error);
   - oneof with several non-null members;
   - int64 output (string or number) and input (accept number, string, exponent, `1.0`);
   - float output text (NaN/Infinity spelling, `1.0` vs `1`, exponent style, -0.0);
   - bytes alphabet and padding on output and accepted on input;
   - enum output (name or number) and unknown enum on input (error, default, drop, keep number);
   - map key forms;
   - default emission;
   - Timestamp and Duration digit policy and range checks;
   - Any form for unregistered types.
3. Prefer codegen-time options or explicitly passed options over thread-local ambient state. buffa needed a thread-local for `std` and a global for `no_std`, and a Rust-to-JS output would need its own equivalent.
4. Use an explicit Any registry populated by generated `register_types` functions, not `inventory`/`typetag`.
5. Write a dedicated float formatter rather than serde_json's, so output does not change across serde_json versions (ryu → zmij).
6. Test against the upstream conformance runner and, separately, differentially against the Scala codec's actual outputs.

### Gaps
- I did not find a Rust crate whose stated goal is byte-exact matching of another language's JSON codec. All aim at spec or "canonical" behavior. Exact-match tooling (golden corpora, differential fuzzing) has no Rust prior art in this space as far as these sources show.
- grpc-gateway-style Rust transcoders were not surveyed beyond connectrpc. tonic itself has no JSON codec. axum-connect is reported to use pbjson, but I did not verify this in its source.
