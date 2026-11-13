# Alternative and newer Rust protobuf implementations: prior art for RustPB (custom type mapping focus)

Scope: quick-protobuf/pb-rs, micropb, femtopb, pilota/pilota-build, buffa, plus newer entrants found in 2024-2026 (gin-tonic, pbrs, connectrpc, bilrost, proto_rs). prost and Google's `protobuf` v4 appear only as reference points; other notes cover them.

Method: each repo was shallow-cloned to `~/Documents/GitHub/<name>` and read at these commits:
- buffa `104716922c` (2026-10-20)
- pilota `8239e1e20b` (2026-11-12)
- micropb `0b86edac8e` (2026-10-24)
- femtopb `eaa5326d72` (2026-08-26)
- quick-protobuf `54e7d6c5d9` (2024-02-14)
- gin-tonic `e31fae4f1f` (2026-07-17)

Source links below point at those commits. Download and dependent counts come from the crates.io API and repo stats from `gh api`, all retrieved 2026-11-13.

## Q1. What is each crate, and how does it differ from prost? (Per-crate profiles)

### Takeaway
buffa is now the main pure-Rust alternative to prost. It has editions, unknown fields, open-enum wrappers, views, reflection and full conformance, and it grew fast: 11.4M downloads within 8 months of its first release. The other crates fill niches:
- micropb and femtopb serve no-alloc embedded targets.
- pilota is a Thrift and protobuf generator with a Salsa-based IR, built for the Volo RPC framework.
- quick-protobuf is effectively unmaintained. It still has high transitive use, and it has open memory-safety reports.
- gin-tonic is small, but it is the only crate found whose main idea is a ScalaPB-style "map this wire scalar to my type" feature.

### Cited Findings

#### buffa (maintainer: Anthropic; v0.9.2, released 2026-09-03)
- **What it is.** buffa is a pure-Rust protobuf implementation with "editions as the core abstraction". Its motivation table lists the gaps it fills: prost v0.13 is "Passive" with no editions, no unknown fields and no reflection, and Google protobuf v4 is "No (upb/C++)". — [DESIGN.md L5-19](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L5-L19)
- **Workspace crates.** `buffa` (runtime), `buffa-types` (well-known types), `buffa-descriptor`, `buffa-proto-options`, `buffa-codegen` (a library that takes `FileDescriptorProto` in and gives Rust out), `protoc-gen-buffa` (the primary entry point), `buffa-build` (build.rs), `buffa-remote-derive` and `buffa-yaml`. — [Cargo.toml](https://github.com/anthropics/buffa/blob/104716922c/Cargo.toml), [DESIGN.md L32-145](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L32-L145)
- **No parser of its own.** buffa relies on protoc or buf for parsing and edition-feature resolution, then decodes the descriptors with its own bootstrapped generated types. "The code generator always works with resolved edition features — it never branches on 'is this proto2 or proto3?'" — [DESIGN.md L104, L122, L452-479](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L452-L479)
- **Two-tier output.** Each message produces an owned `Foo` and a zero-copy `FooView<'a>`. `OwnedView<V>` pairs a `Bytes` buffer with a view to give a `'static + Send + Sync` type. An optional `FooLazyView` is also available. — [DESIGN.md L169, L40, L218](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L169)
- **Linear-time encoding.** An external `SizeCache` makes encoding linear: "Prost recomputes message sizes at every nesting level ... potentially exponential". The cache sits outside the structs, so `Send + Sync` holds structurally. — [DESIGN.md L383-401](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L383-L401)
- **Optional message fields.** buffa uses `MessageField<T>`, which "dereferences to a default instance when unset". It stores the message inline by default (`Option<T>`) and boxes only recursive fields or opt-outs. The rationale is that prost's `Option<Box<M>>` "creates unwrapping ceremony". — [DESIGN.md L337-358](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L337-L358)
- **Table-driven codec option.** `CodecStrategy::Table` is opt-in; `Unrolled` stays the default (issue #463). It is a static table of 12-byte entries built with `offset_of!`, and "The `unsafe` lives in `buffa`", so generated code compiles under `#![forbid(unsafe_code)]`. — [DESIGN.md L570-581](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L570-L581)
- **Origin and production use.** The launch post says buffa was built in about six weeks with an AI model doing most of the implementation, that it is "in production" at its maintainer, and that it underlies connect-rust (ConnectRPC). It reports about 33% faster than tonic+prost on a decode-heavy benchmark. — [DEV post, Iain McGinniss](https://dev.to/iainmcgin/zero-copy-protobuf-and-connectrpc-for-rust-1m3e)
- **Adoption.** 11,400,531 total downloads (5,284,421 in the last 90 days); first published 2026-03-14; 212 reverse dependencies; 940 GitHub stars; 148 open issues. — [crates.io/crates/buffa](https://crates.io/crates/buffa), [github.com/anthropics/buffa](https://github.com/anthropics/buffa)
- **Release cadence.** Eleven releases from 0.1.0 (2026-03-07) to 0.9.2 (2026-09-03). 0.9.2 alone took 55 PRs from 12 contributors, "most of them closing gaps in how buffa bounds untrusted input". — [CHANGELOG.md L9-30, L1876](https://github.com/anthropics/buffa/blob/104716922c/CHANGELOG.md#L9)

#### pilota / pilota-build (maintainer: CloudWeGo; pilota 0.13.1, pilota-build 0.13.12)
- **What it is.** "a thrift and protobuf implementation in pure rust with high performance and extensibility." — [README.md](https://github.com/cloudwego/pilota/blob/8239e1e20b/README.md)
- **Runtime origin.** The protobuf runtime is a modified copy of prost: "copied from the Prost project and modified for performance. For zero copy, we use `Bytes` for decode and `LinkedBytes` for encode." The `Message` trait encodes into `LinkedBytes`, a linked list of `Bytes`/`BytesMut`, and decodes from `Bytes`. — [pilota/src/pb/README.md](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota/src/pb/README.md), [pilota/src/pb/message.rs L17-41](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota/src/pb/message.rs#L17-L41), [crates.io/crates/linkedbytes](https://crates.io/crates/linkedbytes)
- **Strings and maps.** Strings default to `::pilota::FastStr`, a cheap-to-clone string from the `faststr` crate (21M downloads), and maps default to `AHashMap`. — [docs/pilota-build-architecture.md L128-171](https://github.com/cloudwego/pilota/blob/8239e1e20b/docs/pilota-build-architecture.md#L128-L171), [crates.io/crates/faststr](https://crates.io/crates/faststr)
- **Parsing.** .proto files are parsed with rust-protobuf's pure-Rust `protobuf-parse` (`pilota-build/Cargo.toml` L51). Issue #3, "Switch to the official protobuf-parse", has been open since 2022. — [pilota-build/Cargo.toml](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/Cargo.toml#L51), [issue #3](https://github.com/cloudwego/pilota/issues/3)
- **Adoption.** pilota has 637,268 downloads (200,822 in 90 days) and 15 reverse dependencies; pilota-build has 229,219. volo-grpc has 42,230 downloads. The repo has 169 stars and its last push was 2026-11-12. — [crates.io/crates/pilota](https://crates.io/crates/pilota), [crates.io/crates/volo-grpc](https://crates.io/crates/volo-grpc)

#### micropb (maintainer: Yuhan Lin; v0.7.0, released 2026-10-24)
- **What it is.** micropb targets "constrained environments where no allocator is available". It supports no-std and no-alloc, statically or dynamically allocated containers, "Type-safe open enums, rather than `i32`", "custom handlers with user-defined encoding and decoding behaviour" and encode length caching. — [README.md L8-20](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/README.md#L8-L20)
- **Limitations.** No extensions, "Does not support closed enums", no reflection, no RPC (services do have codegen extension points). — [README.md L22-27](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/README.md#L22-L27)
- **Editions.** Only `field_presence` and `repeated_field_encoding` are supported. `enum_type`, `message_encoding` and `utf8_validation` are ignored. — [README.md](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/README.md)
- **Toolchain.** micropb needs `protoc` (it reads `PROTOC` or PATH). 0.7.0 added editions, a CLI, `ServiceGenerator`, wire-type checks and a nesting limit of 100 that fixes a stack-overflow DoS (issue #44). — [micropb-gen/src/lib.rs L35, L989-1005](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/lib.rs#L989), [CHANGELOG.md L1-20](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/CHANGELOG.md)
- **Adoption.** 394,212 downloads (99,471 in 90 days), 7 reverse dependencies, 145 stars, 1 open issue. — [crates.io/crates/micropb](https://crates.io/crates/micropb)

#### femtopb (maintainer: David Flemström; v0.9.0, released 2026-08-23)
- **What it is.** "A tiny footprint, `#[no_std]`, no-alloc, no-panic Protobuf serialization library". Messages decode only from a contiguous `&[u8]` and borrow it. Repeated fields decode lazily. There is no reflection and no well-known types, and the README says "It is probably advisable to use `prost` ... where it is possible". — [README.md](https://github.com/dflemstr/femtopb/blob/eaa5326d72/README.md)
- **Code generation.** `femtopb-build` runs `protox`, a pure-Rust protobuf compiler, and `prost-build`. It then rewrites prost's output with `syn`: `#[prost(..)]` attributes become `#[femtopb(..)]`, and `Box<T>` becomes `Deferred<'a, T>`. — [build/src/lib.rs L3-4, L195-264, L663](https://github.com/dflemstr/femtopb/blob/eaa5326d72/build/src/lib.rs#L195-L264)
- **Adoption.** 84,022 downloads (15,100 in 90 days), 42 stars. — [crates.io/crates/femtopb](https://crates.io/crates/femtopb)

#### quick-protobuf / pb-rs (maintainer: tafia; quick-protobuf 0.8.1 and pb-rs 0.10.0, both last released 2022-11-22)
- **Type mapping.** pb-rs maps `bytes` to `Cow<'a,[u8]>`, `string` to `Cow<'a,str>`, `optional` to `Option`, enums to Rust `enum` and a oneof to an `OneOfName` enum. It uses its own `nom` parser, so "no need to use google `protoc`". — [README.md L14-30](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/README.md#L14-L30), [pb-rs/Cargo.toml](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/pb-rs/Cargo.toml)
- **Rationale against rust-protobuf.** It claims to be "Much faster ... no extra allocation" and to produce generated modules about 10 times smaller. It concedes "you have to handle more things yourself (e.g. `Option` unwrapping, `Cow` management)". — [README.md L222-245](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/README.md)
- **Adoption.** 36,829,661 downloads (2,885,334 in 90 days) and 62 reverse dependencies. Most use is transitive, for example through rust-libp2p (see issue #263). The repo was last pushed 2024-02-14 and has 65 open issues. — [crates.io/crates/quick-protobuf](https://crates.io/crates/quick-protobuf), [issue #263](https://github.com/tafia/quick-protobuf/issues/263)

#### Newer entrants (2024-2026)
- **gin-tonic** (cemoktra, Codeberg; v0.8.9, 2026-06-22)
  - A "prost alternative" with its own generator replacing prost-build, a tonic codec, and a `Scalar` trait that "maps any Rust type to a protobuf wire type".
  - Announced on the Rust forum on 2026-05-17.
  - 20,350 downloads, only 39 in the last 90 days.
  - [forum post](https://users.rust-lang.org/t/try-out-gin-tonic-prost-alternative/140124), [crates.io/crates/gin-tonic](https://crates.io/crates/gin-tonic)
- **pbrs** (mingley; v0.2.1, 2026-11-13; first release 2026-09-05)
  - Provides "the Google protobuf v4 application API" from a pure-Rust runtime and codegen, plus a `protoc-gen-pbrs` plugin.
  - Claims conformance runner v35.1 passes 5,631 binary and JSON cases and 909 text cases with no unexpected failures. Edition 2024 support is a "defined subset".
  - 654 downloads.
  - [docs.rs/crate/pbrs](https://docs.rs/crate/pbrs/latest), [crates.io/crates/pbrs](https://crates.io/crates/pbrs)
- **connectrpc** ("A Tower-based Rust implementation of the ConnectRPC protocol"; v0.9.1)
  - 9,673,900 downloads and 74 reverse dependencies. Built on buffa.
  - [crates.io/crates/connectrpc](https://crates.io/crates/connectrpc), [DEV post](https://dev.to/iainmcgin/zero-copy-protobuf-and-connectrpc-for-rust-1m3e)
- **bilrost** is a prost fork with its own encoding. It is not protobuf wire-compatible, so it is out of scope for byte compatibility. — [github.com/mumbleskates/bilrost](https://github.com/mumbleskates/bilrost)
- **proto_rs** ("Rust-first gRPC macros collection") has 4,448 downloads. **protokit** last released in 2024 and has 10,863 downloads. Both are marginal. — [crates.io/crates/proto_rs](https://crates.io/crates/proto_rs), [crates.io/crates/protokit](https://crates.io/crates/protokit)
- **protox** is a pure-Rust protobuf compiler that produces descriptors without protoc: v0.10.0, 20.6M downloads. femtopb uses it. — [crates.io/crates/protox](https://crates.io/crates/protox)

### Inferences
- Since its 2026 launch, buffa has replaced quick-protobuf and pilota as the reference point for a modern pure-Rust design. Its design document is the richest single source of reasoned decisions (sections 3-13). RustPB should treat it as the main prior art and position itself on the one axis buffa has not finished: semantic type mapping (see Q2).
- Its download count is driven by connectrpc and transitive use. That shows momentum, but not 8 months of hardening; the 0.9.2 security-focused release suggests the edges were still being found.

### Gaps
- No public adoption numbers for pilota in production beyond crates.io. I did not verify the Volo user list.
- I did not verify the year of the buffa DEV post; the page shows only "Mar 25", which matches the 2026-03 first release.
- I did not clone or read pbrs; its claims come from docs.rs only.

## Q2. Per-field custom types: who offers them, how they are configured, and how conversion failures are handled (most important for RustPB)

### Takeaway
None of the established crates has a general ScalaPB TypeMapper. That means a field-level `wire type -> domain type` mapping with conversion functions, applied to scalars and messages alike, carried through binary, JSON and reflection.

The existing designs fall into four groups:
- **(a) Representation swaps.** The storage type changes but the meaning does not, e.g. `String` to `SmolStr`. buffa `string_type/bytes_type/...`, pilota `pilota.rust_type`, micropb `string_type/vec_type`.
- **(b) Whole-message substitution.** `extern_path` plus a hand-written `Message` impl. buffa, and prost-style generators in general.
- **(c) Wire-level custom field handlers.** micropb `custom_field`.
- **(d) Semantic scalar mapping via a proto option plus a trait with a fallible decode.** gin-tonic's `(gin_tonic.v1.rust_type)` with `Scalar<ProtoString>`.

buffa has an open, maintainer-endorsed proposal (#490) for (d), using `buffa.ext.field.custom_type{type, adapter}`.

### Cited Findings

#### buffa: representation knobs (category a)
- **Configuration.** `buffa_build::Config` has per-path-prefix knobs:
  - `string_type_in` / `string_type_custom_in` (L1381, L1437)
  - `bytes_type_in` / `bytes_type_custom` (L1247, L1289)
  - `map_type_custom` (L1568)
  - `box_type_custom` (L1716)
  - `repeated_type_custom` (L1803), where templates use `*` as a placeholder
  - `extern_path` (L1105)
  - plus attribute injectors `type_attribute`, `field_attribute`, `message_attribute`, `enum_attribute`, `oneof_attribute` (L1870-2166)

  — [buffa-build/src/lib.rs](https://github.com/anthropics/buffa/blob/104716922c/buffa-build/src/lib.rs#L1105)
- **Contract.** The `ProtoString`/`ProtoBytes` contract is `fn from_wire(WirePayload<'_>) -> Result<Self, DecodeError>`, so a failed conversion surfaces as a decode error. Supertraits are `Clone + PartialEq + Default + Debug + Send + Sync` plus `Deref<Target=str>`, `AsRef<str>` and `From<String>`. "The `Deref` bound is load-bearing": generated code borrows `&self.field` as `&str`. — [DESIGN.md L546](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L546)
- **Design reversal.** "The earlier design *was* a blanket impl over `From<String>` / `From<Vec<u8>>`; it was tightened to `from_wire` because the blanket path always pays an allocate-and-copy". — [DESIGN.md L546](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L546)
- **`WirePayload`.** It exposes `as_slice()`, `to_str()` (UTF-8 validated), `into_bytes()` and `is_owned()`. `ProtoBytes::PREFERS_OWNED_PAYLOAD` lets a refcounted type share the input `Bytes`. Clearing resets a field to `Default::default()` "because a substituted type may be immutable". — [DESIGN.md L548-550](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L548-L550)
- **Orphan-rule friction.** A foreign type must be wrapped in a local newtype. When the custom type is a `repeated` element or a `map<K, bytes>` value, codegen emits `impl ReflectElement` / `impl ProtoElemJson` for it, so it must be crate-local: "an emitted `impl ForeignTrait for ForeignType` only compiles when the element type is local". — [DESIGN.md L552](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L552)
- **Remote derive.** `buffa-remote-derive` provides `#[buffa(remote = ..., serde)]` derives (`ProtoString`, `ProtoBytes`, `ProtoList`, `ProtoBox`, `MapStorage`) to cut that "newtype tax", "mirroring serde's `remote` attribute pattern". — [CHANGELOG.md L242-246](https://github.com/anthropics/buffa/blob/104716922c/CHANGELOG.md#L242), [issue #212](https://github.com/anthropics/buffa/issues/212)
- **JSON pitfalls.** JSON requirements differ by knob and position. For example, a custom string with explicit presence, in a repeated field, oneof or map, "serializes through the element's native `serde`" and must implement `Serialize`/`Deserialize`, while implicit-presence singular strings go through a with-module. The example README calls this "the most common stumbling point". — [examples/custom-types/README.md](https://github.com/anthropics/buffa/blob/104716922c/examples/custom-types/README.md), [docs/guide.md L387-389](https://github.com/anthropics/buffa/blob/104716922c/docs/guide.md#L387)
- **Representation belongs to the defining crate.** "The representation is a property of a type definition, fixed once by whichever crate generates that message — never a property of the reference site". `string_type` never reaches inside an `extern_path` type. — [DESIGN.md L554-562](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L554-L562)

#### buffa: whole-message substitution (category b)
- "implement the `Message` trait by hand and use `extern_path` ... **This is rare**". The guide's worked example maps `Int64Range` to `std::ops::Range<i64>` through a newtype with a hand-written `compute_size` / `write_to` / `merge_field`. — [DESIGN.md L147-149](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L147-L149), [docs/guide.md L2720-2790](https://github.com/anthropics/buffa/blob/104716922c/docs/guide.md#L2720)
- **Bug, fixed.** "`extern_path` only matches package prefixes — per-type FQN mappings are silently ignored". — [issue #111](https://github.com/anthropics/buffa/issues/111)

#### buffa: open proposal for semantic scalar types (category d)
- **The request.** Issue #490 (2026-09-28, open) asks for `int32 id` to become `::my_types::DummyId` and `double mass` to become `uom::si::f64::Mass` "while preserving their original wire and ProtoJSON representations". It notes that "`extern_path` can replace protobuf message or enum types, but it cannot replace an individual scalar field's Rust representation", and that wrapping the scalar in a message would change the wire format. The proposed `ScalarAdapter<i32> { type Value; fn from_wire(i32) -> Value; fn to_wire(&Value) -> i32 }` is **infallible**. — [issue #490](https://github.com/anthropics/buffa/issues/490)
- **Maintainer response.** The maintainer (2026-10-01) found it "reasonable". He proposed options under a `buffa.ext` package: `[(buffa.ext.field.custom_type) = { type: "...", adapter: "..." }]`, or a named `custom_type_ref` registry. He noted that making the registry work as protoc plugin arguments is unresolved, and reserved an extension range in the protobuf registry (protocolbuffers/protobuf PR #30391). He declined to make it cross-language: "I'm wary of building something here that would claim to be applicable for other languages". — [issue #490 comments](https://github.com/anthropics/buffa/issues/490)
- **Option plumbing is in place but unused.** `buffa-proto-options` already declares `(buffa.ext.file|message|field|oneof|enum|enum_value)`, each with extension number 1381 (range 1381-1390 reserved). At this commit every option message is still **empty**: "An empty option message leaves the generated code unchanged." — [buffa-proto-options/protos/buffa/ext/options.proto](https://github.com/anthropics/buffa/blob/104716922c/buffa-proto-options/protos/buffa/ext/options.proto), [DESIGN.md L78-86](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L78-L86)

#### gin-tonic: proto option plus `Scalar` trait (category d, the closest to ScalaPB)
- **The option.** `extend google.protobuf.FieldOptions { optional string rust_type = 10250; }` in package `gin_tonic.v1`. A field is written `string my_uuid = 1 [(gin_tonic.v1.rust_type) = "uuid::Uuid"];`. — [gin/proto/gin.proto](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/gin/proto/gin.proto), [forum post](https://users.rust-lang.org/t/try-out-gin-tonic-prost-alternative/140124)
- **The trait.** `pub trait Scalar<ProtobufType> { const WIRE_TYPE: u8; fn encode(&self, &mut impl Encode); fn decode(&mut impl Decode) -> Result<Self, ProtoError>; }` is parameterized by a marker for the proto scalar type (`scalars::ProtoString`, `scalars::Bytes`, ...). One Rust type can therefore have several wire mappings: `Uuid` implements both `Scalar<Bytes>` and `Scalar<ProtoString>`. Parse failures map to `ProtoError::Custom(err.to_string())`. — [core/src/traits.rs L3-23](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/core/src/traits.rs#L3-L23), [core/src/types/uuid.rs L1-33](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/core/src/types/uuid.rs#L1-L33)
- **The string mapping changes bytes.** `impl Scalar<ProtoString> for Uuid` encodes with `self.as_simple().to_string()`, the 32-hex-digit form without hyphens. A canonical hyphenated UUID string read from the wire is therefore re-encoded in a **different textual form**. — [core/src/types/uuid.rs L21](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/core/src/types/uuid.rs#L21)
- **Codegen detail.** The generator reads the option through the descriptor pool. If present, it parses the string as a `syn::Type` with `.expect("Invalid path")` (marked `// TODO Better error message.`) and returns it **before** cardinality is considered. Well-known types are mapped by an `ExternalType` list, e.g. `.google.protobuf.Int32Value` becomes `i32`. — [gin/src/codegen/utils.rs L234-240](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/gin/src/codegen/utils.rs#L234-L240), [gin/src/codegen/external_type.rs](https://codeberg.org/cemoktra/gin-tonic/src/commit/e31fae4f1f63811b119c3efac805f10a85c600f5/gin/src/codegen/external_type.rs)
- **Benchmark claim.** The author reports roughly twice prost's encode speed when prost's numbers include the `From` conversions to idiomatic types (1.10 µs vs 2.35 µs). — [forum post](https://users.rust-lang.org/t/try-out-gin-tonic-prost-alternative/140124)

#### pilota: `pilota.rust_type` is a closed list of representation choices (category a)
- **The options.** `pilota.proto` declares field options `rust_type = 1215204`, `rust_wrapper_arc_field = 1215203`, `serde_attribute_field`, `name_field`, `optional_repeated`, plus message, enum and service variants. Thrift uses the same keys as annotations (`pilota.rust_type`). — [pilota-build/proto/pilota.proto L43-71](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/proto/pilota.proto#L43-L71), [pilota-build/src/tags.rs L135-183](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/tags.rs#L135-L183)
- **What `rust_type` accepts.** `Resolver::modify_ty_by_tags` honours only the fixed strings `"string"` (`FastStr` becomes `String`), `"vec"` (`Bytes` becomes `Vec<u8>`), `"btree"` (maps and sets become BTree versions) and `"ordered_f64"`. There is no arbitrary user type and no conversion function. — [pilota-build/src/resolve.rs L290-330, tests L985-1060](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/resolve.rs#L290-L330)
- **Open issues.** "Allow `pilota.rust_type` annotation work on container type" ([#125](https://github.com/cloudwego/pilota/issues/125), open since 2023) and "Make `pilota.rust_wrapper_arc` annotation wrapper Arc outside" ([#126](https://github.com/cloudwego/pilota/issues/126)).

#### micropb: config-based container types plus wire-level custom handlers (categories a and c)
- **Config keys.** `Config` has `string_type`, `bytes_type`, `vec_type`, `map_type` (templates with `$N`, `$T`, `$K`, `$V`), `max_len`, `max_bytes`, `int_size` (shrink e.g. `int32` to `i8`), `optional_repr` (Hazzer, Option or None), `boxed`, `custom_field`, `unknown_handler`, `rename_field`, `skip`, `enum_int_size`, `enum_unsigned` and attributes. — [micropb-gen/src/config.rs L180-520](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/config.rs#L180-L520)
- **How config is applied.** By proto path, `generator.configure(".pkg.Msg.field", Config::new()...)`. Settings inherit down the path tree unless marked `[no_inherit]`. Per-package TOML files are loaded with `parse_config_file`; issue #23 asked for "nanopb-style options files". — [micropb-gen/src/lib.rs L726-790](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/lib.rs#L726), [issue #23](https://github.com/YuhanLiin/micropb/issues/23)
- **`CustomField`.** `Type(String)` means "Fully-qualified type name that replaces the generated type of the field. This type must implement `FieldEncode` and `FieldDecode`". `Delegate(String)` has another field's handler encode and decode this field too, for multi-field custom types. A custom field "overrides every other config option that affects the field's generated type"; "the custom type is responsible for tracking field presence". — [micropb-gen/src/config.rs L59-83, L380-404](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/config.rs#L59-L83)
- **The handler traits work on raw wire data.** `FieldDecode::decode_field(&mut self, tag, &mut PbDecoder<R>) -> Result<bool, DecodeError<R::Error>>` returns false for unrecognized tags. `FieldEncode` has `const MAX_SIZE: Result<usize,&str>`, `encode_fields` (writes tags too) and `compute_fields_size`. — [micropb/src/field.rs L49-63, L123-139](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb/src/field.rs#L49-L63)

#### femtopb and quick-protobuf: no per-field custom types
- **femtopb.** `femtopb-build::Config` exposes only `target`, `protos`, `includes` and `derive_defmt`. — [build/src/lib.rs L165-194](https://github.com/dflemstr/femtopb/blob/eaa5326d72/build/src/lib.rs#L165-L194)
- **pb-rs.** Flags cover derives and representation only: `--custom_struct_derive`, `--custom_repr`, `--dont_use_cow`, `--owned`, `--nostd`, `--hashbrown`, `--generate-getters`. Open requests: [#267 "Missing support for custom enum derives"](https://github.com/tafia/quick-protobuf/issues/267), [#212 custom derive not applied to oneof enums](https://github.com/tafia/quick-protobuf/issues/212). — [pb-rs/src/main.rs](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/pb-rs/src/main.rs)

### Inferences
- **Two error policies are in use**: fallible decode (buffa `from_wire -> Result<_, DecodeError>`, gin-tonic `Result<_, ProtoError::Custom(String)>`, micropb's `FieldDecode`) and infallible adapters (the buffa #490 proposal). ScalaPB's TypeMapper is infallible by convention, with exceptions on failure. For RustPB, a fallible `TryFrom`-style decode that maps into the decode error, plus an infallible encode, matches what working Rust crates converged on. Stringified error payloads (gin-tonic) lose structure; a typed error would be better.
- **Byte-compatibility risk.** gin-tonic's UUID mapping is a concrete warning: a semantic mapping can break byte-for-byte round-trips. The `as_simple()` re-encoding (L21) turns a hyphenated UUID into a different string. RustPB should either (1) require mappers to be bijective on the values they accept, with property tests for `encode(decode(b)) == b`, or (2) document the lossy case. This is an inference from the code; I found no gin-tonic issue about it.
- **Copy buffa's keying rule.** Key the mapping by proto field path (buffa, micropb) or by proto option (gin-tonic, pilota, the buffa #490 proposal). Following buffa's rule that the mapping belongs to the *defining* crate, never to the reference site, avoids two Rust types for one proto message.
- **The orphan rule shapes everything.** Generated code that emits `impl LibTrait for UserType` (for reflection or JSON elements) forces user types to be crate-local. RustPB should design the adapter as a separate zero-sized type (`impl TypeMapper<Wire, Domain> for MyMapper`, as in the #490 proposal), not as trait impls on the domain type. That sidesteps the orphan rule for foreign domain types like `uuid::Uuid` and `chrono::DateTime`. Note that gin-tonic only gets away with `impl Scalar<..> for uuid::Uuid` because the impl lives inside gin-tonic-core.
- **Message-to-domain mapping** (Timestamp to a time type) exists today only as whole-message substitution with a hand-written `Message` impl (buffa's Range example). None of the crates map a *message-typed field* to a domain type while keeping the generated message type for wire and JSON, as ScalaPB does for `Timestamp`. That looks like a genuine gap RustPB could fill.

### Gaps
- I did not find out how gin-tonic applies `rust_type` to repeated or optional fields: the type path is returned before cardinality is applied. Nor did I find how its JSON, if any, treats mapped types.
- I found no published conformance results for any crate's custom-type paths.
- I could not determine whether buffa will implement #490, or when; it has no milestone.

## Q3. Unknown fields, enums, oneofs, presence, recursion, bytes and strings

### Takeaway
buffa and femtopb preserve unknown fields by default. pilota and micropb make it opt-in. quick-protobuf drops them, and it also maps unknown enum values to the default variant, a silent data-loss bug class.

Open enums take three shapes:
- `EnumValue<T> { Known, Unknown(i32) }` in buffa and femtopb;
- a newtype `struct E(i32)` with associated consts in pilota and micropb;
- a closed Rust enum with silent fallback in quick-protobuf.

Every crate maps a oneof to `Option<Enum>`.

### Cited Findings
- **buffa unknown fields.** Preserved by default as `UnknownFields { Vec<UnknownField{number, data: Varint|Fixed64|Fixed32|LengthDelimited(Vec<u8>)|Group(UnknownFields)}> }`. The stated cost is mostly memory (a 24-byte `Vec` header per message). There are opt-outs, `preserve_unknown_fields(false)` and `preserve_unknown_fields_in(prefixes)`. — [DESIGN.md L425-450](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L425-L450)
- **buffa enums.** Generated as real Rust enums with variant names "verbatim from the .proto — no case transform". Open-enum fields are `EnumValue<T>`. Closed-enum fields use `T` directly, and "unknown values are routed to unknown fields during decoding". In JSON, `EnumValue::Unknown(n)` serializes as the integer. — [DESIGN.md L360-381, L514](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L360-L381)
- **buffa oneofs.** Rust enums under a reserved `__buffa::oneof::<msg>::Kind` module tree, re-exported at the natural path. `__buffa` is "the only name buffa reserves in user namespace". `unbox_oneof` controls boxing. — [DESIGN.md L265-331](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L265-L331), [buffa-build/src/lib.rs L1324](https://github.com/anthropics/buffa/blob/104716922c/buffa-build/src/lib.rs#L1324)
- **buffa spec gaps.** Where the spec is silent, buffa follows Java for closed enums inside a oneof, and rejects overflow bits in the 10th varint byte where C++ is said to discard them. (prost 0.14.6 also rejects them, `encoding/varint.rs`, following protobuf-go.) — [DEV post](https://dev.to/iainmcgin/zero-copy-protobuf-and-connectrpc-for-rust-1m3e)
- **buffa recursion.** Recursive types (`Struct` ↔ `Value`) compile without manual `Box` because of arithmetic-limit decoding; "prost cannot compile these without manual `Box` indirection". The default recursion limit is 100 and can be configured through `DecodeOptions`. — [DESIGN.md L591, L481-491](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L481-L491)
- **buffa UTF-8.** Strings are validated; editions `utf8_validation = NONE` fields become `Vec<u8>`/`&[u8]`. — [DEV post](https://dev.to/iainmcgin/zero-copy-protobuf-and-connectrpc-for-rust-1m3e), [DESIGN.md L25](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L25)
- **pilota enums and oneofs.** Enums are generated as `pub struct Enum(i32)` with `pub const A: Self = Self(0)` and `From<i32>`, so open by construction. A oneof becomes `Option<module::Enum>`, boxed (`Option<Box<a::B>>`) when recursive. Recursive message fields become `Option<Box<A>>` through a built-in `BoxedPlugin`. — [pilota-build/test_data/protobuf/oneof.rs L5-14, L296-297, L457-484](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/test_data/protobuf/oneof.rs#L457), [pilota-build/src/plugin/mod.rs L157](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/plugin/mod.rs#L157)
- **pilota unknown fields.** Only for messages listed in `Builder::keep_unknown_fields(paths)`. They are stored as `pub _unknown_fields: ::pilota::BytesVec`, a list of raw byte chunks re-emitted on encode, and as an `_UnknownFields(BytesVec)` variant for enums or unions. — [pilota-build/src/lib.rs L255](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/lib.rs#L255), [pilota-build/src/codegen/mod.rs L146-147, L439-440](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/codegen/mod.rs#L146), [codegen/pb/mod.rs L578-657](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/codegen/pb/mod.rs#L578)
- **micropb presence.** A "hazzer" bitfield tracks optional fields: "`Option<T>` actually takes up twice as much space as `T`". The pitfall: "A field will be considered empty ... if its bit in the hazzer is not set, *even if the field itself has been written*." — [micropb-gen/src/lib.rs L161-240](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/lib.rs#L161-L240)
- **micropb enums, oneofs, recursion.** Enums are `#[repr(transparent)] pub struct Language(pub i32)` with associated consts ("proto3 requires enums to store unrecognized values"). Oneofs are `Option<Msg_::Number>` enums. Cycles are "broken by automatically boxing". — [micropb-gen/src/lib.rs L266-300, L407-436](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/lib.rs#L407-L436)
- **micropb unknown fields.** Skipped by default. `unknown_handler` adds an `_unknown` field of a user type implementing `FieldEncode`/`FieldDecode`, applied only to the configured path (`[no_inherit]`). — [micropb-gen/src/config.rs L498-512](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/config.rs#L498-L512)
- **femtopb unknown fields.** "every message that has at least one field must include an `unknown_fields` field". It backs the decoder's catch-all arm; omitting it is a compile error. `UnknownFields<'a>` is not a copy: it holds `known_tags: &'static [u32]` plus a window into the source buffer, so unknown fields are preserved without allocating. — [src/lib.rs L59-70](https://github.com/dflemstr/femtopb/blob/eaa5326d72/src/lib.rs#L59-L70), [src/unknown_fields.rs L9-16](https://github.com/dflemstr/femtopb/blob/eaa5326d72/src/unknown_fields.rs#L9-L16)
- **femtopb enums and recursion.** `EnumValue<E> { Known(E), Unknown(i32) }`. Recursive or boxed submessages become `Deferred<'a, A>`, which holds the raw bytes, decodes on demand, and "writes those bytes back out verbatim". — [src/enumeration.rs L23-29](https://github.com/dflemstr/femtopb/blob/eaa5326d72/src/enumeration.rs#L23-L29), [src/deferred.rs L6-21](https://github.com/dflemstr/femtopb/blob/eaa5326d72/src/deferred.rs#L6-L21), [build/src/lib.rs L1593-1610](https://github.com/dflemstr/femtopb/blob/eaa5326d72/build/src/lib.rs#L1593-L1610)
- **quick-protobuf enum fallback.** Generated `impl From<i32>` for enums ends with `_ => Self::default()`, so unknown enum values are silently replaced by the default. — [pb-rs/src/types.rs L2203-2211](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/pb-rs/src/types.rs#L2203-L2211)
- **quick-protobuf unknown fields and cycles.** Unknown fields are skipped with `r.read_unknown(bytes, t)?` and not preserved. Reference cycles become `Option<Box<..>>`. — [pb-rs/src/types.rs L1741, L815](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/pb-rs/src/types.rs#L1741)
- **quick-protobuf open issues.** "proto3 optionals ignored" ([#219](https://github.com/tafia/quick-protobuf/issues/219)), "proto3: optional fields with default values not serialized" ([#260](https://github.com/tafia/quick-protobuf/issues/260)), "Default values not respected" ([#216](https://github.com/tafia/quick-protobuf/issues/216)).

### Inferences
- For RustPB's byte-compatibility goal, femtopb's two ideas are worth noting, especially for JS targets where allocation is costly:
  - a window into the source buffer for unknown fields;
  - `Deferred<T>`, raw bytes kept and re-emitted verbatim.
- **Two open-enum designs.** `EnumValue<T>` (buffa, femtopb) gives exhaustive `match` plus an explicit unknown case. A newtype `struct E(i32)` (pilota, micropb) is smaller and makes unknown values first-class, but loses exhaustiveness. ScalaPB uses a sealed trait with an `Unrecognized(value)` case, which `EnumValue<T>` mirrors most closely.
- **Oneof representation is a settled question.** All five crates use `Option<Enum>` per oneof.

### Gaps
- I did not test femtopb's claim that unknown fields round-trip in order when they are interleaved with known fields. The window spans from the first to the last unknown field, so known fields in between are presumably excluded through `known_tags`.

## Q4. Codegen architecture: IRs, plugin systems and configuration surfaces worth copying

### Takeaway
pilota has the only explicit multi-stage IR in the group, and it is incremental and shared between Thrift and protobuf: `parser -> IR -> Resolver -> RIR -> CodegenTy`, on Salsa. Its plugin system is additive, though: plugins add attributes and nested items and mark fields boxed, but cannot change field types.

buffa uses no custom IR. It treats protoc/buf-resolved descriptors as the IR, has one code path parameterized by resolved edition features, and offers a library codegen crate shared by a protoc plugin and build.rs.

micropb has a typed, path-inherited `Config` that can also be loaded from TOML. femtopb shows the cheapest possible approach: reuse prost-build and rewrite its AST with `syn`.

### Cited Findings
- **pilota layers.** Builder API, then Plugin System, then Codegen (Thrift, Protobuf and Workspace backends), then Middle (Context, Resolver, Type System), then IR (`IR -> Resolver -> RIR -> CodegenTy`), then Parser, then the Salsa database for incremental compilation. — [docs/pilota-build-architecture.md L33-127](https://github.com/cloudwego/pilota/blob/8239e1e20b/docs/pilota-build-architecture.md#L33-L127)
- **pilota types.** `TyKind::Path` is unresolved in the IR. RIR `Path { kind: DefKind, did: DefId }`. `CodegenTy` maps to Rust (`FastStr` to `::pilota::FastStr`, `Map` to `AHashMap`, `Adt` as Struct, Enum or NewType). Tags (`TagId`) attach annotations. — [docs/pilota-build-architecture.md L128-200](https://github.com/cloudwego/pilota/blob/8239e1e20b/docs/pilota-build-architecture.md#L128-L200)
- **pilota backends.** `CodegenBackend` has `codegen_struct_impl`, `codegen_service_impl`, `codegen_enum_impl` and `codegen_newtype_impl`. A newer `PbBackend` uses "pilota's own encoding library". — [docs/pilota-build-architecture.md L342-394](https://github.com/cloudwego/pilota/blob/8239e1e20b/docs/pilota-build-architecture.md#L342-L394)
- **pilota plugins.** The `Plugin` trait has the hooks `on_codegen_uint`, `on_item`, `on_field`, `on_variant` and `on_emit`. Built-ins are SerdePlugin, BoxedPlugin, ImplDefaultPlugin and AutoDerivePlugin. The `Adjust` API offers only `set_boxed`, `add_attrs` and `add_nested_item`. — [pilota-build/src/plugin/mod.rs L21-38](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/plugin/mod.rs#L21-L38), [pilota-build/src/middle/adjust.rs L6-43](https://github.com/cloudwego/pilota/blob/8239e1e20b/pilota-build/src/middle/adjust.rs#L6-L43)
- **pilota drops custom option values.** Open issue #372 (2026-09-02): when parsing protobuf custom options, pilota-build "only recorded which extension was used (tag ID) and discarded the option value". Backends cannot emit, for example, `const SERVICE_ID: u32 = 0x10016f` and must re-parse the `FileDescriptorSet`. The code is `UsedOptions::from_pb_unknown_fields` (`ir/ext/pb.rs:86-99`). — [issue #372](https://github.com/cloudwego/pilota/issues/372)
- **pilota keeps only its own annotations.** Issue #271 (custom annotations for plugins): the maintainer says the resolver "only keeps annotation info specified in pilota-build". — [issue #271](https://github.com/cloudwego/pilota/issues/271)
- **buffa pipeline.** `.proto` goes to protoc or buf (which "resolve, edition feature resolution baked into descriptors"), then `FileDescriptorSet`, then buffa-build or protoc-gen-buffa, then `buffa-codegen`. "Adding support for future editions ... is a matter of adding new default feature values". — [DESIGN.md L153-167, L452-479](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L153-L167)
- **buffa escape hatches.** `Config::descriptor_set(path)` and `descriptor_set_bytes` accept pre-built descriptors. buffa also offers a remote BSR plugin. — [DESIGN.md L139-145](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L139-L145), [README.md L378](https://github.com/anthropics/buffa/blob/104716922c/README.md)
- **buffa output layout.** One content file per kind, `<stem>.__oneof.rs`, `__view_oneof.rs`, `__ext.rs` and so on, plus a per-package `<pkg>.mod.rs` stitcher. — [DESIGN.md L302-308](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L302-L308)
- **buffa reflection modes.** Bridge mode re-encodes into a `DynamicMessage`; VTable mode (the default) implements `ReflectMessage` on generated types and views. Both are exercised by dedicated conformance runs (`via-reflect`, `via-vtable`). — [DESIGN.md L518-540](https://github.com/anthropics/buffa/blob/104716922c/DESIGN.md#L518-L540)
- **micropb config inheritance.** `Config` is built by a declarative macro (`config_decl!`); each field is `Option<T>` so unset values inherit, and `[no_inherit]` blocks propagation. With the `serde` feature it is serializable, which enables TOML config files and the CLI. — [micropb-gen/src/config.rs L105-170](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/micropb-gen/src/config.rs#L105)
- **femtopb pipeline.** `protox::compile`, then `prost_build::Config` (generated in memory), then `syn::parse_file` with `transform_item` / `transform_prost_attr` / `transform_field_type`. — [build/src/lib.rs L195-264, L379-663](https://github.com/dflemstr/femtopb/blob/eaa5326d72/build/src/lib.rs#L195-L264)

### Inferences
- **For a ScalaPB-like RustPB**, the strongest combination looks like this:
  - buffa's "descriptors are the IR; resolve features first" approach;
  - micropb's typed, path-inherited, serializable config;
  - one option namespace (buffa `buffa.ext.*` with a reserved registry number, gin-tonic `gin_tonic.v1.rust_type`, pilota `pilota.*`), so the same setting can come from a `.proto` option or from build config.
- **pilota's limits show what to avoid.** Plugins cannot change types, and option values are dropped (#372). If RustPB has plugins, they need full read access to option values and a sanctioned hook for type substitution; otherwise type mapping ends up a hard-coded list (pilota's `"string" | "vec" | "btree" | "ordered_f64"`).
- **femtopb's prost-rewrite approach** is cheap but ties output to prost's choices. It is not suitable when representation, such as oneofs and enum wrappers, must differ.

### Gaps
- I did not benchmark pilota-build's Salsa incrementality, or measure whether it helps one-shot build.rs use.

## Q5. Known bugs, design mistakes, abandoned features; maintenance status

### Takeaway
quick-protobuf is the clearest cautionary tale. Its owner went silent: the last push was 2024-02, maintainers asked to hand it over (#263), and since then open memory-safety reports have accumulated (OOB read, uninitialized memory, Miri UB). Its "owned" mode depended on `transmute`-based self-referencing and needed an unsoundness fix.

micropb removed two features (`field_lifetime`, `recursive_field`). buffa reversed its blanket-impl design for custom strings and spent a whole release hardening input bounds.

### Cited Findings
- **quick-protobuf soundness and security issues, all open:**
  - [#273](https://github.com/tafia/quick-protobuf/issues/273) "VULNERABILITY: 0.8.1 Heap OOB read via unvalidated submessage length" (2026-08-25): `read_len` sets `end = start + len` without clamping to the buffer.
  - [#274](https://github.com/tafia/quick-protobuf/issues/274) unsound `Reader::from_reader` `set_len` on uninitialized bytes (2026-09-20).
  - [#272](https://github.com/tafia/quick-protobuf/issues/272) Miri-reported UB in `BytesReader`.
  - [#270](https://github.com/tafia/quick-protobuf/issues/270) uninitialized memory exposure in `write_packed_fixed_*`.
  - [#269](https://github.com/tafia/quick-protobuf/issues/269) unaligned access in `read_packed_fixed`.
- **quick-protobuf maintenance.** [#263](https://github.com/tafia/quick-protobuf/issues/263) "Status and future maintenance of the project" (2024-08) says rust-libp2p "depends highly" on it while requests went unattended. One commenter asked the owner to hand the project over. Another called it "the best library I found to generate code that will be used with wasm".
- **pb-rs owned mode.** `--owned` generates a `Pin<Box<..>>` holding the buffer and the parsed message, using `core::mem::transmute` to `'static`. pb-rs 0.10.0's changelog says "fix unsoundness in Owned variant (breaking change)". — [pb-rs/src/types.rs L1789-1842](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/pb-rs/src/types.rs#L1789-L1842), [Changelog.md](https://github.com/tafia/quick-protobuf/blob/54e7d6c5d9/Changelog.md)
- **micropb removals and fixes.** 0.5.0 removed the `field_lifetime` and `recursive_field` configs (BREAKING). 0.7.0 fixed the unbounded-recursion DoS ([#44](https://github.com/YuhanLiin/micropb/issues/44)) and the missing wire-type validation on known fields ([#45](https://github.com/YuhanLiin/micropb/issues/45)). — [CHANGELOG.md](https://github.com/YuhanLiin/micropb/blob/0b86edac8e/CHANGELOG.md)
- **buffa hardening.** 0.9.2 addressed CVE-2026-55407 follow-ups: the extension-merge path "gave every wire record its own full allowance", plus a stack overflow in reflective `Any` serialization and "the map-entry parser differential between view and owned decoding". It also notes that "`buffa-types` gains `google::protobuf::Option`, which shadows the prelude `Option`" under glob imports. — [CHANGELOG.md L9-30](https://github.com/anthropics/buffa/blob/104716922c/CHANGELOG.md#L9-L30)
- **buffa doc inconsistency.** The README says in one place "binary, JSON, and text — with zero expected failures" (L15) and elsewhere "binary and JSON conformance test suite (v33.5 ...)" (L374). — [README.md](https://github.com/anthropics/buffa/blob/104716922c/README.md#L15)
- **pilota bugs.**
  - [#371](https://github.com/cloudwego/pilota/issues/371): "read_struct_end does not restore last_read_field_id, corrupting field ids decoded after nested structs" (Thrift compact, 2026-08-26).
  - [#61](https://github.com/cloudwego/pilota/issues/61): TBinaryProtocol returns the wrong size with zero-copy.
  - [#3](https://github.com/cloudwego/pilota/issues/3): parser migration, open since 2022.

### Inferences
- **Bounds checking.** quick-protobuf's OOB read and buffa's 0.9.2 hardening share a root cause: length-prefixed sub-reads that trust the declared length or share one budget. RustPB should bound every sub-read against the parent buffer and keep a decode-wide element and memory budget from day one.
- **The Rust-to-JS constraint.** Designs built on `unsafe`, `transmute` self-reference, `offset_of!` tables or `set_len` on uninitialized memory will not translate to JavaScript and are where soundness bugs cluster. Counted roughly (lines matching `\bunsafe\b`, comments excluded):

  | Runtime | Lines mentioning `unsafe` | Where |
  |---|---|---|
  | buffa | 241 | mostly `table/*` and `view.rs` |
  | pilota | 67 | |
  | micropb | 33 | |
  | quick-protobuf | 5 | |
  | femtopb | 0 | |
  | gin-tonic-core | 0 | |

  buffa's default `Unrolled` codec plus owned types needs none of the table code, which is opt-in. femtopb shows a full codec is possible with zero `unsafe`.

### Gaps
- I did not verify whether quick-protobuf's #273 has a RustSec advisory or CVE.
- I did not audit buffa's 148 open issues for correctness bugs beyond custom types.

## Q6. Conformance suite results

### Takeaway
Of the crates in scope, only buffa publishes conformance results (zero expected failures in the main binary, JSON and text lists). pbrs claims clean results. micropb, femtopb, pilota, quick-protobuf and gin-tonic show no conformance testing in their repos.

### Cited Findings
- **buffa.** "passes the protobuf binary and JSON conformance test suite (v33.5, editions up to 2024). Both `std` and `no_std` builds pass the full suite including JSON." `conformance/known_failures.txt` and `known_failures_text.txt` contain only header comments, so zero entries. Mode-specific lists hold a few entries each:

  | Mode | Known-failure lines |
  |---|---|
  | view | 8 |
  | view_json | 10 |
  | view_vtable | 14 |
  | table | 9 |
  | lazy | 8 |
  | reflect | 3 |
  | nostd | 0 |

  — [README.md L372-374](https://github.com/anthropics/buffa/blob/104716922c/README.md#L372-L374), [conformance/](https://github.com/anthropics/buffa/tree/104716922c/conformance)
- **pbrs.** Claims 5,631 binary and JSON cases and 909 text cases with no unexpected failures (runner v35.1); only a subset of Edition 2024. — [docs.rs/crate/pbrs](https://docs.rs/crate/pbrs/latest)
- **The rest.** A case-insensitive grep for "conformance" over the micropb, femtopb and pilota repos (`*.rs`, `*.md`, `*.toml`) found nothing. — local clones at the commits above

### Inferences
- The known-failure counts above are line counts, which include header comments, so the true number of failing cases is lower. buffa's alternative modes (views, table codec, reflection) carry a few residual failures, which suggests every alternative codec path roughly doubles conformance effort. RustPB's JS-compiled path should be treated as another such mode and run through the conformance runner separately.

### Gaps
- I did not count the exact entries in buffa's mode-specific failure lists after removing comment lines.
- I found no third-party conformance comparison table covering these crates.
