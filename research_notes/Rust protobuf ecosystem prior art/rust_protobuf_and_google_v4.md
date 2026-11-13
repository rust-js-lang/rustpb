# rust-protobuf (stepancheg, v2/v3) and Google's official Rust Protobuf (v4 / google-protobuf) as prior art for RustPB

Research date: 2026-11-13. Sources read locally from depth-1 clones:
- `~/Documents/GitHub/rust-protobuf` at commit `0118c79e51c8d54ac40deb8b82631ddef81fb00e` (last commit 2025-09-21). Links below use the `RP` prefix: https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/
- `~/Documents/GitHub/protobuf` (sparse: `rust/`, `src/google/protobuf/compiler/rust/`, `conformance/`, `docs/`) at commit `8e2636c3aaeedc3cd22963814e7727564a017bfc` (2026-11-13). Links below use the `PB` prefix: https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/
- crates.io numbers come from the crates.io API, fetched 2026-11-13.

## Q1. Project status and lineage as of Nov 2026 (who maintains what, which crate name to use)

### Takeaway
rust-protobuf 3.x is at end of life. Its last release is 3.7.2 (2025-03-10), and its maintainer redirects users to Google's implementation. Google released its own implementation as the `protobuf` crate 4.x (betas from 4.30.0-beta1 in Dec 2024 to 4.36.2-release in Sep 2026). In Sep-Oct 2026 Google moved new development to a new crate, `google-protobuf` 0.x (latest 0.37.0-rc.1, 2026-10-15). The `protobuf` 4.x name becomes an unpublished re-export. Google's implementation is still a beta, not stable.

### Cited Findings
- rust-protobuf README: "This implementation is approaching end of life. Official protobuf implementation (not based on this implementation) is on the way, that will be released as protobuf=4 soon." The README also says the crate "needs help: a new maintainer" — [RP README.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/README.md)
- Maintainer, 2025-10-14: "protobuf 2.x was last released three years ago, it won't be updated. protobuf 3.x is also approaching end of life, protobuf 4.x, official implementation by Google is on the way." — [rust-protobuf #773](https://github.com/stepancheg/rust-protobuf/issues/773)
- Maintainer, 2025-03-10: "rust-protobuf library version 3 is reaching end of life and about to be replaced with official google implementation, not based on this code." — [rust-protobuf #758](https://github.com/stepancheg/rust-protobuf/issues/758)
- The 3.7.1 changelog says `protoc-gen-rust` was renamed to `protoc-gen-rs`, because "`protoc --rust_out=` no longer works with latest Google protobuf, they are working on official protobuf support for Rust." 3.7.2 is marked "(to be released)" in the changelog. It fixes "Do not stack overflow on groups" (PR #756) — [RP CHANGELOG.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG.md)
- crates.io (fetched 2026-11-13) for `protobuf`:
  - max_stable_version 3.7.2 (released 2025-03-10, about 76.7M downloads for that version alone).
  - 2.28.0 (2022-09-26, about 88.1M downloads).
  - 4.x versions use a SemVer pre-release suffix: 4.30.0-beta1 (2024-12-18), 4.31.1-release (2025-05-28), 4.33.4-release (2026-01-12, about 318k downloads), 4.36.2-release (2026-09-17).
  - Cargo therefore treats every 4.x as a pre-release. `max_stable_version` stays 3.7.2.
  - Source: [crates.io protobuf](https://crates.io/crates/protobuf)
- crates.io `google-protobuf`: versions 0.35.2-test-internal-do-not-use (2026-08-06), 0.36.1-release (2026-08-31), 0.36.2-release (2026-09-17), 0.37.0-rc.1 (2026-10-15). It has about 13.6k total downloads and no stable version. `google-protobuf-codegen` has about 378 downloads — [crates.io google-protobuf](https://crates.io/crates/google-protobuf)
- Release-crates README: "New development happens in the `google_protobuf*` crates. The legacy `protobuf*` crates are kept as re-exports of their `google_protobuf*` equivalents so that existing users keep building, but they set `publish = false` and are no longer pushed to crates.io." — [PB rust/release_crates/README.md](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/README.md)
- The `google-protobuf` README says:
  - "This is currently a beta release: the API is subject to change."
  - Rationale for the change: the crate is "backed by either a pure C implementation (upb) or the Protobuf C++ implementation. This choice was made for performance, feature parity, development velocity, and security reasons."
  - "It is not planned for the pure Rust lineage to be actively developed going forward... many open source projects may reasonably continue to stay on the `protobuf` crate."
  - "V4.36.0 of the `protobuf` crate corresponds to V0.36.0 of this crate."
  - Source: [PB rust/release_crates/google_protobuf/README.md](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/google_protobuf/README.md)
- The legacy `protobuf` Cargo template sets `version = "{PROTOBUF_LEGACY_VERSION}"`, `publish = false`, and depends on `google-protobuf`. `protobuf_version.bzl` sets `PROTOBUF_RUST_VERSION = "0.38.0"` and `PROTOBUF_LEGACY_RUST_VERSION = "4.38.0"`. `version.json` on main shows `"rust": "0.38-dev"` and `"legacy_rust": "4.38-dev"` — [PB rust/release_crates/protobuf/Cargo-template.toml](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/protobuf/Cargo-template.toml), [PB version.json](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/version.json)
- Beta announcement:
  - Adam Cozzette, for the Protobuf team, 2025-06-19: version 4.31.1-release is beta. "-release" marks each beta.
  - The crates.io release supports only the upb kernel.
  - The team aimed for a stable release by end of 2025. As of Nov 2026, no stable release exists per crates.io above.
  - Source: [protobuf Google Group announcement](https://groups.google.com/g/protobuf/c/Ur70AGjgHnw)
- Google maintainer (esrauchg), 2026-03-10: "This strictness is temporary... We're still doing some final breaking changes to try to improve the developer ergonomics before we finally stabilize it (including to traits, etc). This has the effect that the gencode is incompatible across versions." — [protobuf #26271](https://github.com/protocolbuffers/protobuf/issues/26271)
- Same maintainer, 2025-12-16: "Note that the Proto2 and Proto3 syntaxes are considered fully 'done'/frozen, so for users of those I think you also can likely consider Prost and the v3 version of the Protobuf crate to be effectively 'feature complete'... We did push a security fix on the v3 version of the Protobuf crate." — [protobuf #24880](https://github.com/protocolbuffers/protobuf/issues/24880)
- The 2022 "Past, present and future" document by stepancheg says he has "a full-time job unrelated to rust-protobuf" and that "having more than one protobuf implementation is not necessarily good". He floated keeping v2 and recommending prost instead — [RP doc/past-present-future.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/past-present-future.md)

### Inferences
- In late 2026 no maintained, pure-Rust, spec-complete implementation exists, by the projects' own statements:
  - rust-protobuf is EOL.
  - prost is described (quoted in #24880) as being in passive maintenance.
  - Google's crate is C-backed and still beta.
- That leaves a real opening for RustPB, and a real burden: correctness and security fixes would be RustPB's to own.
- The crate rename from `protobuf` 4.x to `google-protobuf` 0.x, made while still in beta, shows how unsettled Google's packaging is.

### Gaps
- No public announcement of the `google-protobuf` rename was found, only the README in the repo and crates.io. The date and reason for the rename beyond the README are unknown.

## Q2. rust-protobuf codegen: parser, plugin, generated layout, customization (is there a custom type mapping?)

### Takeaway
Codegen is a Rust library, `protobuf-codegen`, with two `.proto` front ends:
- a pure-Rust parser and typechecker (`protobuf-parse`);
- `protoc` producing a descriptor set.

It also ships a `protoc-gen-rs` plugin. Generated code is plain structs with public fields, plus large amounts of reflection glue.

Customization is a small set of boolean switches. These can be set per file, message or field, through the `rustproto.proto` extension options or a `CustomizeCallback`, plus a free-text `before` hook that injects attributes. There is no custom type mapping. A request for a ScalaPB-style type mapping was declined as "no way to do that now", and the user moved to prost.

### Cited Findings
- `Codegen::pure()` and `Codegen::protoc()` switch the front end.
  - The doc comment on `Codegen::new()` says "Uses `protoc` from `$PATH` by default", but `impl Default for WhichParser` returns `WhichParser::Pure`.
  - The 3.0.0-alpha.3 changelog says "Default `.proto` parser in `protobuf-codegen` is now pure-rust (not using `protoc` command)". The code docs are internally inconsistent here.
  - Sources: [RP protobuf-codegen/src/codegen/mod.rs L63-82](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-codegen/src/codegen/mod.rs#L63-L82), [RP protobuf-parse/src/which_parser.rs L10-13](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-parse/src/which_parser.rs#L10-L13), [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- The pure-Rust parser was implemented after issue #271, "Implement pure-rust proto parser and code generator" (2018). In 3.0.0-alpha.2 it moved to a separate crate, `protobuf-parse` — [rust-protobuf #271](https://github.com/stepancheg/rust-protobuf/issues/271), [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- The full option set in `rustproto.proto`: `generate_accessors`, `generate_getter`, `tokio_bytes`, `tokio_bytes_for_string`, `oneofs_non_exhaustive`, `btreemap`, and file-level `lite_runtime_all`.
  - Each has `_all` (FileOptions), plain (MessageOptions) and `_field` (FieldOptions) forms, at extension numbers 17004-17035.
  - A comment credits gogoproto as "the original idea".
  - Source: [RP proto/rustproto.proto](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/proto/rustproto.proto)
- `Customize` contains `before: Option<String>` ("Code to insert before the element in the generated file"), the booleans above, `gen_mod_rs` and `inside_protobuf`.
  - `CustomizeCallback` has hooks `file`, `message`, `field`, `special_field`, `enumeration` and `oneof`, each returning a `Customize`.
  - Sources: [RP protobuf-codegen/src/customize/mod.rs L15-45, L88-116](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-codegen/src/customize/mod.rs#L15-L116)
- The serde example uses the `before` hook to add `#[derive(::serde::Serialize, ::serde::Deserialize)]`.
  - It needs external `serialize_with` functions, because "`EnumOrUnknown` is not a part of rust-protobuf, so external serializer is needed".
  - It adds `#[serde(skip)]` on special fields.
  - Source: [RP protobuf-examples/customize-serde/build.rs](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-examples/customize-serde/build.rs)
- Custom type request (2021). The user asked to map `bytes locallyFree = 1 [(scalapb.field).type = "scala.collection.immutable.BitSet"]` to a Rust type.
  - stepancheg: "There's no way to do that now, but I'm open do considering designing and implementing this extension."
  - The user: "finally I solved it with prost."
  - Source: [rust-protobuf #552](https://github.com/stepancheg/rust-protobuf/issues/552)
- v3 layout changes:
  - "Nested messages are now generated in nested mods".
  - "all fields are public now in generated messages" (alpha.11).
  - Getters lost the `get_` prefix (alpha.7).
  - Keywords are escaped with a `_` suffix (alpha.12).
  - Enum variant names are converted to camel case and `SpecialFields` is generated (alpha.10).
  - `gen_mod_rs` is on by default (alpha.6).
  - The parser "switches by tag, not by field name" (alpha.7).
  - Source: [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- Generated `merge_from` is a `while let Some(tag) = is.read_raw_tag_or_eof()? { match tag { 10 => ..., tag => crate::rt::read_unknown_or_skip_group(tag, is, self.special_fields.mut_unknown_fields())? } }` loop over precomputed wire tags — [RP protobuf/src/well_known_types/struct_.rs L709-721](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/well_known_types/struct_.rs#L709-L721)
- Each generated file embeds a serialized `FileDescriptorProto` (`static file_descriptor_proto_data: &'static [u8] = b"..."`) and parses it lazily for reflection. `struct_.rs` is 996 lines for 4 small messages and 1 enum — [RP struct_.rs L848, L970](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/well_known_types/struct_.rs#L848)
- stepancheg on prost's derive-based approach: derives are "unnecessary, if code is generated" and "make code harder to understand: something like `cargo expand` is needed" — [RP protobuf-examples/vs-prost/README.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-examples/vs-prost/README.md)

### Inferences
- RustPB's headline feature, per-field TypeMapper, has no equivalent in rust-protobuf.
- The `before` text hook is the only extension point. It is untyped, so any attribute added through it must also cope with runtime types (`EnumOrUnknown`, `SpecialFields`) that the attribute's crate does not know about.
- Having both a declarative `.proto` option and a programmatic callback for the same setting is a "two ways" design worth avoiding or unifying.

### Gaps
- No evidence that the pure parser handles Protobuf Editions. `grep -i edition` in `protobuf-parse/src/pure/parser.rs` returned nothing. This is an inference from the absence of matches, not a tested result.

## Q3. rust-protobuf runtime representation: MessageField, EnumOrUnknown, special fields, oneofs, repeated, maps, bytes/strings

### Takeaway
The v3 representation is close to "plain structs":
- `MessageField<T>(pub Option<Box<T>>)` for singular sub-messages;
- `EnumOrUnknown<E>`, an `i32` newtype, for every enum field;
- `Vec<T>` for repeated fields;
- `HashMap` or optionally `BTreeMap` for maps;
- `Option<oneof_enum>` for oneofs;
- one `special_fields: SpecialFields` field, which bundles `UnknownFields` and an atomic `CachedSize`.

v2's `SingularPtrField` and `RepeatedField` (types built to reuse allocations) were demoted and then removed. `Deref` to a default instance is kept, but `DerefMut` was removed as "too dangerous".

### Cited Findings
- `pub struct MessageField<T>(pub Option<Box<T>>);` is documented as a "Wrapper around `Option<Box<T>>`, convenient newtype".
  - `impl<M: Message> Deref for MessageField<M>` returns `get_or_default()`, which is a static default instance when the field is unset.
  - There is deliberately no `From<Option<Box<T>>>` "because it would make type inference worse".
  - Source: [RP protobuf/src/message_field.rs L7-33, L167-189](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/message_field.rs#L7-L189)
- Changelog for 3.0.0-alpha.9: "Unimplement `DerefMut` for `MessageField`: `DerefMut` which modifies the object state is too dangerous. `Deref` is kept though." — [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- Why `Vec` and `Option<Box>` replaced the v2 types:
  - On `RepeatedField`: it "helps with high performance code: allocated memory for one message can be reused for next message, but practically people rarely reuse messages." An option to generate `Vec` was added in 2018.
  - The analogous option `singular_field_option_box` was added for singular fields.
  - Sources: [rust-protobuf #280](https://github.com/stepancheg/rust-protobuf/issues/280), [rust-protobuf #300](https://github.com/stepancheg/rust-protobuf/issues/300)
- `EnumOrUnknown<E> { value: i32, _marker: PhantomData<E> }` is `#[repr(transparent)]`, documented as "Protobuf enums with possibly unknown values are preserved in this struct". `from_i32` is a `const fn`, and the type is not bounded on `E: Enum` because of the const-fn trait-bound limits of the time — [RP protobuf/src/enum_or_unknown.rs L11-31](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/enum_or_unknown.rs#L11-L31)
- Background: v2 failed to decode unknown enum values (issue #233, 2017).
  - One user said every client had to upgrade when a server added a value.
  - Another user pointed out that proto2 semantics require unknown closed-enum values to go to unknown fields.
  - The fix became `EnumOrUnknown` in v3.
  - Source: [rust-protobuf #233](https://github.com/stepancheg/rust-protobuf/issues/233)
- Proto2 (closed) enums are also generated as `Option<EnumOrUnknown<E>>`. Examples: `pub label: Option<EnumOrUnknown<field_descriptor_proto::Label>>` and `self.label = Some(is.read_enum_or_unknown()?)`. Unknown values are kept in the field rather than moved to unknown fields — [RP protobuf/src/descriptor.rs L1531, L1974](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/descriptor.rs#L1531)
- `SpecialFields { unknown_fields: UnknownFields, cached_size: CachedSize }` derives `Default, Eq, PartialEq, Clone, Debug, Hash` — [RP protobuf/src/special.rs L4-18](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/special.rs#L4-L18)
- `CachedSize { size: AtomicUsize }`:
  - "It is always equal to itself to simplify generated code (Generated code can use `#[derive(Eq)]`)".
  - `get` and `set` use `Ordering::Relaxed` and truncate to `u32`.
  - Source: [RP protobuf/src/cached_size.rs L6-35](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/cached_size.rs#L6-L35)
- FAQ rationale for `cached_size`: serialization is two passes, size then write. "Previously rust-protobuf stored message sizes in external buffer, but it was slightly worse performance-wise." It offers an option to skip the field on request — [RP doc/FAQ.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/FAQ.md)
- vs-prost note on prost not caching sizes: "In the worst case, with deeply nested messages, it results in exponential growth serialization time." Before 2014 rust-protobuf kept sizes in a queue or stack, and switched "because storing cached size is faster" — [RP vs-prost README](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-examples/vs-prost/README.md)
- `UnknownFields` wraps `fields: Option<Box<HashMap<u32, UnknownValues, BuildHasherDefault<DefaultHasher>>>>`. `Option` is used because "HashMap constructor performs allocation".
  - A field-number-keyed map does not preserve the original interleaving order of unknown fields on re-serialization.
  - Source: [RP protobuf/src/unknown.rs L224-237](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/unknown.rs#L224-L237)
- Oneofs are generated as `pub kind: Option<value::Kind>` with an enum in a nested module.
  - Oneof enums are `#[non_exhaustive]` by default (3.0.0-alpha.6). An option to disable that was added in 3.5.0 (PR #726).
  - Sources: [RP struct_.rs L182-188](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/well_known_types/struct_.rs#L182-L188), [RP CHANGELOG.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG.md)
- `Chars` is a "Thin wrapper around `Bytes` which guarantees that bytes are valid UTF-8 string". It is used with `tokio_bytes_for_string`, while `tokio_bytes` maps `bytes` fields to `bytes::Bytes` — [RP protobuf/src/chars.rs L10-13](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/chars.rs#L10-L13), [RP rustproto.proto](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/proto/rustproto.proto)
- `Message` requires `PartialEq` (3.0.0-alpha.10). `Display` prints text format and `Debug` is a standard derive (alpha.4); in v2, `Debug` printed text format. Maintainer: "there was no clear explanation what `Debug` was supposed to do" — [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md), [RP doc/past-present-future.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/past-present-future.md)
- Default instances: "For each message, rust-protobuf generates `M::default_instance()` function and `Default for <&M>` (similarly to what C++ and Java generators do)" — [RP vs-prost README](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-examples/vs-prost/README.md)
- The runtime's own source contains 47 lines with `unsafe`, across 13 files, including:
  - `coded_input_stream/buf_read_iter.rs` (uninitialized buffers);
  - `reflect/repeated/transmute.rs`;
  - `message_dyn.rs`;
  - `owning_ref.rs`.
  - This was counted locally with grep, so treat it as approximate.
  - A 2022 fix covered "invalid aliasing and uninitialized memory access" (PR #592).
  - Sources: [RP protobuf/src/coded_input_stream/buf_read_iter.rs L219-233](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/coded_input_stream/buf_read_iter.rs#L219-L233), [rust-protobuf PR #592](https://github.com/stepancheg/rust-protobuf/pull/592)

### Inferences
- `MessageField` plus `Deref` to a default instance is ergonomic, but it hides presence: `msg.sub.x` silently reads a default. ScalaPB-style `Option` is more explicit.
- `EnumOrUnknown` for open enums is a good pattern for RustPB.
- Applying it to closed (proto2) enums is a spec deviation (see Q6).
- Storing unknown fields as a `HashMap` by number loses their wire order. That conflicts with RustPB's "byte-compatible" goal for round-trips; an ordered byte buffer is simpler and exact.
- `AtomicUsize` `CachedSize` and `Box`-based fields matter for a Rust-to-JS target: atomics and interior mutability must lower cleanly.

### Gaps
- Lazy fields: no `[lazy=true]` support was found in codegen (`grep lazy` in `protobuf-codegen/src/gen/field` found nothing). The `Lazy` type in the runtime is only for static descriptor initialization (`once_cell`).

## Q4. rust-protobuf reflection, dynamic messages and binary-size cost

### Takeaway
v3's main reason for existing was full runtime reflection modeled on C++. That means descriptors built from embedded `FileDescriptorProto` bytes, `MessageDyn` trait objects, dynamic messages built from `.proto` at runtime, and JSON and text format implemented on top of reflection. The maintainer states the cost directly: large generated code and binary size. A lite mode without reflection was restored in 3.0.0-alpha.8.

### Cited Findings
- README: "Version 3... implements: runtime reflection; JSON and text format parsing and printing; dynamic messages (messages which can be created from `.proto` file on the fly without code generation)" — [RP README.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/README.md)
- "v3 cannot be implemented in v2 without breaking backwards compatibility". The changes include "proper runtime reflection", "dynamic messages", "json and text format", and "better mapping of proto messages to rust module namespaces" — [RP doc/past-present-future.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/past-present-future.md)
- On cost: "rust-protobuf fully supports runtime reflection... implemented similarly to C++ implementation of protobuf. The drawback is that generated code is quite large (which also affects binary size)." — [RP vs-prost README](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-examples/vs-prost/README.md)
- "Lite runtime generation is restored. When lite runtime requested, code is generated without reflection support" (3.0.0-alpha.8). "Smaller generated code (common code snippets extracted into the library runtime)" (alpha.4) — [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- The reflection module tree includes `reflect/dynamic`, `reflect/acc` (accessors), `reflect/repeated/transmute.rs`, `reflect_eq.rs`, `service` and `oneof` — [RP protobuf/src/reflect](https://github.com/stepancheg/rust-protobuf/tree/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/reflect)
- serde was removed in v3: "JSON is supported natively in rust-protobuf, it is implemented using reflection; for those who still needs serde, generated code can be customized to insert any annotations" — [rust-protobuf #519](https://github.com/stepancheg/rust-protobuf/issues/519)
- Open reflection bugs:
  - #741 (2024): "Calling `set_singular_field` on dynamic message with generated descriptor".
  - #690 (2023): "`From` implementations for some WKTs break their contracts".
  - Source: [rust-protobuf issues](https://github.com/stepancheg/rust-protobuf/issues)

### Inferences
- Making JSON depend on reflection ties JSON's cost to full descriptors in every binary. For a JS output target, embedded descriptors and reflection tables become bundle size.
- Generated, monomorphic JSON code per message (the approach ScalaPB-json and pbjson take) is the safer default. Optional reflection can be added later.

### Gaps
- No measured binary-size numbers for rust-protobuf with and without lite were found.

## Q5. rust-protobuf JSON (protobuf-json-mapping) and conformance level

### Takeaway
JSON is a separate crate, `protobuf-json-mapping` (3.7.2), built on `MessageDyn` reflection. Print and parse options cover the standard ones. `Any` JSON printing and parsing are not implemented. No conformance-suite runner exists in the repository. One open issue (#777, Oct 2026) asks for leniency for unknown enum strings.

### Cited Findings
- Crate docs: "JSON printer and parser which tries to follow protobuf conventions" — [RP protobuf-json-mapping/src/lib.rs L1-2](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/lib.rs#L1-L2)
- `PrintOptions { enum_values_int, proto_field_name, always_output_default_values, _future_options: () }`. `ParseOptions { ignore_unknown_fields, _future_options: () }`. `_future_options` is a field-based trick to prevent exhaustive struct literals — [RP print.rs L556-568](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/print.rs#L556-L568), [RP parse.rs L852-860](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/parse.rs#L852-L860)
- `Any` is unsupported in both directions:
  - print: `#[error("JSON printing of Any is not implemented")] AnyPrintingIsNotImplemented`, returned from `impl PrintableToJson for Any`;
  - parse: `AnyParsingIsNotImplemented` from `merge_wk_any`.
  - Sources: [RP print.rs L42-43, L247-249](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/print.rs#L42-L249), [RP parse.rs L75-76, L818-820](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/parse.rs#L75-L820)
- Well-known types handled specially: Struct, ListValue, Value, Duration, Timestamp, FieldMask and wrappers (`merge_wk_*`; `well_known_wrapper.rs`, `rfc_3339.rs`) — [RP parse.rs L471-818](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/parse.rs#L471-L818)
- 3.0.0-alpha.2: "Use `always_output_default_values` option to output empty array for repeated fields when serializing to JSON" (PR #550) — [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- #777 (open, 2026-10-20) asks the JSON parser to accept unknown enum strings. It cites the spec: "Adding values to an enum is safe if the 'Emit enum values as integers' is set on all relevant clients" — [rust-protobuf #777](https://github.com/stepancheg/rust-protobuf/issues/777)
- No conformance runner: `grep -rl conformance` outside the vendored Google protos found nothing in the repo (local check at commit 0118c79).

### Inferences
- rust-protobuf's JSON conformance level is unknown and probably partial, given the missing `Any`.
- For RustPB, proto3 JSON should be gated on the official conformance runner from day one. Google's `conformance_test_runner` drives a stdin/stdout testee, so RustPB can plug in natively and, through the JS compiler, as a JS process.

### Gaps
- No published pass or fail counts for rust-protobuf against the conformance suite were found.

## Q6. rust-protobuf known bugs and design mistakes (decoding correctness, panics, performance, ergonomics)

### Takeaway
Documented problems:
- a stack-overflow DoS when skipping nested groups (RUSTSEC-2024-0437 / CVE-2025-53605, fixed in 3.7.2);
- an open, byte-compatibility bug that writes corrupt output for negative unknown enum values in packed repeated fields (#766);
- historical panics on short buffers and aliasing UB;
- v2's failure on unknown enums.

Code inspection also shows two spec deviations:
- singular and oneof message fields that appear twice on the wire are replaced, not merged;
- closed proto2 enums keep unknown values in the field instead of routing them to unknown fields.

### Cited Findings
- RUSTSEC-2024-0437 "Crash due to uncontrolled recursion in protobuf crate":
  - affects `<=3.4.0`, patched `>=3.7.2`;
  - affected function `CodedInputStream::skip_group`;
  - "An attacker can exploit this to cause a stack overflow while parsing a message from untrusted data";
  - CVE-2025-53605, GHSA-2gh3-rmm4-6rq5, issue #749.
  - Source: [RustSec advisory](https://rustsec.org/advisories/RUSTSEC-2024-0437.html)
- The 3.7.2 changelog entry is "Do not stack overflow on groups" (PR #756) — [RP CHANGELOG.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG.md)
- Message recursion is otherwise limited: `const DEFAULT_RECURSION_LIMIT: u32 = 100;` with the comment "100 is the default value of C++'s implementation", via `incr_recursion()`. The bug was that the group-skipping path bypassed it — [RP coded_input_stream/mod.rs L50-51, L105-116](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/coded_input_stream/mod.rs#L50-L116)
- #766 (open since 2025-05-24) "Error when encoding unknown enum":
  - Parsing then re-serializing a packed repeated enum containing -1 produces a length prefix of 5 but a 10-byte varint. Reparse fails with `UnexpectedEof`.
  - A commenter suspects `vec_packed_enum_or_unknown_data_size` uses `compute_raw_varint32_size(e.value() as u32)`, which undercounts negative values.
  - Source: [rust-protobuf #766](https://github.com/stepancheg/rust-protobuf/issues/766)
- #758 asked whether negative `int32` should take 5 bytes. The maintainer showed that C++ also writes 10 bytes (sign-extended), and the reporter confirmed. This is a useful byte-compat fact: negative int32 and enum values are 10-byte varints — [rust-protobuf #758](https://github.com/stepancheg/rust-protobuf/issues/758)
- Singular message merge deviation: `read_singular_message_into_field` does `let mut m = M::new(); is.merge_message(&mut m)?; *target = MessageField::some(m);`, which replaces any existing value instead of merging into it — [RP protobuf/src/rt/message.rs L8-19](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/rt/message.rs#L8-L19)
- Oneof message members likewise do `self.kind = Some(value::Kind::StructValue(is.read_message()?))`. The protobuf encoding spec requires the parser to merge repeated occurrences of a singular message field — [RP struct_.rs L500-505](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/well_known_types/struct_.rs#L500-L505)
- Closed-enum deviation: proto2 enum fields are `Option<EnumOrUnknown<E>>` and decode any `i32` into the field. See Q3, [RP descriptor.rs L1531, L1974](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/descriptor.rs#L1531). In #233 a user cited the spec rule that unknown closed-enum values go to unknown fields — [rust-protobuf #233](https://github.com/stepancheg/rust-protobuf/issues/233)
- Other historical fixes:
  - "don't panic on short buffer" (2.25.2, PR #571);
  - "Fixed invalid aliasing and uninitialized memory access" (2.27.0 and 3.0.0-alpha.6, PR #592);
  - "Fix writing large messages" (3.5.0, PR #725);
  - "CodedOutputStream panics on broken pipe" (#714; flush errors in `Drop` are now ignored);
  - "Make code generation deterministic" (2.25.0, #562);
  - "Default to packed for repeated primitives in proto3" (3.5.0, PR #707) — before that, proto3 repeated scalars were apparently not packed by default;
  - "Fix text format parsing of non-ASCII" (3.6.0).
  - Sources: [RP CHANGELOG.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG.md), [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- Ergonomics and design changes:
  - "`PartialEq` with large number of fields now panics. Previously it could cause stack overflow in the Rust compiler." (3.0.0-alpha.1);
  - "Message size computation uses u64 now. It is explicit error now on attempt to serialize a message larger than 2GiB" (alpha.7);
  - "write_to_bytes is infallible but returns a Result" (#404).
  - Sources: [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md), [rust-protobuf #404](https://github.com/stepancheg/rust-protobuf/issues/404)
- UTF-8 is validated on read: `String::from_utf8(vec)` maps to `WireError::Utf8Error`, and `Chars::from_bytes` maps to `ProtobufError::Utf8` — [RP coded_input_stream/mod.rs L606, L630-632](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/coded_input_stream/mod.rs#L606-L632)
- Maintainer on historical baggage: "Many API decisions of rust-protobuf which seem to be strange, they can be explained by historical reasons", such as `Debug` printing text format and a codegen built for files committed to the source tree before `build.rs` existed — [RP doc/past-present-future.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/past-present-future.md)

### Inferences
- Any group- or unknown-field-skipping path needs the same depth budget as message recursion. The CVE was exactly a second recursion path that missed the limit.
- Size computation and writing must be one rule, one owner, with negative `int32` and enum values encoded as 10-byte sign-extended varints. #766 is a divergence between `compute_size` and `write` for the same value.
- Merge semantics for repeated singular-message occurrences and closed-enum routing should be tested explicitly. They are easy to get wrong and the conformance suite covers them.

### Gaps
- The exact status of the merge behavior against the conformance suite is unverified. No rust-protobuf conformance run was found, and I did not run one.

## Q7. rust-protobuf v2 to v3 breaking changes and stated reasons

### Takeaway
v3 (2022-05-01) was a broad break, framed as "mostly fixing past mistakes". The changes were reflection, dynamic messages, JSON and text format, open enums via `EnumOrUnknown`, nested modules, plain `Vec`/`Option<Box>`-like fields, public fields, dropping `get_` prefixes, dropping native serde, and moving JSON to its own crate.

### Cited Findings
- "Version 3-alpha contains multiple improvements (mostly fixing past mistakes)"; "It is not possible to implement certain features of rust-protobuf 3 in version 2 without breaking backwards compatibility." — [RP doc/past-present-future.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/doc/past-present-future.md)
- 3.0.0-alpha.1 breaking list:
  - enums generated as `ProtobufEnumOrUnknown<E>` (#233);
  - nested messages in nested mods;
  - no getters for public fields;
  - removal of global `parse_length_delimited*`;
  - `PartialEq` panics with many fields.
  - Later alphas renamed `ProtobufError` to an opaque `Error`, replaced `SingularPtrField` with `MessageField`, removed serde (#519), moved JSON to `protobuf-json-mapping` (alpha.14), made `file_descriptor()` return `&'static FileDescriptor`, and hid `UnknownValues`.
  - Source: [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md)
- v2's last release 2.28.0 (2022-09-26) backported proto3 `optional` fields (PR #649). "Only most critical bugfixes will be applied to 2.x version" — [RP CHANGELOG-before-3.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/CHANGELOG-before-3.md), [RP README.md](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/README.md)
- 2.28.0 remains vulnerable to RUSTSEC-2024-0437 and will not be patched — [rust-protobuf #773](https://github.com/stepancheg/rust-protobuf/issues/773)

### Inferences
- Many v3 changes (open enums, `Vec`, public fields, `Debug` as derive) converged toward plain-struct designs like prost's and ScalaPB's. RustPB can start from that converged point instead of repeating the path.

### Gaps
- None material.

## Q8. Google v4: design rationale (kernels, views/muts proxies, why not plain structs, goals)

### Takeaway
Google's Rust Protobuf is "a safe Rust API implemented on top of existing Protobuf implementations" (kernels: upb in C, C++, and C++ Lite). The stated reasons:
- zero-cost in-memory interop with C++ in mixed binaries;
- one API across kernels;
- not adding a new parser, for security and correctness.

The opaque `Foo`/`FooView<'a>`/`FooMut<'a>` proxy design follows from the kernel choice: with the C++ kernel, children live on the C++ heap, so no Rust `&T` can exist. The proxies also keep freedom to optimize and block `mem::swap` of children. A pure-Rust kernel is a wish with "no concrete roadmap". Google's Rust lead said it would most likely come from rewriting upb in Rust.

### Cited Findings
- Design-decisions page:
  - kernels C++, C++ Lite and upb, with upb "the default in open-source builds";
  - the cost of serialization at language boundaries "would prevent adoption" for C++-to-Rust migration;
  - "No Pure Rust Kernel": a new Rust parser would be "much more likely to contain critical vulnerabilities" than mature C++/upb parsers; Google expects a pure-Rust implementation eventually but has "no concrete roadmap";
  - a second official implementation with a different API is not planned.
  - Source: [protobuf.dev Rust Proto Design Decisions](https://protobuf.dev/reference/rust/rust-design-decisions/)
- Same page on proxies:
  - "A `&T` requires a real Rust `T` in memory"; with the C++ kernel a parsed message's children live on the C++ heap;
  - proxies let Google control exposed behavior and change the implementation (for example a lazily built `Mut`);
  - `&mut SomeChild` would allow `std::mem::swap()`, breaking parent/child invariants;
  - APIs should take `SomeMsgView<'a>` rather than `&'a SomeMsg`.
  - Source: [protobuf.dev Rust Proto Design Decisions](https://protobuf.dev/reference/rust/rust-design-decisions/)
- Same page on strings: `string` getters return `ProtoStr`/`ProtoString`, which "may rarely contain invalid UTF-8", because C++ may hold non-UTF-8 in string fields and validates only during serialization. Setters accept `&str`/`String`. Own std-like types (`protobuf::Utf8Error`) keep room to swap in C++'s faster UTF-8 validator — [protobuf.dev Rust Proto Design Decisions](https://protobuf.dev/reference/rust/rust-design-decisions/)
- esrauchg (Google), 2025-12-16, on pure Rust:
  - "likely by rewriting upb itself into Rust (which could then be used for all upb-backed implementations, including Python, Ruby, PHP)... we unfortunately don't have very strong enough business impact justifications for the pure Rust support."
  - "the primary concern here is less about performance and more about correctness and maintenance concerns by adding yet another Protobuf parser... our Pure Python and Pure PHP offerings have some correctness concerns which don't exist when those languages are backed by upb."
  - "Prost has a number of non-spec edge cases that would disqualify it from Google 1P usage... the Prost authors even deliberately chose not to implement 'correctly' in some cases like groups, unknown fields, closed enums."
  - Source: [protobuf #24880](https://github.com/protocolbuffers/protobuf/issues/24880)
- On equality (2025-08-11): "Protobuf discourages equality checks in most languages... it not possible to canonicalize unknown fields." Tree shaking can turn known fields into unknown fields at runtime, so `Eq` "will work in production... until the moment that it doesn't". Messages do not implement `PartialEq`. A free function `message_eq` shipped in 4.34.0-rc.1 — [protobuf #23058](https://github.com/protocolbuffers/protobuf/issues/23058), [PB rust/shared.rs L17-18](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/shared.rs#L17-L18)
- On builder-style setters returning `&mut self` (declined 2025-09-09): use the `proto!` macro instead, for example `proto!(SubMsg { x: 3, y: "hello", submsg: SomeSubmsg { z: 42 } })` — [protobuf #23412](https://github.com/protocolbuffers/protobuf/issues/23412)
- On `required` and presence (2025-06): Google pointed to the Go Opaque API as the analogous design shift. It does not adopt builder-only construction as a first-class concept — [protobuf #22075](https://github.com/protocolbuffers/protobuf/issues/22075)

### Inferences
- Every v4 design driver (C++ ABI interop, swappable kernels, tree shaking, non-UTF-8 strings from C++) comes from a large mixed C++/Rust deployment. None applies to RustPB, which is pure Rust compiled to native and to JS.
- RustPB can legitimately choose plain owned structs with `String`, keeping guaranteed UTF-8 and rejecting invalid UTF-8 at parse time as proto3 requires.
- The concerns Google raises are still worth designing for:
  - unknown-field-aware equality;
  - `mem::swap` breaking invariants, which only matters if structs carry hidden invariants (for example cached sizes or presence bits);
  - correctness of a new parser, which argues for heavy conformance and fuzz testing.

### Gaps
- No published Google benchmarks comparing v4 (upb) to prost or rust-protobuf were found.

## Q9. Google v4: codegen and generated API shape (getters/setters, enums, oneofs, presence, unknown fields, editions)

### Takeaway
Codegen is a C++ `protoc` plugin (`src/google/protobuf/compiler/rust/`). It is driven by Bazel `rust_proto_library` or, for Cargo, by a `build.rs` crate that shells out to a protoc whose version must exactly match the crate.

The API is accessor-based:
- `foo()`, `set_foo()`, `has_foo()`, `clear_foo()`, `foo_opt()`, `foo_mut()`;
- `proto!` macro for construction;
- enums are `#[repr(transparent)]` `i32` newtypes with associated constants: open enums get `From<i32>`, closed enums get `TryFrom<i32>`;
- oneofs give a borrowed enum of views plus a `Case` enum;
- unknown fields are preserved by the kernel, with no public Rust API to inspect them;
- editions are supported through protoc.

### Cited Findings
- Generated API:
  - explicit presence: `has_foo()`, `foo()`, `foo_opt()`, `set_foo()`, `clear_foo()`; implicit presence: only `foo()`/`set_foo()`;
  - strings: `&ProtoStr` getters, setters take `impl IntoProxied<ProtoString>`;
  - sub-messages: `foo()` returns `BarView<'_>`, `foo_mut()` returns `BarMut<'_>` and sets presence;
  - repeated: `RepeatedView`/`RepeatedMut`; maps: `MapView`/`MapMut`;
  - packed vs expanded via the editions feature `repeated_field_encoding`;
  - `Any` gets no special handling; arenas and services have no Rust API.
  - Source: [protobuf.dev Rust Generated Code Guide](https://protobuf.dev/reference/rust/rust-generated/)
- Oneof codegen comment: each oneof emits `pub enum SomeOneof<'msg> { FieldA(i32) = 7, FieldB(View<'msg, SomeMsg>) = 9, not_set(PhantomData<&'msg ()>) = 0 }` and a `#[repr(C)] pub enum SomeCase { FieldA = 7, FieldB = 9, not_set = 0 }`, with accessors `some_oneof()` and `some_oneof_case()` on owned, Mut and View — [PB src/google/protobuf/compiler/rust/oneof.cc L30-70](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/src/google/protobuf/compiler/rust/oneof.cc#L30-L70)
- Enum open/closed split:
  - closed enums: `impl TryFrom<i32> for E { type Error = UnknownEnumValue<Self>; ... if is_known(val) { Ok(Self(val)) } else { Err(...) } }`; open enums: `impl From<i32> for E { fn from(val) -> E { Self(val) } }`;
  - the default value is the first listed value;
  - the `Enum` trait doc: "If `Self` is a closed enum, then `TryFrom<i32>` for `value` succeeds if and only if [`is_known`] returns `true`";
  - `EnumValueNames::name()` returns `None` for unknown numbers and `from_name` accepts aliases.
  - Sources: [PB compiler/rust/enum.cc L283-320](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/src/google/protobuf/compiler/rust/enum.cc#L283-L320), [PB rust/enum.rs](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/enum.rs)
- Closed enums under upb also get a generated `MiniTable` built from a mini-descriptor string, via `OnceLock` and `unsafe` `build_enum_mini_table`. This is how upb routes unknown closed-enum values — [PB compiler/rust/enum.cc L90-109](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/src/google/protobuf/compiler/rust/enum.cc#L90-L109)
- Generated messages carry `unsafe impl Sync`/`Send` for `$Msg$`, `$Msg$View` and `$Msg$Mut`, with safety comments (for example "`$Msg$` is `Send` because it uniquely owns its arena") — [PB compiler/rust/message.cc L605-744](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/src/google/protobuf/compiler/rust/message.cc#L605-L744)
- Public runtime traits:
  - `Parse`, `Serialize`, `Clear`, `ClearAndParse`, `CopyFrom`, `MergeFrom`, `TakeFrom`;
  - `Message`, `MessageMut`, `MessageView`, `MessageType`;
  - `AsView`/`AsMut`/`IntoView`/`IntoMut`/`IntoProxied`;
  - `WithReflection` (non-lite only: just `full_name()`).
  - `__internal` "is not considered a semver breaking change", and is hidden only by convention in Cargo.
  - Sources: [PB rust/shared.rs L17-64](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/shared.rs#L17-L64), [PB rust/codegen_traits.rs L74-79](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/codegen_traits.rs#L74-L79)
- Unknown fields: no public Rust API exists (`grep -i unknown rust/*.rs` matches only `UnknownEnumValue` and a gtest matcher note about "false-negatives or false-positives in the face of unknown fields"). Preservation is a kernel (upb/C++) behavior. The text encoder has a `SKIP_UNKNOWN` option — [PB rust/gtest_matchers_impl.rs L18](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/gtest_matchers_impl.rs#L18), [PB rust/upb/text.rs](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/upb/text.rs)
- Cargo build:
  - `google-protobuf-codegen` exposes `CodeGen::new().input(..).include(..).dependency(..).protoc_path(..).generate_and_compile()`;
  - it runs `protoc --version` and errors unless it equals the version derived from the crate ("Expected protoc version {} but found {}");
  - `PROTOC` env overrides the path.
  - A user asked to relax this or auto-download protoc (#26271, open); Google said "This strictness is temporary".
  - Sources: [PB rust/release_crates/google_protobuf_codegen/src/lib.rs L45-170](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/google_protobuf_codegen/src/lib.rs#L45-L170), [protobuf #26271](https://github.com/protocolbuffers/protobuf/issues/26271)
- Bazel: `rust_proto_library` picks the kernel by a global flag; "Don't use `rust_upb_proto_library` or `rust_cc_proto_library` directly." — [protobuf.dev Building Rust Protos](https://protobuf.dev/reference/rust/building-rust-protos/)
- Name collisions: a message named `StreamView` collides with the generated view type of a message named `Stream` (#25736, 4.34.0-rc1). Google: "two name collisions unfortunately exist in every one of our language impls"; the issue was put on an internal backlog — [protobuf #25736](https://github.com/protocolbuffers/protobuf/issues/25736)

### Inferences
- `TryFrom` for closed and `From` for open enums, with an `i32` newtype and associated constants, is a clean, spec-correct enum model RustPB could copy. It is a one-type-per-enum alternative to rust-protobuf's `E` + `EnumOrUnknown<E>` pair, at the cost of no exhaustive `match`.
- RustPB's ScalaPB-style "sealed oneof as Rust enum" with owned payloads is simpler than v4's view enum plus `Case` enum.
- Generated `Foo`/`FooView`/`FooMut` names can collide with user message names. Plain structs avoid that whole class of collisions.

### Gaps
- Editions support level in the Rust generator was not checked feature by feature. The docs mention the editions feature `repeated_field_encoding` and presence via editions.

## Q10. Google v4: limitations (pure Rust/wasm, FFI, JSON, reflection, maturity) and user criticism

### Takeaway
v4 requires a C toolchain and FFI:
- `google-protobuf` declares `links = "upb"` and its `build.rs` compiles `upb.c` with `cc`;
- the runtime uses `unsafe` extensively at the kernel boundary.

The JSON and full API are missing. A Google comment from Dec 2025 says Rust supports only the "lite" API, with the full API intended in 2026. The conformance runner skips all non-binary tests.

Criticism focuses on:
- C dependency and build friction (exact protoc version, Bazel issues);
- verbose, non-idiomatic accessor-based code;
- sparse documentation;
- an unstable API and crate churn.

wasm32-unknown-unknown requires compiling C to wasm. No official pure-Rust or JS path exists.

### Cited Findings
- `google-protobuf` Cargo template: `links = "upb"`, `[build-dependencies] cc`, deps `linkme` and `paste-complete`, `rust-version = "1.79"`. `build.rs` runs `cc::Build::new().flag("-std=c99")...file("libupb/upb/upb.c").file("libupb/third_party/utf8_range/utf8_range.c")...compile("libupb")` — [PB google_protobuf/Cargo-template.toml](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/google_protobuf/Cargo-template.toml), [PB google_protobuf/build.rs](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/google_protobuf/build.rs)
- `unsafe` occurrences counted by grep in the runtime: `rust/upb/message.rs` 49, `rust/repeated.rs` 26, `rust/upb/arena.rs` 15, `rust/upb/reflection.rs` 14. The runtime opens with `#![deny(unsafe_op_in_unsafe_fn)]` — [PB rust/shared.rs L8](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/shared.rs#L8), [PB rust/upb](https://github.com/protocolbuffers/protobuf/tree/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/upb)
- The design page lists as limitations: "C-based kernels currently require a C toolchain such as Clang at build time"; C++ zero-copy sharing only with ABI-compatible C++ gencode; `ProtoStr` may contain invalid UTF-8 — [protobuf.dev Rust Proto Design Decisions](https://protobuf.dev/reference/rust/rust-design-decisions/)
- JSON. A feature request for ProtoJSON (`serialize_json`/`parse_json`) was closed. Google (2025-12-23): "Rust proto is only supporting lite.api today. We intend to support the full api in 2026." — [protobuf #24938](https://github.com/protocolbuffers/protobuf/issues/24938)
- Conformance runner for Rust:
  - `if req.requested_output_format() != WireFormat::Protobuf { resp.set_skipped("only wire format output implemented") }` and "only wire format input implemented";
  - the expected-failure lists are tiny: `failure_list_rust_cc.txt` has 1 active entry, `Required.*.ProtobufInput.BadTag_FieldNumberSlightlyTooHigh`, and `failure_list_rust_upb.txt` has 0 active entries;
  - both lists carry commented-out `RejectInvalidUtf8` items for proto2 "not expected to be fixed".
  - Sources: [PB conformance/conformance_rust.rs L61-69](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/conformance/conformance_rust.rs#L61-L69), [PB conformance/failure_list_rust_cc.txt](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/conformance/failure_list_rust_cc.txt), [PB conformance/failure_list_rust_upb.txt](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/conformance/failure_list_rust_upb.txt)
- Reflection: `WithReflection` exposes only `full_name()`. Text output goes through upb's `upb_TextEncode`/`upb_DebugString` over FFI — [PB rust/codegen_traits.rs L74-79](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/codegen_traits.rs#L74-L79), [PB rust/upb/reflection.rs](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/upb/reflection.rs)
- Bazel friction issues:
  - #24256 "Rust Protobuf crate does not compile in bazel without `--spawn-strategy=standalone`" (2025-11);
  - #29090 `rust_proto_library` fails in downstream Bzlmod builds (2026-08);
  - #30684 "Non-Rust consumers inherit Rust compiler downloads via Protobuf" (open, 2026-10);
  - #23626 "Compilation error in Rust with the Rust 2024 Edition" (2025-09).
  - Source: [protobuf issues](https://github.com/protocolbuffers/protobuf/issues?q=label%3Arust)
- A user on #25736: "I find the Rust Protobuf landscape very confusing" — [protobuf #25736](https://github.com/protocolbuffers/protobuf/issues/25736)
- A DEV Community article (2026, single author's opinion) says Google's crate relies on a C++-centric codegen pipeline, that output is verbose and boilerplate-heavy, documentation sparse and the roadmap opaque — [DEV Community article](https://dev.to/serbyte/rusts-protocol-buffers-ecosystem-in-flux-prost-unmaintained-googles-protobuf-crate-faces-2m32)
- `PartialEq` is not implemented on messages (see Q8). This is an ergonomic gap noted by a gRPC developer: "we also can't implement `PartialEq` ourselves" (orphan rule) — [protobuf #23058](https://github.com/protocolbuffers/protobuf/issues/23058)
- The Bazel lockfile lists `wasm32-unknown-unknown` and `wasm32-wasip1` platforms, and a tree-shaking test script has a `wasm` branch. This suggests some wasm build path inside Google's tooling, but no documented wasm support was found — [PB rust/Cargo.Bazel.lock L1317-1321](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/Cargo.Bazel.lock#L1317-L1321), [PB rust/test/treeshaking/verify_except_tables.sh L19](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/test/treeshaking/verify_except_tables.sh#L19)

### Inferences
- v4 cannot serve a Rust-to-JS compiler target. The whole runtime is FFI to C (upb), opaque arena pointers and `unsafe`, which a source-level Rust-to-JS compiler cannot lower.
- For RustPB's JS target, rust-protobuf's pure-Rust approach is the only relevant precedent of the two. Even it relies on `unsafe` in its input stream and reflection, which RustPB should avoid.
- v4's conformance number means "binary-only, and the C kernel is conformant". It says nothing about a Rust implementation's correctness, because the Rust layer does no parsing.

### Gaps
- No Reddit threads or broad user surveys were found. Criticism evidence is limited to GitHub issues, Google Groups and one blog article.
- Whether `google-protobuf` builds for `wasm32-unknown-unknown` through Cargo (C to wasm via clang) is untested here.

## Q11. Ideas worth copying vs pitfalls to avoid for a plain-struct RustPB

### Takeaway
Ideas to copy:
- pure-Rust `.proto` parsing so codegen needs no protoc (rust-protobuf);
- open/closed enum split with `TryFrom` vs `From` (v4);
- an `EnumOrUnknown`-style type for open enums (rust-protobuf);
- one `special_fields`-style slot for unknown fields;
- recursion limit 100 matching C++, applied to every recursive path including groups and unknown skipping;
- `_future_options`-style non-exhaustive option structs;
- `proto!`-style construction;
- conformance-runner discipline.

Pitfalls to avoid:
- no typed customization (rust-protobuf's only hook is free-text `before`);
- replace-instead-of-merge for repeated singular messages;
- closed enums treated as open;
- size/write divergence for negative values;
- unknown fields stored unordered;
- JSON tied to heavy reflection with `Any` unimplemented;
- `Deref` to defaults hiding presence;
- interior-mutable `CachedSize`;
- dual `rustproto` option plus callback paths;
- `unsafe` in hot paths;
- for v4, opaque proxies, the C toolchain, exact-version protoc coupling and crate churn.

### Cited Findings
- Each item is sourced in Q2-Q10 above. The key ones:
  - pure parser: [RP which_parser.rs](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-parse/src/which_parser.rs);
  - enums: [PB enum.cc L283-320](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/src/google/protobuf/compiler/rust/enum.cc#L283-L320);
  - CVE: [RUSTSEC-2024-0437](https://rustsec.org/advisories/RUSTSEC-2024-0437.html);
  - merge: [RP rt/message.rs L8-19](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/rt/message.rs#L8-L19);
  - enum size bug: [rust-protobuf #766](https://github.com/stepancheg/rust-protobuf/issues/766);
  - no type mapping: [rust-protobuf #552](https://github.com/stepancheg/rust-protobuf/issues/552);
  - Any JSON: [RP print.rs L42](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf-json-mapping/src/print.rs#L42);
  - v4 FFI: [PB google_protobuf/build.rs](https://github.com/protocolbuffers/protobuf/blob/8e2636c3aaeedc3cd22963814e7727564a017bfc/rust/release_crates/google_protobuf/build.rs).
- The two different stances on equality are worth recording:
  - rust-protobuf v3 made `Message: PartialEq` mandatory and `CachedSize` "always equal to itself" [RP cached_size.rs](https://github.com/stepancheg/rust-protobuf/blob/0118c79e51c8d54ac40deb8b82631ddef81fb00e/protobuf/src/cached_size.rs);
  - Google refuses `PartialEq` because unknown fields cannot be canonicalized [protobuf #23058](https://github.com/protocolbuffers/protobuf/issues/23058).

### Inferences
These are design suggestions derived from the evidence, not sourced facts.
- **TypeMapper:** neither project has one. RustPB's TypeMapper would be novel in the Rust ecosystem. It should be a typed trait, for example `TypeMapper<Base, Custom>` with `to_custom`/`to_base`, applied in the generated field type, codec and JSON. It should not be a text injection, so that serde-like attributes do not need hand-written adapters, as they did in rust-protobuf's serde example.
- **Unknown fields:** an ordered `Vec<u8>` or an ordered list of `(tag, raw bytes)` gives byte-exact round-trips. It is simpler than rust-protobuf's `HashMap<u32, UnknownValues>`.
- **Equality:** deriving `PartialEq` that includes unknown bytes is consistent with ScalaPB, which includes `unknownFields` in case-class equality. Google's caveat, that equality differs across schema versions, should be documented.
- **Cached sizes:** avoiding a cached size inside the struct (computing sizes into a side stack, as rust-protobuf did before 2014) keeps structs plain and free of atomics, which also helps the JS target. The trade-off is quadratic size computation for deep nesting unless a side buffer is used.
- **Conformance:** binary and JSON conformance runs should be required for both native and JS outputs from the start. Tests should cover the specific historical failures: merge of repeated sub-messages, closed-enum routing, negative enum/int32 10-byte varints, group recursion depth, and invalid UTF-8 rejection in proto3.

### Gaps
- No head-to-head performance numbers for rust-protobuf vs v4 vs prost were found in this pass.
