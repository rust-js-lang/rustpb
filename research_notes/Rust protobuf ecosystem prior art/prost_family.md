# The prost family as prior art for RustPB

Sources read locally (shallow clones made 2026-11-13, all at their default branch HEAD):
- prost workspace (prost, prost-derive, prost-build, prost-types, conformance): `~/Documents/GitHub/prost` at commit 894f4a7 "chore: Release version 0.14.6 (#1476)". Line numbers below refer to this tree; the GitHub links point to tag `v0.14.6`.
- protox 0.10.0: `~/Documents/GitHub/protox` (last commit 2026-10-07)
- prost-reflect 0.16.5: `~/Documents/GitHub/prost-reflect`
- protoc-gen-prost 0.5.0 (+ -serde 0.4.0, -crate, tonic): `~/Documents/GitHub/protoc-gen-prost` (last commit 2026-07-08)
- tonic 0.14.9 (tonic-build, tonic-prost-build, plus grpc-protobuf): `~/Documents/GitHub/tonic`
Issues, PRs and release notes were read through the `gh` CLI (`gh issue view`, `gh pr view`, `gh release view`). Current prost release: **0.14.6, 2026-11-12** ([releases](https://github.com/tokio-rs/prost/releases)).

---

## 1. Codegen pipeline: protoc vs protox, build.rs vs checked-in, module layout, naming

### Takeaway
prost-build is a library run from `build.rs` that turns a `FileDescriptorSet` (from `protoc`, from protox, or supplied by the user) into one `.rs` file per protobuf package in `OUT_DIR`. The user `include!`s those files. Types are named with heck-based case conversion, keywords become raw identifiers, enum value prefixes are stripped by default, and nested types go into a snake_case module named after the parent message. protox (pure Rust) and protoc-gen-prost (a protoc/buf plugin for checked-in code) are add-ons to the same `compile_fds`/`generate` entry points.

### Cited Findings
- **protoc dependency.** Since prost-build 0.11, `protoc` must be on PATH or set through `PROTOC`. prost no longer bundles or builds it ("Prost will no longer provide bundled `protoc` or attempt to compile `protoc` for users"). Before that, 0.10 compiled protoc from bundled source and ≤0.9 bundled binaries — [prost README "protoc"](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md), [v0.11.0 release](https://github.com/tokio-rs/prost/releases/tag/v0.11.0), [v0.10.0 release](https://github.com/tokio-rs/prost/releases/tag/v0.10.0)
- **How protoc is invoked.** `Config::load_fds` runs `protoc --include_imports [--include_source_info] -o <tmp>/prost-descriptor-set -I... <protos>`, then decodes the descriptor set with prost itself. `skip_protoc_run` + `file_descriptor_set_path` lets you skip protoc — `prost-build/src/config.rs:918-1000` ([link](https://github.com/tokio-rs/prost/blob/v0.14.6/prost-build/src/config.rs#L918))
- **Descriptor-in entry points.** `compile_fds(FileDescriptorSet)` (config.rs:826) and `generate(...)` (config.rs:1113) skip protoc entirely. "Extract file descriptor loading from compile_protos() (#1067)" landed in 0.13.2 — [CHANGELOG 0.13.2](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **protox** is "An implementation of the protobuf compiler in rust, intended for use as a library with crates such as `prost-build` to avoid needing to build `protoc`". Usage: `prost_build::compile_fds(protox::compile(["root.proto"], ["."])?)`. It reports errors with `miette` source spans — [protox README](https://github.com/andrewhickman/protox/blob/main/README.md)
- protox links and validates through prost-reflect's `DescriptorPool` (`protox/src/compile/mod.rs:8,49`). Its test suite compares its output with real protoc output serialized to YAML (`protox/tests/compare.rs`) — local source
- protox's parser accepts only `syntax = "proto2"|"proto3"` and otherwise emits `UnknownSyntax` (`protox-parse/src/parse/mod.rs:153-156`, `error.rs:58`). Neither protox-parse nor protox mentions editions anywhere in its sources — local grep
- protox 0.10.0 (2026-10-07) "Fixed a stack overflow vulnerability when parsing deeply nested proto files. The maximum allowed nesting depth is now limited to 32 (#106)" — [protox CHANGELOG](https://github.com/andrewhickman/protox/blob/main/CHANGELOG.md)
- **Output layout.** One file per package, named after the package path (`Module::to_file_name_or`, `prost-build/src/module.rs:51-66`). Users write `include!(concat!(env!("OUT_DIR"), "/foo.bar.rs"))` inside hand-written `pub mod` nests. Alternatively, `Config::include_file("_includes.rs")` generates the `pub mod` tree for you (config.rs:764-795). Since 0.13.0, generated files and the include file are rewritten only when their contents change (#1058) — [CHANGELOG 0.13.0](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Cargo rerun tracking was tried and reverted.** 0.14.0 began emitting `cargo:rerun-if-changed` (#1140). 0.14.1 reverted it because "this change caused regressions for some users — for example, those generating `protos` from an `includes` directory. These edge cases are difficult to address reliably" ([#1296](https://github.com/tokio-rs/prost/issues/1296)) — [CHANGELOG 0.14.1](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Formatting.** Output is formatted with `prettyplease` behind the `format` feature (config.rs:803). Files start with `// This file is @generated by prost-build.` (#935, 0.12.4) — [v0.12.4 release](https://github.com/tokio-rs/prost/releases/tag/v0.12.4)
- **Cross-type references** are emitted as relative paths. `resolve_ident` strips the common package and type-path prefix, then emits `super::` for each remaining local level, then snake_case module segments, then the UpperCamel type (`prost-build/src/code_generator.rs:1010-1043`)
- **Nested types** go into `pub mod <snake_case(message)>` under the parent struct. Oneof enums go there too, named after the oneof — [README "Oneof Fields"](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- **Keywords.** `sanitize_identifier` makes raw identifiers (`r#type`) for strict and reserved keywords of the 2015/2018/2024 editions, including `gen` (#1257). It appends `_` for `_`, `super`, `self`, `Self`, `extern` and `crate`, which can't be raw. It prefixes `_` for names that start with a digit — `prost-build/src/ident.rs:5-30` ([link](https://github.com/tokio-rs/prost/blob/v0.14.6/prost-build/src/ident.rs#L5))
- **Enum value prefix stripping** is on by default and disabled with `Config::retain_enum_prefix()` (config.rs:630). `strip_enum_prefix` strips only if the remainder starts with an uppercase char, so "Foo" is not stripped from "Foobar" (ident.rs:52-69). If two values collapse to the same Rust name, codegen **panics**: "Generated enum variant names overlap" (code_generator.rs:1170)
- **Collision bugs fixed late.** "Resolve OneOf type name conflict with embedded message (#1294)" and "Avoid OneOf type collision with enums and keyword names (#1341)" (0.14.2); "Enum variant named Error causes ambiguous item (#1098)" (0.13.1); "Prevent panic for service generator in empty module (#1442)" (0.14.5) — [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Editions are unsupported.** `Syntax::from` panics with `unknown syntax: {s}` for any value other than proto2/proto3 (`prost-build/src/code_generator/syntax.rs`). Issue [#1031](https://github.com/tokio-rs/prost/issues/1031) "Add support for editions" (open since 2024-04-18) reports that panic. A contributor listed what editions need: "Unknown fields, as discussed in #2. Open and closed enumeration behavior… Field presence disciplines"
- **Services.** prost-build calls a pluggable `ServiceGenerator` trait for `service` blocks (`prost-build/src/lib.rs:178`). tonic 0.14's `tonic-prost-build` implements `prost_build::ServiceGenerator` (`tonic-prost-build/src/lib.rs:390-392`). It re-exports `prost_build::Config` and passes `extern_path`, `field_attribute`, `type_attribute`, `boxed`, `btree_map` and `bytes` straight through (lib.rs:77, 485-560). The codec is configurable (`codec_path` defaults to `"tonic_prost::ProstCodec"`, lib.rs:111). tonic-build itself became protobuf-library-agnostic ("Decouple codegen from prost (#170)") — [tonic CHANGELOG](https://github.com/hyperium/tonic/blob/master/CHANGELOG.md); local source
- **Checked-in code / buf.** protoc-gen-prost is "A protoc plugin that generates code using the Prost! code generation engine", meant for polyglot setups with `buf`. Its plugins: protoc-gen-prost, -crate (include file + Cargo manifest), -serde (JSON via pbjson-build), -validate, and protoc-gen-tonic. It maps plugin options one-to-one onto `prost_build::Config` (`boxed=`, `btree_map=`, `bytes=`, `extern_path=`, `field_attribute=`, `retain_enum_prefix`, etc.) and calls prost-build's `generate` — [protoc-gen-prost README](https://github.com/neoeinstein/protoc-gen-prost/blob/main/README.md); `protoc-gen-prost/src/lib.rs:78,315`; `protoc-gen-prost-serde/src/generator.rs:8`

### Inferences
- prost-build's real API is "descriptor set in, Rust text out". Each frontend (protoc, protox, buf plugin) is a thin adapter. RustPB can copy this: take `FileDescriptorSet`/`CodeGeneratorRequest` as the single input contract and treat a pure-Rust parser as an optional frontend. That matters if the JS-target build cannot shell out to protoc.
- Relative `super::` paths make generated code depend on where the user `include!`s it. This is fragile when packages are split across crates. ScalaPB-style absolute paths (or `crate::`-rooted paths with a configurable root) are a design alternative to evaluate.
- Panicking in codegen on name collisions or editions is a reportable error path that RustPB should instead turn into diagnostics.

### Gaps
- I did not verify whether `edition = "2023"` fails in protox with a specific error message (inferred from the parser code only).
- I did not measure build-time cost of prost-build vs protox.

---

## 2. Runtime representation choices

### Takeaway
prost generates plain public-field structs that derive `prost::Message`. Every enum field is stored as a raw `i32` (always open, even in proto2). Message-typed fields are `Option<T>`, and prost auto-boxes them only on recursion cycles. Oneofs are `Option<enum>`. Maps are `HashMap` (opt-in `BTreeMap`). Bytes are `Vec<u8>` (opt-in `bytes::Bytes`). Strings are always `String`. proto3 implicit-presence scalars are bare `T`, and `optional` gives `Option<T>`. There is no unknown-field storage, by long-standing design choice. A PR adding it opt-in has been open since 2025-09.

### Cited Findings
- Field mapping (README table): proto2 `optional`→`Option<T>`, `required`→`T`; proto3 default→`T` for scalars and `Option<T>` for messages; proto3 `optional`→`Option<T>` — [README "Field Modifiers"](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md). proto3 optional support landed in 0.7.0 — [v0.7.0](https://github.com/tokio-rs/prost/releases/tag/v0.7.0)
- The presence decision is made in codegen. `proto3_optional` → optional. A non-`Optional` label is never optional. `Message` → optional. Otherwise only proto2 is optional (`prost-build/src/code_generator.rs:1085-1097`)
- **Enums as i32.** "All `.proto` enumeration types convert to the Rust `i32` type… The `enum` type isn't used directly as a field, because the Protobuf spec mandates that enumerations values are 'open', and decoding unrecognized enumeration values must be possible." `resolve_type` maps `Type::Enum` to `"i32"` regardless of syntax (code_generator.rs:990-996). The README lists "Preserves unknown enum values during deserialization" as a feature — [README](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- **Generated enum helpers.** `#[derive(Enumeration)]` generates `const fn is_valid` (const since 0.14.4, #1401), a deprecated `from_i32`, `From<Enum> for i32`, `TryFrom<i32>` with error `UnknownEnumValue(i32)` (changed from `DecodeError` in 0.13.0, #1010), and `Default` = **the first declared variant** (`prost-derive/src/lib.rs:300-372`). prost-build adds `as_str_name`/`from_str_name` (since 0.11). Generated enums derive `Clone, Copy, Debug, PartialEq, Eq, Hash, PartialOrd, Ord` with `#[repr(i32)]` (`code_generator.rs:765`)
- **Typed accessors on the message.** For a `Plain`/`Required` enum field, the getter is `TryFrom::try_from(self.field).unwrap_or(default)`. Unknown values silently become the default (`prost-derive/src/field/scalar.rs:290-300`). For a **repeated** enum field, the getter returns `filter_map(...ok())`, which silently drops unknown values (scalar.rs:336-344)
- **Debug of enum fields.** The derived Debug wraps each enum `i32` field and prints the variant name when `TryFrom` succeeds, otherwise the number (scalar.rs:218-229)
- **Message fields.** Wrapped in `Option` and unboxed. prost "will automatically box a message field if the field type and the parent type are recursively nested". The FAQ explains the Option: "In the protobuf wire protocol all fields are optional. The design of prost choose to expose this to the user… even at the cost of verbosity" (doc added in 0.14.2, #1358) — [README](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- **Boxing logic.** `MessageGraph` adds an edge for each non-repeated message-typed field and uses petgraph `has_path_connecting` to detect cycles (`prost-build/src/message_graph.rs:51-90`). `should_box_impl` boxes when the field's type reaches back to the parent, or when `Config::boxed` matches the path, including oneof variants as `msg.oneof_name` (`prost-build/src/context.rs:122-180`). Repeated fields are never boxed. Since 0.14.0 (#1237, breaking), even an explicit `boxed` on a repeated field is ignored to avoid `Vec<Box<T>>` — [CHANGELOG 0.14.0](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Unconstructible recursive proto2 types.** With `required Foo x = 1;` in `message Foo`, prost emits `Box<Foo>` (not optional). `Foo::decode(b"")` overflows the stack because `Default` recurses forever. Issue [#924](https://github.com/tokio-rs/prost/issues/924) is open since 2023-09-21. A maintainer: "struct Foo can never be constructed… The C++ implementation solves this by… lazy initialization… I don't know how to solve this besides throwing an error"
- **Oneofs.** The field is `Option<module::OneofName>`, an enum with one tuple variant per member. "`oneof` fields are always wrapped in an `Option`" — [README](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- **Maps.** `HashMap` by default, or `BTreeMap` via `Config::btree_map(paths)` (config.rs:114). The conformance harness compiles with BTreeMap "otherwise the comparison may fail due to inconsistent `HashMap` entry encoding ordering" (`tests/src/lib.rs:154-155`)
- **Bytes.** `Vec<u8>` by default, or `bytes::Bytes` via `Config::bytes(paths)` (config.rs:174). "When deserializing from a `Bytes` instance, this enables zero-copy deserialization for bytes fields" (0.7.0). Zero-copy support for Bytes-based fields was added in 0.8.0 — [v0.7.0](https://github.com/tokio-rs/prost/releases/tag/v0.7.0), [v0.8.0](https://github.com/tokio-rs/prost/releases/tag/v0.8.0)
- **Strings are always `String`.** A request to support `bytestring::ByteString` for zero-copy strings ([#752](https://github.com/tokio-rs/prost/issues/752)) was closed. Maintainer: "I don't want to add another ad-hoc type to the config. I want to solve this generically". In 2026-05: "Prost is currently in maintenance mode and is not adding new features… Prost will remain a pure rust/ergonomic library that doesn't focus on performance"
- **Well-known wrapper types become Rust primitives.** prost-build installs default extern paths `.google.protobuf.BoolValue`→`bool`, `Int32Value`→`i32`, `StringValue`→`String`, `BytesValue`→`Vec<u8>`, `Empty`→`()`, and the rest of `.google.protobuf`→`::prost_types` (`prost-build/src/extern_paths.rs:39-66`). prost implements `Message` for `bool`, `u32`, `u64`, `i32`, `i64`, `f32`, `f64`, `String`, `Vec<u8>` and `()` with the wrapper's field-1 encoding (`prost/src/types.rs:1-6,22-360`)
- **Derived traits.** Messages always derive `Clone, PartialEq, Message`. Since 0.13.0 prost-build also derives `Copy` when every field allows it (#950). Since 0.14.0 it derives `Eq, Hash` when possible (#1175). Both were breaking changes (`code_generator.rs:245-258`; `context.rs:183-230`; [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)). Since 0.14.0, `Debug` is no longer a supertrait of `Message` (#1147). The trait still requires `Send + Sync` (`prost/src/message.rs:15`)
- **Unknown fields: not preserved.** The README says "Does not include support for runtime reflection or message descriptors". Issue [#2](https://github.com/tokio-rs/prost/issues/2) "Add support for unknown fields" has been open since **2017-06-26**. The original author (2018-07-29): "unknown fields are full of subtle footguns… the described usecases are better either not a good fit for protobuf (RMW) or better done through preserving the original encoded message (intermediary servers)". The author's goals listed "Idiomatic generated code, performance, straightforward build system integration, Compatibility with other Protobuf libraries… Support the core proto2/proto3 features" and said feature parity with other libraries was never a goal
- In the same thread, a user reported in 2026-05 data loss or CAS failure in a read-modify-write DB workflow without unknown fields. Another commenter noted wrapper types mapped to Rust primitives "obviously won't be able to keep unknown values. The same applies to maps" — [#2](https://github.com/tokio-rs/prost/issues/2)
- Designs discussed in #2 (2024-11, a contributor and a maintainer):
  - an in-message field `__prost_unknown_fields: Vec<prost::UnknownField>` with an `.unknown_fields()` accessor;
  - an out-of-band parallel `UnknownFields` tree;
  - separate generated struct variants in a `with_unknown_fields` module.
  The maintainer: "I think it should be an optional feature to save unknown fields". Later commenters (2025-11) found that a public unknown-fields field forces every struct literal and pbjson to set it, and suggested generated constructors instead — [#2](https://github.com/tokio-rs/prost/issues/2)
- **PR [#1340](https://github.com/tokio-rs/prost/pull/1340)** "Support for Unknown Fields (Rebase of #574)" is open (created 2025-09-24, last activity 2026-10-23). It is opt-in via config. Users report running it in production for about 9–12 months. Maintainer (2026-10-23): "I have limited time and energy for prost. My company lost interest in sponsoring my time… When I do have time, I look at some quick fixes instead of a large PR… I agree this is a useful feature and I see this is behind a config flag, so the risk of breaking users is low… I can't promise anything"
- **Default values.** Plain scalar fields are encoded only when `!= default` (scalar.rs:115-125). Map entries skip default key and value inside the entry (`prost/src/encoding.rs:1044-1059`). `merge_with_default` exists "because enumeration values can have a default value other than 0 in proto2" (encoding.rs:1022-1025). For explicit `[default = …]` on proto2 optional fields, the getter returns the declared default when `None` (scalar.rs:352-372)

### Inferences
- prost's `i32` enum is the clearest negative lesson. It loses type safety (any enum's `as i32` can be assigned to any enum field, as [#76](https://github.com/tokio-rs/prost/issues/76) points out). Its typed getters quietly change unknown values into the default, or drop them in repeated fields. A sealed Rust enum plus an `Unrecognized(i32)` variant (ScalaPB's model, and the `OpenEnum<T>` of PR #1079, see §6) avoids both problems.
- Treating every enum as open, including proto2 enums, means prost cannot implement closed-enum semantics (unknown proto2 enum values belong in unknown fields). prost has no unknown-field storage, so this inference is consistent with PR #1079's description, which proposes closed proto2 enums as new behavior.
- Wrapper types mapped to bare primitives are a built-in "type mapper", but they make unknown-field preservation impossible for those types. If RustPB wants both TypeMapper and unknown-field fidelity, it has to decide where unknown fields live for mapped types.
- A public unknown-fields field breaks struct-literal construction. If RustPB stores unknown fields in-struct, it should plan constructors/builders or `..Default::default()` idioms from day one. Note a contributor's point in #1031: `..Default::default()` does not work across crates for `#[non_exhaustive]` structs.

### Gaps
- No benchmark data gathered on `HashMap` vs `BTreeMap` or on `Vec<u8>` vs `Bytes`.

---

## 3. prost-derive: what the derive generates, field kinds, encode/merge/encoded_len, errors, recursion limit

### Takeaway
`#[derive(Message)]` reads `#[prost(...)]` attributes on a hand-written or generated struct and emits:
- `encode_raw`: fields in tag order;
- `merge_field`: a `match tag` dispatch, with `skip_field` for unknown tags;
- `encoded_len`: a sum over fields;
- `clear`, `Default` and `Debug`.
Each field is one of a closed set of five kinds: scalar, message, map, oneof, group. All wire logic lives in free functions in `prost::encoding`. Decoding is merge-based, with a hard-coded recursion limit of 100. Encoding is two-pass: compute the length, then write. Errors carry a field-path stack.

### Cited Findings
- **The trait.** `Message` has `encode_raw(&self, &mut impl BufMut)` and `merge_field(&mut self, tag, wire_type, &mut impl Buf, DecodeContext)` (both `#[doc(hidden)]`, "Meant to be used only by `Message` implementations"), `encoded_len`, `clear`. Provided methods: `encode` (checks capacity and returns `EncodeError`), `encode_to_vec`, `encode_length_delimited*`, `decode` (= `Default` + `merge`), `merge` (loops `decode_key` + `merge_field`). `Box<M>` forwards the impl — `prost/src/message.rs:14-178` ([link](https://github.com/tokio-rs/prost/blob/v0.14.6/prost/src/message.rs#L14)). Since 0.13.0, methods take `impl Buf`/`impl BufMut` instead of generic params (#1045, breaking). I/O abstraction: `bytes::{Buf, BufMut}`, not `std::io` — README
- **Closed set of field kinds.** `enum Field { Scalar, Message, Map, Oneof, Group }` (`prost-derive/src/field/mod.rs:17-29`). Scalar `Ty` = `Double, Float, Int32, Int64, Uint32, Uint64, Sint32, Sint64, Fixed32, Fixed64, Sfixed32, Sfixed64, Bool, String, Bytes(Vec|Bytes), Enumeration(Path)` (`field/scalar.rs:380-397`). Scalar `Kind` = `Plain(default) | Optional(default) | Required(default) | Repeated | Packed` (scalar.rs:67-87). `packed` defaults to true for numeric repeated fields unless `packed = "false"` is given (scalar.rs:83). Unknown attributes are hard errors (e.g. `unknown attribute(s): #[prost(tags = "5,6")]`, lib.rs:678)
- **Tag inference.** Fields without `tag` are numbered sequentially after the previous one — README "Tag Inference for Existing Types"; `prost-derive/src/lib.rs:73`
- **Field order.** Fields are sorted by tag for encoding. Comment: "TODO: This encodes oneof fields in the position of their lowest tag, regardless of the currently occupied variant, is that consequential?" Debug keeps declaration order. Duplicate tags are a compile-time error (`prost-derive/src/lib.rs:84-101`)
- **Generated `merge_field`.** `match tag { <tags> => { field merge }.map_err(|e| { e.push(STRUCT_NAME, "field"); e }), _ => skip_field(...) }`, followed by `encoded_len` = `0 + Σ field_len` and `clear` (lib.rs:110-195)
- **Merge semantics.**
  - Optional message: `message::merge(wire_type, field.get_or_insert_with(Default::default), …)`, so a repeated occurrence merges into the existing value (`field/message.rs:92-107`).
  - Strings and bytes: replace, citing the spec's "last one wins" (`encoding.rs:710-722`).
  - Oneof: if the same variant is already set, merge into it; otherwise decode into a fresh default and replace (`lib.rs:442-454`).
  - Repeated numerics: accept both packed and unpacked input regardless of the declared form (`merge_repeated_numeric!`, `encoding.rs:212-242`).
- **Wire-type checking.** A known field arriving with the wrong wire type is a decode error (`UnexpectedWireType`), not stored as unknown (`prost/src/encoding/wire_type.rs:37-42`; used in `bytes::merge`, `message::merge`, etc.). Map fields were missing this check until **0.14.6 (2026-11-12, #1473)**: "A map field sent with e.g. wire type `Varint` had its following bytes reinterpreted as an entry length and decoded successfully instead of returning an error" — [PR #1473](https://github.com/tokio-rs/prost/pull/1473)
- **Encode.** `message::encode` writes the key, then `encoded_len()` as a varint, then `encode_raw` (`encoding.rs:788-795`). `encoded_len_repeated` again calls `encoded_len` per element (`encoding.rs:855-865`). `encode_raw` "will panic if the buffer has insufficient capacity" (message.rs:18)
- **UTF-8 validation.** `string::merge` decodes into the String's backing Vec through `unsafe { as_mut_vec() }` with a drop guard that clears it on error or panic. It checks `str::from_utf8` and returns `InvalidString`. The comment explains the unsafe was chosen for "up to 10% better performance" (`encoding.rs:557-605`)
- **Recursion limit.** `DecodeContext { recurse_count }` starts at `RECURSION_LIMIT = 100` ("100 is the default recursion limit in the C++ implementation", `prost/src/lib.rs:27-30`). It is decremented by `enter_recursion()` on each nested message/group and checked with `limit_reached()`. It is not configurable, and the only option is to compile it out with feature `no-recursion-limit` (`encoding.rs:31-98`; README feature list). `skip_field` also counts recursion for nested groups (encoding.rs:166-199)
- **No encode-side limit.** Issue [#736](https://github.com/tokio-rs/prost/issues/736) (open since 2022-10-26): encoding has no recursion limit, so "it is possible to encode a message that cannot be decoded anymore". A maintainer said they would accept a PR. A 2025 contributor noted the obstacle that `EncodeError` only models out-of-capacity errors
- **DecodeError design.** A boxed `Inner { description: DecodeErrorKind, stack: Vec<(&'static str, &'static str)> }`. Display prints `failed to decode Protobuf message: Msg.field: Inner.field: <kind>`. Kinds: `LengthDelimiterTooLarge, InvalidVarint, RecursionLimitReached, InvalidWireType, InvalidKey, InvalidTag, UnexpectedWireType, BufferUnderflow, DelimitedLengthExceeded, UnexpectedEndGroupTag, InvalidString, UnexpectedTypeUrl, Other` (`prost/src/error.rs:13-137`). The kind enum is `pub(crate)`. The description became an enum in 0.14.2 (#1330). That removed `DecodeError::new`, which "broke code in other crates", so it was re-added as deprecated in 0.14.3 (#1382) — [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Oneof merge dispatch.** `match tag { ..., _ => unreachable!("invalid <Oneof> tag: {}") }` (`prost-derive/src/lib.rs:482`). It is only reachable if the outer `merge_field` routes an unlisted tag
- **Verification.** prost has fuzz targets (FUZZING.md, `fuzz/`), Kani proofs (KANI.md, "Change some proptest to kani proofs (#1133)") and proptest round-trips in `encoding.rs` — repo root; [CHANGELOG 0.13.2](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)

### Inferences
- The `encode_raw` + `encoded_len` split recomputes `encoded_len()` of a nested message once per ancestor level. That makes encoding O(depth × size) for deep trees. C++ avoids this with cached sizes. RustPB could cache sizes, or compute lengths in one bottom-up pass, especially since JS output makes repeated traversals visible. This is an inference from `encoding.rs:788-795`; I found no prost issue quantifying it.
- Rejecting wrong-wire-type known fields differs from upstream parsers, which (to my knowledge) treat them as unknown fields. Upstream behavior is unverified here; see Gaps.
- The free-function `encoding::<kind>::{encode, merge, encoded_len, encode_repeated, merge_repeated, encode_packed, …}` modules form a clean, closed "wire kind" vocabulary. That vocabulary is the natural seam for a TypeMapper: a mapped field still uses the base kind's wire functions and converts at the boundary.
- `unsafe` in the string decode path and the `Send + Sync` supertrait are Rust-native choices. A Rust→JS target needs neither to have semantic weight; RustPB should keep the runtime free of `unsafe` so the JS compiler doesn't need to model it.

### Gaps
- I did not confirm what the C++ or upb parsers do on a wire-type mismatch for a known field, or whether the conformance suite tests it beyond what prost passes.

---

## 4. Extension points and custom types (TypeMapper-like features)

### Takeaway
prost-build's knobs are path-matched string injections and a few hard-coded representation switches:
- `type_attribute`, `field_attribute`, `message_attribute`, `enum_attribute`;
- `bytes`, `btree_map`, `boxed`, `skip_debug`, `disable_comments`;
- `extern_path`, `prost_path`, `type_name_domain`;
- `ServiceGenerator`.
There is **no per-field custom-type mapping**. The only workaround is a hand-written `impl Message` combined with `extern_path`, and it works only for message-typed fields. Maintainers have endorsed a generic "type override" in principle since 2022 but never built it. The project is now passively maintained and not taking features.

### Cited Findings
- **Config surface** (`prost-build/src/config.rs`):
  - `btree_map` (114), `bytes` (174), `field_attribute` (210), `type_attribute` (259), `message_attribute` (298), `enum_attribute` (347), `boxed` (370), `service_generator` (379), `compile_well_known_types` (386), `disable_comments` (414), `skip_debug` (428), `extern_path` (542);
  - `file_descriptor_set_path` (577), `skip_protoc_run` (598), `skip_source_info` (619), `retain_enum_prefix` (630), `out_dir` (639), `default_package_filename` (649), `enable_type_names` (661), `type_name_domain` (683), `prost_path` (699), `prost_types_path` (710), `protoc_arg` (732), `protoc_executable` (756), `include_file` (789), `format` (803), `compile_fds` (826), `load_fds` (918), `compile_protos` (1032), `generate` (1113).
- **Path matching.** Paths are fully qualified protobuf paths (`.pkg.Msg.field`, `.` for everything). `type_name_domain` became cumulative in 0.14.0 (#1228, breaking). `prost_path` (configurable `::prost` path) was added in 0.14.2 (#1274) — [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **extern_path.** "allows prost types in external crates to be referenced in generated code"; it works at package or type granularity, e.g. `.extern_path(".uuid", "::uuid")` or `.extern_path(".uuid.Uuid", "::uuid::Uuid")` (config.rs:440-545). It substitutes a **message type**, and the external type must implement `Message`
- **"Inject a custom type and serialize as bytes"** ([#448](https://github.com/tokio-rs/prost/issues/448), closed). The original author (2021-03-19): "Unfortunately this is not currently possible in a straightforward way… The somewhat good news is that… define struct XXX yourself and impl Message for XXX… then use prost_build::Config::extern_path". A maintainer (2024-09-11) closed it: "the requested feature… can be implemented with a custom impl Message". On a user's need for different encodings of one type: "I think that use case doesn't fit prost well. We aim to be a Rust equivalent of the C++ protobuf library." The user replied that they "moved to prost because it looked like it specifically is not an equivalent of C++ library but a Rust-idiomatic crate"
- **"Allow for custom types"** ([#427](https://github.com/tokio-rs/prost/issues/427), open since 2021-01-18). It proposes a proto option `string id = 1 [(prost.type).path = uuid::Uuid]` with `TryFrom<String>` conversion, motivated by "parse, don't validate". Other users report "a lot of duplication of types that are basically the same as the ones prost generated, but just validated". No maintainer design response
- **"Mini-RFC: Custom type wrappers"** ([#764](https://github.com/tokio-rs/prost/issues/764), open since 2022-11-25).
  - Use cases: validation wrappers (`ordered_float::NotNan<f64>`), zero-copy types, `Box`/`Arc`/heapless.
  - Key observation: wrappers "should be transparent wrt the wire format, and implementations on primitive types or wrappers around them are not necessarily messages in their own right". The earlier maintainer idea `config.add_type_override(".package.msg.field", "foo:bar::Baz")` with "its up to the user to ensure that type implements Message" conflates scalar encoding with message encoding.
  - A maintainer (2022-12) said they needed time to review. Users in 2023 and 2026-02 asked for "wire-transparent newtype wrappers for IDs" (`SmolStr`, `Arc<str>`). No implementation exists.
- **Endorsed but never built.** On [#752](https://github.com/tokio-rs/prost/issues/752) (2022-11-16) a maintainer said "I think we need better support for being able to swap out generated types… likely a substantial amount of work". The 1.0 roadmap ([#624](https://github.com/tokio-rs/prost/issues/624), open since 2022-04-13) lists "Make prost-build modular, Add Json support, Unknown fields, Extensions, Better Any support, Add support for Arc/Rc/Box". A maintainer later called 1.0 "more of a want than actually something I think we can achieve right now" ([#75](https://github.com/tokio-rs/prost/issues/75), 2022-12-12)
- **Related open requests.**
  - Field options to Rust attributes: [#658](https://github.com/tokio-rs/prost/issues/658) "Rust-specific .proto field option… to be turned into an attribute".
  - Custom options and extensions: [#256](https://github.com/tokio-rs/prost/issues/256), [#674](https://github.com/tokio-rs/prost/issues/674), [#100](https://github.com/tokio-rs/prost/issues/100) "Are proto2 extensions supported?", all open.
  - "Stabilise useful functions from encoding module": [#903](https://github.com/tokio-rs/prost/issues/903), open. `prost::encoding` is `#[doc(hidden)] pub mod` (`prost/src/lib.rs:17-18`), yet hand-written `Message` impls must use it.
- **Maintenance status.** Since 0.14.2 the README says "passively maintained": "The current maintainer is not contributing new features and doesn't have the time to review new features. Bug fixes and small improvements are welcome… The maintainer expects the official `protobuf` project to release their rust library soon and expects it to be as fully featured as the C++ library" (#1359) — [README "Contributing"](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md), [CHANGELOG 0.14.2](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **gRPC moving off prost.** The tonic repository now also contains `grpc-protobuf`/`grpc-protobuf-build` crates ("Protobuf integration for the grpc crate", authors "gRPC authors"). They depend on the official `protobuf = "4.36.2-release"` crate rather than prost (`tonic/grpc-protobuf/Cargo.toml`) — local source

### Inferences
- prost shows the design trap RustPB should avoid. Its only type-substitution seam (`extern_path`) is at the **message** level, so a scalar field (`string` → `UserId`) cannot be mapped without changing the `.proto`. A ScalaPB-style TypeMapper must be **field-level and wire-kind-preserving**: the base kind decides encoding, and the mapper only converts `Base ⇄ Custom` (possibly fallibly on decode). #764 states exactly this.
- String-typed attribute injection (`type_attribute(".", "#[derive(Serialize)]")`) is how the ecosystem bolts on serde, builders and validation. It is simple but unchecked: typos and paths that match nothing fail silently (an inference; I did not check for warnings). RustPB could keep a similar escape hatch but validate paths.
- Since prost is passively maintained and expects users to move to the official protobuf crate, a new generator should not count on upstream prost gaining TypeMapper, unknown fields, editions or JSON.

### Gaps
- I did not verify whether prost-build warns on `field_attribute`/`bytes`/`boxed` paths that match no field.

---

## 5. Reflection and JSON: prost-reflect, pbjson, conformance claims

### Takeaway
prost has no descriptors at runtime. prost-reflect (by the protox author) bolts reflection on from the outside:
- a `DescriptorPool` built from an encoded `FileDescriptorSet`;
- a `DynamicMessage` that keeps known, extension and unknown fields in one `BTreeMap<u32, …>`;
- a `ReflectMessage` trait that converts a static prost message to `DynamicMessage` by re-encoding through bytes.
prost-reflect passes the full upstream conformance suite (binary, JSON, text format) with **empty** failure lists. Static-type proto3 JSON is not in prost. It comes from pbjson-build, which protoc-gen-prost-serde wraps, or from serde attributes users inject themselves.

### Cited Findings
- prost-reflect "extending prost with reflection support and dynamic messages". `DescriptorPool` "wraps a FileDescriptorSet output by the protobuf compiler", and `DynamicMessage` "provides encoding, decoding and reflection of an arbitrary protobuf message definition". With feature `serde`, `DynamicMessage` (de)serializes "to and from the canonical JSON mapping" — [prost-reflect README](https://github.com/andrewhickman/prost-reflect/blob/main/README.md)
- **ReflectMessage.** `trait ReflectMessage: Message { fn descriptor(&self) -> MessageDescriptor; fn transcode_to_dynamic(&self) -> DynamicMessage }`. The default implementation goes "through the byte representation" and `expect`s success (`prost-reflect/src/reflect/mod.rs:8-27`). The derive takes `descriptor_pool = "STATIC"` or `file_descriptor_pool_bytes`, plus `message_name`. `prost-reflect-build::Builder` wires this into prost-build — README
- **DynamicMessage storage.** `DynamicMessageFieldSet { fields: BTreeMap<u32, ValueOrUnknown> }` with `ValueOrUnknown = Taken | Value(Value) | Unknown(UnknownFieldSet)` (`prost-reflect/src/dynamic/fields.rs:36-50`). Unifying storage reduced the footprint from 48 to 40 bytes. Unknown-field APIs `unknown_fields()`/`take_unknown_fields()` (`dynamic/mod.rs:541-556`) — [prost-reflect CHANGELOG](https://github.com/andrewhickman/prost-reflect/blob/main/CHANGELOG.md)
- **Conformance.** `prost-reflect-conformance-tests/failure_list.txt` and `text_format_failure_list.txt` are both **empty**. The runner uses `--enforce_recommended` and skips only the JSPB category (`src/main.rs:72-83`, `tests/conformance.rs:25-30`). The conformance suite is pinned through a git submodule at protobuf commit `e915ce24b3d4…` — local source
- The changelog entry where "The library now passes the protobuf conformance tests" lists the edge cases found:
  - "unpacked fields not accepting packed bytes";
  - "Unknown fields are now preserved and roundtripped";
  - "default value for enums with negative values";
  - "multiple fields of a oneof… only the last is set";
  - JSON integer `10.00` accepted;
  - `null` handling;
  - `NullValue` accepting "NULL_VALUE";
  - float range validation;
  - base64 with nonzero trailing bits;
  - FieldMask camelCase round-trip;
  - Duration trailing-zero precision;
  - Timestamp/Duration range errors;
  - duplicate oneof in JSON an error.
  Later fixes: oneof with multiple JSON members where only one is non-null is accepted; `deny_unknown_fields(false)` also ignores unknown enum values — [prost-reflect CHANGELOG](https://github.com/andrewhickman/prost-reflect/blob/main/CHANGELOG.md)
- **Editions.** prost-reflect sources mention "edition" only in `reflect/wkt.rs` (local grep). I did not find editions-feature handling
- **JSON for static types.** Requested in [#75](https://github.com/tokio-rs/prost/issues/75) "JSON Mapping" (open since 2018-02-13). The original author (2020-12-30) proposed emitting serde-derive annotations compliant with the JSON spec. A maintainer (2022-12-12): "I think we could eventually try to get pbjson merged into here" and asked for a proposal first. [#852](https://github.com/tokio-rs/prost/issues/852) "serde support for prost-types" is open
- The prost README FAQ on serde: "Probably not… Protobuf fields require a numbered tag… The mapping of Protobuf type to Rust type is not 1-to-1… six different Protobuf field types correspond to a Rust Vec<i32>… But it is possible to place serde derive tags onto the generated types" — [README FAQ](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- protoc-gen-prost-serde generates "Canonical JSON serialization of protobuf types" through `pbjson_build::Builder`. Its example `buf.gen.yaml` uses `extern_path=.google.protobuf=::pbjson_types`, replacing prost-types' WKTs with pbjson's JSON-aware ones — [protoc-gen-prost README](https://github.com/neoeinstein/protoc-gen-prost/blob/main/README.md); `protoc-gen-prost-serde/src/generator.rs:8,65`

### Inferences
- The "transcode via bytes" bridge is simple and correct by construction, but costs an encode and a decode per reflective access. A generator that emits descriptors and field accessors directly (ScalaPB's `GeneratedMessageCompanion`) avoids that.
- JSON in the prost world is split across three implementations: pbjson for static types, prost-reflect for dynamic messages, and ad-hoc serde attributes. Only prost-reflect publishes conformance results. RustPB generating proto3 JSON from the same IR as binary codecs, checked by the conformance runner, would close a real gap.
- Because static prost types drop unknown fields, `transcode_to_dynamic` cannot recover them, even though `DynamicMessage` can store them.

### Gaps
- I did not clone pbjson or check its conformance status or JSON edge cases.
- I did not map the pinned protobuf commit in prost-reflect to a release version.

---

## 6. Known bugs, breaking changes, long-standing complaints

### Takeaway
The correctness bug classes come from four areas:
- (a) stack overflow on hostile or recursive input (RUSTSEC-2020-0002; recursive proto2 `required` types still open);
- (b) missing wire-type checks (maps, fixed 2026-11);
- (c) packed-default mistakes that changed bytes on the wire (proto3 fields with options, open from 2023 until fixed 2026);
- (d) parse/format edge cases in prost-types time conversions.
Design complaints have stayed the same for about eight years: i32 enums, no unknown fields, no JSON, no reflection, `Option` for every message field, no custom types, no editions.

### Cited Findings
- **RUSTSEC-2020-0002** (prost): "Specially crafted messages can trigger a stack overflow when parsed" — [RustSec prost page](https://rustsec.org/packages/prost.html). Origin: [#267](https://github.com/tokio-rs/prost/issues/267) (2020-01-16), found with afl.rs. The original author traced it to recursion in `skip_field` for groups; it was fixed in 0.6.1 the same day. A reporter noted "in absence of stack probes on architectures like ARM, stack overflow is unsound… worse than DoS". Today `skip_field` still recurses for groups but is bounded by the recursion limit (`encoding.rs:178-188`)
- 0.8.0 release: "contains a security fix for prost-types… recommend that you upgrade… from <0.7" and changed `Timestamp → SystemTime` to fallible `TryFrom` — [v0.8.0](https://github.com/tokio-rs/prost/releases/tag/v0.8.0)
- prost-types time bugs fixed in 0.13.2:
  - "Fix date-time parsing (#1096)";
  - "'+' is not a numeric digit (#1104)";
  - "Converting DateTime to Timestamp is fallible (#1095)";
  - "Parse timestamp with long second fraction (#1106)";
  - "Format negative fractional duration (#1110)";
  - "Allow unknown local time offset (#1109)".
  Fuzz targets were added for `parse_date`/`parse_duration` (#1127, #1129) — [CHANGELOG 0.13.2](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Packed default bug ([#976](https://github.com/tokio-rs/prost/issues/976), fixed by [PR #1399](https://github.com/tokio-rs/prost/pull/1399), merged 2026-07-03, released 0.14.5).** "When a proto3 repeated packable field has any field option set (e.g., `[deprecated = true]` or custom extensions)… The `packed()` accessor returns `false` (descriptor default) instead of `true`… causing an incorrect `packed = "false"`… This changes the wire encoding from packed to unpacked". The fix reads the raw `Option<bool>` (`code_generator.rs:461-472`)
- **Map wire-type bug ([PR #1473](https://github.com/tokio-rs/prost/pull/1473), 0.14.6, 2026-11-10/12).** Maps accepted non-length-delimited wire types and reinterpreted the bytes. A fuzz-corpus case in `test_all_types_proto3` had been asserting the wrong (successful) round trip
- **Derive panic on non-ASCII attribute strings ([PR #1425](https://github.com/tokio-rs/prost/pull/1425), 0.14.5).** A byte-index slice in `Ty::from_str` panicked
- Invalid enumeration default identifiers now error instead of misbehaving (#1426, 0.14.4). Generated enumeration attributes use Display instead of Debug (#1419, 0.14.4). `Name::full_name()` is wrong for empty packages (#1386, fixed 0.14.3). `derive(Copy)` was removed on boxed fields (#1157, 0.13.3) — [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md)
- **Open:**
  - recursive proto2 `required` self-reference overflows on decode: [#924](https://github.com/tokio-rs/prost/issues/924);
  - no encode recursion limit: [#736](https://github.com/tokio-rs/prost/issues/736);
  - "I found two recursive functions which may cause stack overflow?": [#652](https://github.com/tokio-rs/prost/issues/652).
- **Oneof field-order TODO.** Oneofs are encoded at their lowest tag position regardless of the active variant (`prost-derive/src/lib.rs:87-91`, comment)
- **Enum ergonomics.**
  - [#76](https://github.com/tokio-rs/prost/issues/76) (2018, closed). The author refused a "reject unknown enum values" mode: "that would mean that prost can't be used for applications that require schema evolution… Properly handling this is one of the reasons I wrote prost".
  - [#276](https://github.com/tokio-rs/prost/issues/276) "A more ergonomic enumeration representation" (open since 2020-02-03, 28 comments) proposes a newtype with associated constants. The original author: "I'm still not seeing how this is materially better than the generated getters". Getters "have a discoverability problem… RA doesn't seem to 'see' through it". A current maintainer (2024-05-12): "I agree, we should work towards using the actual enum for enum fields".
  - [PR #1079](https://github.com/tokio-rs/prost/pull/1079) "`OpenEnum`" (open since 2024-05-28) proposes opt-in `Config::typed_enum_fields`: proto2 → the closed Rust enum, where "unknown values of the field are ignored"; proto3 → `enum OpenEnum<T> { Known(T), Unknown(Unknown) }` with "an opaque i32 wrapper to prevent arbitrary construction".
- **Option-wrapped message fields.** [#763](https://github.com/tokio-rs/prost/issues/763) "Structs should have option to generate with default values rather than Option" is open. In [#1031](https://github.com/tokio-rs/prost/issues/1031) a contributor notes "pushback even about message-typed fields being represented with an Option… and a number of duplicate issues", and that editions make *all* fields explicit-presence by default
- **Breaking changes and reasons** — [CHANGELOG](https://github.com/tokio-rs/prost/blob/v0.14.6/CHANGELOG.md):
  - 0.13.0: auto `derive(Copy)` (#950); `impl Buf` signatures (#1045); `UnknownEnumValue` error (#1010).
  - 0.14.0: `Debug` no longer a `Message` supertrait (#1147); feature `prost-derive`→`derive` (#1247); boxed repeated → `Vec<T>` (#1237); cumulative `type_name_domain` (#1228); auto `Eq`/`Hash` (#1175).
  - 0.13.0 and 0.14.2 were both yanked or superseded quickly.
  - The `DecodeError::new` removal/re-add (0.14.2/0.14.3) shows that `#[doc(hidden)] pub` items became de-facto public API.

### Inferences
- Several bugs (packed default, map wire type) were *byte-level divergences*, found late because prost's conformance run only exercises binary round-trips of two test messages. A byte-compatible RustPB needs differential tests against a reference encoder, not only round-trip tests.
- Auto-deriving traits based on field content (`Copy`, `Eq`, `Hash`) means adding a `float` or `string` field to a message silently removes trait impls. That is a source-breaking change triggered by a wire-compatible schema change. RustPB should make derived-trait sets explicit or stable.
- The `#[doc(hidden)] pub` encoding module plus "meant to be used only by Message implementations" became an unofficial ABI that the ecosystem depends on (protox, prost-reflect, custom impls). A deliberate, documented runtime API for generated code is safer.

### Gaps
- The prost CHANGELOG.md only covers 0.13.0 onward. Earlier history comes from GitHub release notes, which are sparse; I did not mine pre-0.13 bug fixes exhaustively.
- I did not check RustSec for prost-types or prost-derive advisories beyond the prost package page. The 0.8.0 note mentions a prost-types security fix whose advisory ID I did not confirm.

---

## 7. Conformance

### Takeaway
prost runs the official conformance runner (protobuf v25.8) for **binary protobuf only**. It skips all JSON, text-format and JSPB input and output. Only two tests are listed as failing, both about unknown fields. prost-reflect runs the full suite (binary, JSON, text) with no listed failures.

### Cited Findings
- `conformance/failing_tests.txt` has exactly: `# TODO(tokio-rs/prost#2): prost doesn't preserve unknown fields.`, `Required.Proto2.ProtobufInput.UnknownVarint.ProtobufOutput`, `Required.Proto3.ProtobufInput.UnknownVarint.ProtobufOutput` — local source ([link](https://github.com/tokio-rs/prost/blob/v0.14.6/conformance/failing_tests.txt))
- The runner returns `Skipped("JSON output is not supported")`, `Skipped("TEXT_FORMAT output is not supported")` and `Skipped("JSPB output is not supported")`. For JSON, JSPB and text *payloads* it returns `Skipped("JSON input is not supported")`. That message is reused for all three, a copy-paste slip (`conformance/src/main.rs:52-92`). It only handles `TestAllTypesProto2` and `TestAllTypesProto3` (main.rs:94-103)
- The test runs `conformance_test_runner --enforce_recommended --failure_list failing_tests.txt` (`conformance/tests/conformance.rs`). The protobuf version is pinned at `PROTOBUF_VERSION = "25.8"` (`protobuf/build.rs:8-9`). It is not run on Windows
- The response for a binary request is `decode → encode → decode → encode`, checking `encoded_len` equals actual length and both encodings match (`tests/src/lib.rs:156-206`). A commented-out assertion notes "TODO: Reenable this once sign-extension in negative int32s is figured out" (lib.rs:168-170)
- prost-reflect: empty `failure_list.txt` and `text_format_failure_list.txt`, `--enforce_recommended`, JSPB skipped (local source, see §5)

### Inferences
- Because v25.8 predates editions in the conformance suite, prost's "two failures" claim says nothing about editions conformance or closed-enum behavior. I did not verify which v25 tests exist for those.
- A RustPB conformance harness should cover binary + JSON (+ text if supported) from the start, pin a recent protobuf release, and keep the failure list as a tracked artifact like both projects do.

### Gaps
- I did not run either conformance suite locally. Counts of passing tests are not available from the repo files.

---

## 8. Good ideas worth copying

### Takeaway
Copy prost's simplicity:
- plain structs;
- derive-driven codegen, so a hand-written type can opt in;
- a small closed set of wire kinds behind free functions;
- descriptor-set-in / text-out codegen with pluggable frontends and service generators;
- a decode recursion limit;
- error field-path stacks;
- automatic recursion boxing;
- comment retention;
- change-only file writes.
Avoid its gaps: i32 enums, no unknown fields, message-only type substitution, no JSON, editions panics, unbounded encode recursion.

### Cited Findings
- "Generates simple, idiomatic, and readable Rust types by taking advantage of Rust derive attributes", "Retains comments from .proto files", "Allows existing Rust types (not generated from a .proto) to be serialized and deserialized by adding attributes", "Respects the Protobuf package specifier" — [README](https://github.com/tokio-rs/prost/blob/v0.14.6/README.md)
- The derive with tag inference lets hand-written structs be protobuf messages (README "Serializing Existing Types"). `extern_path` lets separately compiled crates share generated types (config.rs:440-545)
- `DecodeError` stack: `failed to decode Protobuf message: Outer.field: Inner.field: invalid string value…` (error.rs:84-92, lib.rs:117-121)
- Recursion limit 100, matching C++ (lib.rs:27-30). The `DecodeContext` is passed by value, so no exit bookkeeping is needed (encoding.rs:58-63)
- Recursive-type boxing is computed from a message graph (message_graph.rs), and `boxed()` overrides it (context.rs:122-180)
- Packed and unpacked repeated numerics are both accepted on decode (encoding.rs:212-242)
- Zero-copy `Bytes` fields when decoding from `Bytes` (encoding.rs:719-722; 0.8.0 notes)
- `include_file` generates the module tree. Files are rewritten only on change (#1058)
- `ServiceGenerator` plug-in boundary lets tonic and other RPC frameworks reuse the message codegen (prost-build/src/lib.rs:178; tonic-prost-build lib.rs:390)
- protox's `miette`-based diagnostics with source spans, and its test that compares descriptor output to protoc (protox README; tests/compare.rs)
- prost-reflect's single `BTreeMap<u32, ValueOrUnknown>` field store, holding known, extension and unknown fields in one ordered map (fields.rs:36-50)

### Inferences (RustPB-specific, relevant to the stated needs)
- **TypeMapper.** Put it at field level, keyed by fully qualified proto path or by a proto field option. Define it as conversions over the base wire kind's value type, fallible on decode (with the failure reported through the same path-stacked decode error). Never as "the custom type implements Message" (§4, #764).
- **Sealed oneofs.** prost's `Option<OneofEnum>` with one tuple variant per member is a good baseline. Consider encoding the active variant at its own tag position (prost's TODO). Make the `None` case explicit as in ScalaPB's `Empty`.
- **Enums.** Generate a Rust enum with an `Unrecognized(i32)` variant (or prost PR #1079's `OpenEnum<T>`). Apply closed-enum semantics (unknown → unknown fields) for proto2 / `enum_type = CLOSED`.
- **Unknown fields.** Store them in-struct from day one. prost's experience shows that adding them later is a breaking change that stalls for years (#2, #1340). Provide constructors or `Default` + update syntax so the extra field doesn't burden struct literals.
- **Byte compatibility.** prost's two late byte-level bugs (packed default with options; map wire types) suggest differential testing of RustPB's encoder against protoc/upb output on generated corpora, beyond the conformance suite.
- **JS target.** prost's runtime leans on `bytes::{Buf, BufMut}` trait objects and generics, on `unsafe` in string decoding, on `Send + Sync`, and on derive macros. A Rust→JS compiler must handle all of these. A plainer runtime over `&[u8]`/`Vec<u8>` with no `unsafe` will likely produce more readable JS. This is an inference; I did not evaluate it against the rust-js compiler.

### Gaps
- No quantitative comparison (code size, compile time, decode speed) of prost against alternatives was gathered here.
