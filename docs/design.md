# rustpb: ScalaPB for Rust — design

Status: draft, 2026-11-13. Name: **rustpb** (crates `rustpb`, `rustpb-derive`,
`rustpb-build`, `rustpb-json`); written RustPB in prose titles.

## 1. Goal, in one picture

`.proto` stays the source of truth. rustpb turns it into idiomatic Rust,
the way ScalaPB turns it into Scala. The same Rust compiles natively for
the server and, through rust-js, to JavaScript for the browser.

```
 Layer 1  SCHEMA       column.proto                      ← hand-written, source of truth
                         [(rustpb.field).type = "OrderId"]
                               │
                               │ rustpb-build  (build.rs; pure-Rust .proto parser)
                               ▼
 Layer 2  RUST TYPES   pub struct ColumnRecord { pub order_id: OrderId, .. }
                         #[derive(rustpb::Message)]              ← generated
                               │
              ┌────────────────┴────────────────┐
              ▼ rustc                           ▼ rust-js
 Layer 3  SERVER (native)                  BROWSER (JS)
          bytes: database, workflows,      JSON over HTTP (bytes too, if ever needed)
          message queues
```

**The contract:** for every message, rustpb reads and writes the same bytes
as ScalaPB, and the same JSON as the existing Scala app's ScalaPB JSON printer.
Old data stays readable, and a Scala service and a Rust service can share
a topic or a table during the migration.

## 2. What we must cover (measured in a production ScalaPB codebase, 2026-11-13)

858 `.proto` files, all `proto3`, 2 `service` declarations. ScalaPB options
in use, by count:

| Count | ScalaPB option | What it does |
|---:|---|---|
| 7785 | `(field).type` | TypeMapper: `string` on the wire, `OrderId` in code. 534 of these are `Option[X]` |
| 857 | file `(scalapb.options)` | `package_name`, `single_file`, `import`, `preamble`, `flat_package`, `lenses`, `getters` |
| 504 | `(message).extends` | case class mixes in a trait |
| 280 | `(field).no_box` | message field is `T`, not `Option[T]` |
| 263 | `preserve_unknown_fields` | 243 `false`, 20 `true`; **the default is `true`**, so ~600 files keep unknown fields |
| 463 | `annotations`, `companion_annotations` | Scala annotations on class, companion, field |
| 178 | `(field).collection_type` | 169 `Set`, 7 `List`, 1 `Vector`, 1 `Seq` |
| 131 | `key_type`, `value_type` | TypeMapper on map keys and values |
| 26+ | `sealed_oneof_*` | `oneof sealed_value` becomes a sealed trait |
| 12 | `(field).required` | `no_box`, and parsing fails when the field is absent |
| 12+6 | `derives`, `sealed_oneof_derives` | Scala 3 `derives` clause |
| ~10 | enum options, `map_type`, `(message).type` | rare |

TypeMappers are ~223 hand-written Scala files, mostly ids.

## 3. Pieces

```
 rustpb/                      (this repo, plain native Rust; no rust-js dependency)
 ├── rustpb            runtime: Message, TypeMapper, UnknownFields, DecodeError,
 │                     rustpb::wire (pinned prost arithmetic, §3 dec. 1)
 ├── rustpb-derive     #[derive(rustpb::Message)]: encode/decode/size/merge, Default
 ├── rustpb-build      .proto → Rust source, called from build.rs
 ├── rustpb-json       proto3 JSON (serde impls), matching the Scala app's printer
 ├── rustpb-migrate    coexistence tool: add / check / drop-scalapb (§4.5)
 └── proto/rustpb.proto   the custom options, like scalapb/scalapb.proto

 rust-js/                     (separate repo, its own ADR)
 └── compiler-owned lowering of #[derive(rustpb::Message)] → readable encode/decode
```

### Decisions and why

1. **Wire arithmetic from prost, behind one `rustpb::wire` module.**
   Varints, zigzag and fixed-width codecs are where subtle bugs hide, and
   prost's are fuzzed (CIFuzz on every PR) and used everywhere. Its Kani
   proofs cover only prost-types' `Timestamp` and `Duration`, not the varint
   code. Even so, only the *arithmetic* is trusted.
   `prost::encoding` is `#[doc(hidden)]` and not a stable API (prost #903);
   prost is passively maintained; its string merge uses `unsafe`; and its
   *policies* differ from ScalaPB's (§4.6). So:
   - every call to prost goes through `rustpb::wire`, which pins prost
     exactly;
   - rustpb takes no policy from prost: packedness, wire-type mismatch,
     oneof placement, map checks and unknown fields are all rustpb's own;
   - strings are decoded by rustpb's safe code.

   Replacing the arithmetic later is then a change to one module.

2. **Our own derive, not prost's.** prost-derive's field kinds are a closed
   list (`scalar`, `string`, `bytes`, `message`, `enumeration`, `map`,
   `oneof`, `group`): no custom types, no sealed oneofs, no unknown fields.
   7,785 typed fields cannot go through it. `rustpb-derive` generates its
   own encode, decode and size code, calling `rustpb::wire` for the
   arithmetic, with ScalaPB's policies (§4.6).

3. **build.rs, with a pure-Rust parser (`protox`).** It is the Rust
   convention (prost-build, tonic-build). It needs no `protoc` binary on any
   machine and runs the same under Cargo for native builds and for rust-js.
   Generated code is not committed. ScalaPB's protoc-plugin form is not
   offered as a second way.

4. **Fields are written in field-number order, as ScalaPB does**
   (`ProtobufGenerator.scala:676`, `message.fields.sortBy(_.getNumber)`).
   That includes **oneof members, each written at its own field number**
   among the other fields. prost-derive instead places a whole oneof at its
   lowest tag (its own `lib.rs` TODO), which reorders bytes, so rustpb does
   not copy it. For messages without sets, maps or unknown fields, the
   output is byte-identical to ScalaPB's, which makes the proof in §6 simple.

5. **The runtime has no `unsafe`, no FFI, no link-time registries, no
   atomics and no self-referential types.** These are the patterns that
   don't lower to JavaScript, and where the ecosystem's soundness bugs
   cluster (quick-protobuf #269–#274). femtopb shows a full codec needs none
   of them. Size computation keeps no cached size inside the struct (§4.6).

6. **rustpb reads descriptors with its own generated code.** prost-types
   drops extension values (pilota #372 shows the same failure), which would
   silently lose every `(rustpb.*)` option. buffa's answer, as connect-rust
   uses it, is simpler than a reflection library: decode the descriptor set
   with the library's own generated `descriptor.proto` types. They keep
   unknown fields, so custom options survive. rustpb does the same:
   - protox parses the `.proto` files and encodes a `FileDescriptorSet` as
     bytes;
   - rustpb-build decodes those bytes with rustpb-generated types for
     `descriptor.proto` and `rustpb.proto`, and reads `(rustpb.*)` options
     from the preserved extension fields;
   - those bootstrap types are generated by rustpb itself and checked in;
     CI regenerates them and fails on any difference.

   So rustpb's own decoder, unknown fields included, runs in every build:
   rustpb tests itself on every build. A CI check also compares protox's
   and protoc's descriptors (`json_name`, options, packed) for every schema
   file.

7. **Code generation never panics.** A name collision becomes a diagnostic
   naming both protos. Examples: a oneof vs a nested type; an enum variant
   named `Error`; the generated `Empty` or `Unrecognized` variant vs a
   proto value of that name; keywords like `gen` and `self`. Generated code
   writes `::core::option::Option`, because a proto message named `Option`
   would shadow the prelude. prost panics on several of these.

## 4. Mapping every ScalaPB feature

### 4.1 TypeMapper → `rustpb::TypeMapper`

```rust
pub trait TypeMapper<Base> {
    type Custom;
    fn to_custom(base: Base) -> Result<Self::Custom, MapError>;
    fn to_base(custom: &Self::Custom) -> Base;
}

/// Only for mappers of singular fields in defaultable messages (§4.1, defaults).
pub trait DefaultMapper<Base>: TypeMapper<Base> {
    /// Must equal `to_custom(base default)`. Generated tests check it.
    fn default_custom() -> Self::Custom;
}
```

- **Fallible on purpose.** ScalaPB's `toCustom` either succeeds or throws.
  A throw inside `parseFrom` becomes an error value in the app's binary
  codec (`Left`). In Rust that is `Err(MapError)`, which becomes the
  message's `DecodeError`: same outcome, no panic. Mappers that never fail
  return `Ok`.
- **Behaviour is the mapper's, kept exactly.** For example, a typical generic
  id mapper turns an unparsable string into the id's *default value*, not
  an error. Its Rust twin must do the same: `Ok(default)`.
- **The mapper is a type; by default, the custom type itself.**

  ```protobuf
  string order_id   = 3 [(rustpb.field).type = "OrderId"];
  string created_by = 4 [(rustpb.field).type = "Option<UserId>",
                         (rustpb.field).mapper = "OptionalUserId"];
  ```

  `OrderId` is the app's own type, so it implements `TypeMapper<String>`
  itself (`type Custom = Self`). `Option<UserId>` cannot: Rust's orphan
  rule forbids implementing a foreign trait for `Option<_>`. So the field
  names a small local mapper type. The same applies to foreign types such as
  `Instant` (`TimestampMessage` → a time crate's type). It is one mechanism:
  a mapper is always a type implementing the trait. `mapper` only says
  which type, when the custom type cannot be its own mapper.
- **Less boilerplate than ScalaPB.** One derive (or a macro) per id family
  writes the impl, so ~223 hand-written TypeMapper files become derives.
  A known ScalaPB pain point, "no compile-time check that `.type` names a
  real type", stays as it is: the generated code fails to compile,
  naming the field.
- `key_type`, `value_type`: the same trait, applied to map keys and values.
- In JSON, a mapped field prints its *base* value, as ScalaPB's does.

#### Mapped fields by kind: decoding, encoding, defaults (review #1, finding 3)

ScalaPB's rules depend on the field's *kind*, as its generator defines
them (`DescriptorImplicits`): **singular** is `!supportsPresence &&
!isRepeated && !isInOneof`, so it covers plain proto3 scalars, `no_box`
and `required` message fields. **Presence** is proto3 `optional`, or a
message field boxed in `Option`. Then come **oneof** members, and
**repeated** fields and maps. rustpb copies each rule per kind:

| Kind | Decoding (`ParseFromGenerator`) | Encoding (`ProtobufGenerator.generateWriteTo`) |
|---|---|---|
| Singular | Reads the *base* value: the last occurrence wins for scalars, and occurrences merge for messages. Calls `to_custom` **once**, after the whole message is read (`usesBaseTypeInBuilder = isSingular`) | Written only if `to_base(v)` is non-empty: a scalar not equal to the base default, a message with a non-zero serialized size |
| Singular, `required` | As singular; absent → `DecodeError` naming the field | **Always written**, even when the base is the default |
| Presence (`Option<C>`) | Mapped **on each occurrence**: a scalar maps its new value; a message maps `to_custom(merge(to_base(current), next))`. A failure at any occurrence fails the decode, even if a later occurrence would have mapped | **Every `Some` is written**, `Some` of a base default included (`Some(0)`, `Some("")`) |
| Oneof member | Mapped **on each occurrence**, as presence. A rejected value fails the decode even when a later occurrence selects another variant | The selected variant is **always written**, even when its base is empty |
| Repeated, map key/value | Each element is mapped as it is read, from packed and unpacked chunks alike | **Every element is written**, empty ones included; packed elements via `to_base` |

**Absent singular field:** `to_custom(base default)` (a scalar default, or `Base::try_default()?` for a message base), never the custom
type's own `Default`. A failure gives `DecodeError::Map { field, source }`:
no panic, no substitute value. Absent presence, oneof and repeated fields
are `None`, unselected or empty: **the mapper is never called**, as in
ScalaPB, so these fields put no condition on the mapper.

**`impl Default` asks only what ScalaPB's default instance asks.**
ScalaPB's `defaultInstance` calls `toCustom(base default)` for singular
mapped fields only. So:

- A message with no singular mapped field, and no singular child that is
  non-defaultable (see "Default construction composes" below), gets
  `impl Default` with no condition on any mapper. A mapper used only in
  optional, oneof or repeated positions (for example, a non-empty-string id
  that rejects `""`) needs nothing beyond `TypeMapper`, and its behaviour is
  unchanged.
- For each singular mapped field, the generated `impl Default` calls
  `<M as DefaultMapper<Base>>::default_custom()`. So `DefaultMapper` is
  required of exactly the mappers ScalaPB would call for the default
  instance, and it never panics. For each such mapper, `rustpb-build`
  generates a `#[test]` asserting
  `M::to_custom(base default) == Ok(M::default_custom())`. Decoding still
  calls `to_custom`, not `default_custom`, so decoding stays faithful
  whatever the test says.
- **A singular field whose mapper rejects its base default.** ScalaPB
  accepts this, and throws the first time the default instance is touched.
  rustpb does not ask the mapper to change. The message declares
  `option (rustpb.message).default = false`. If that option is missing,
  the generated `impl Default` fails to compile, naming the mapper that
  lacks `DefaultMapper`.

#### Default construction composes through the message graph

ScalaPB's `defaultInstance` is built from the defaults of the message's
singular fields. A `no_box` or `required` child's default is the child's
`defaultInstance`, and a mapped field's is `toCustom(base default)`, where a
message base's default is *its* `defaultInstance`. So one failing default
fails every message that reaches it through singular fields (review #1,
follow-up). rustpb copies that graph exactly:

1. **Every generated message implements
   `rustpb::Message::try_default() -> Result<Self, DefaultError>`**, the
   twin of `defaultInstance`. It builds each singular field:
   - a scalar: its proto default;
   - an unmapped `no_box`/`required` message: `Child::try_default()?`;
   - a mapped field: `M::to_custom(base_default?)`, where `base_default` is
     the scalar default or `Base::try_default()?`;
   - optional, oneof and repeated fields: `None`, unset and empty, with no
     call.

   `DefaultError` names the path that failed, such as
   `Parent.child.owner_id`, and keeps the mapper's `MapError`.
2. **Defaultability is computed by `rustpb-build`** over the whole
   descriptor set, imports included, as a fixed point. A message is
   **non-defaultable** if it declares `default = false`, or if any singular
   field's default needs a non-defaultable message: an unmapped
   `no_box`/`required` child, or a mapped field whose base message is
   non-defaultable. Only the root cause declares the option. Parents
   inherit the property, and their generated docs say why, naming the chain.
   **Cycles:** a cycle made only of singular default edges (an unmapped
   `no_box`/`required` child, or a mapped field's base message), such as
   `A.b: no_box B` with `B.a: no_box A`, has no finite default even with no
   `default = false` in it. `rustpb-build` detects it and refuses, with an
   "unsupported default-construction cycle" error that names the cycle,
   instead of generating a recursive `Default` that overflows the stack. An
   optional, oneof or repeated edge breaks the cycle, because its default
   is `None` or empty and needs no call.
3. **`impl Default` exists exactly for defaultable messages.** It is
   infallible: scalars, `Child::default()`, and `default_custom()`. A
   generated test asserts `M::try_default() == Ok(M::default())` for each.
   A non-defaultable message has no `impl Default`; its default
   construction is only `try_default()`, which fails as ScalaPB's would.
   `DefaultMapper` is required only for singular mapped fields of
   *defaultable* messages. A mapped field whose base is non-defaultable
   makes its message non-defaultable, so `default_custom` is never asked
   of it.
4. **Decoding never uses `Default` or `try_default` to start.** The
   generated decoder holds each singular field in a builder slot, the base
   value as `Option<Base>` (absent is `None`), and each repeated field as a
   growing `Vec`. Nested timing is ScalaPB's (`ParseFromGenerator`): each
   occurrence of a message field is **decoded completely when it is read**,
   with its own end-of-message checks. A later occurrence merges into the
   already-decoded value. Undecoded bytes are never collected across
   occurrences to check later. At the end of the message, in this order:
   (a) every `required` field must be present, or `DecodeError` names it,
   because ScalaPB checks its required bitmask before building the message;
   (b) only then does an absent ordinary singular slot become its default
   by the rule in 1; (c) singular mapped slots call `to_custom` (§4.1).
   So:
   - a populated `Parent` whose `child` is present decodes, even though
     `Child` is non-defaultable;
   - an absent `child` or `owner_id` fails with `DecodeError::Default`
     naming the path, as ScalaPB's parser throws;
   - an `Option<Child>` that is absent is `None`, with no call. A present
     but empty `Child`, in an optional, oneof or repeated position, fails,
     because decoding that `Child` hits its own absent singular field. That
     too is what ScalaPB does.
5. **The prost boundary.** prost's `Message::decode` and
   `message::merge_repeated` require `M: Default`, and so does its map-entry
   merge. `message::merge` and `message::encode` require only
   `M: Message`, but they are built around prost's `Message` trait. rustpb
   therefore owns its own `Message` trait, message merge, map-entry decode
   and the builder above. It reuses only prost's scalar wire primitives:
   `encode_varint`/`decode_varint`, `encode_key`/`decode_key`, `WireType`,
   length delimiters, `skip_field`, and the scalar and string/bytes
   codecs that merge into an existing value.

Fixtures for the graph, each checked against ScalaPB:

- `Child`: a singular mapped `string` that rejects `""`, with
  `default = false`. `Parent`: an unmapped `no_box Child`, with no option of
  its own. `Parent` must have no `impl Default`, and `Parent::try_default()`
  must fail, naming `Parent.child.<field>`. A populated `Parent` must decode
  and round-trip. A `Parent` with `child` absent must fail.
- A `required` child, and a mapped field whose *base message* is
  non-defaultable. Both propagate the same way.
- `Option<Child>`, `Vec<Child>` and a oneof `Child`: absent or empty
  containers decode; a present empty `Child` fails; populated elements
  decode.
- A singular-only cycle (`A.b: no_box B`, `B.a: no_box A`): `rustpb-build`
  refuses it, naming the cycle. The same cycle through an optional edge
  generates and decodes.
- A `required` field absent alongside an absent non-defaultable singular
  child: the required error is reported, since it is checked first.
- A singular message field twice, where the first occurrence alone fails
  its own decode: the decode fails at the first occurrence, as ScalaPB's
  does.
- Three levels deep (`Grandparent → Parent → Child`): the error path names
  all three.

Fixtures, each checked against ScalaPB:

- A singular mapper whose mapped base default differs from the custom
  type's `Default`. An absent field must decode to the former.
- A mapper used only in repeated, optional and oneof positions that rejects
  `""`. The message gets `impl Default`; empty containers decode; a `""`
  element fails the decode.
- A singular mapper that rejects its base default, with `default = false`.
  `try_default()` fails; decoding with the field absent fails; decoding
  with a valid value succeeds.
- Oneof: a rejected mapped value followed by another variant. The decode
  fails.
- Optional message: an occurrence whose intermediate merge is rejected,
  followed by one that would make it valid. The decode fails.
- Encoding: `Some(0)` and `Some("")` on mapped presence fields, a selected
  mapped oneof variant whose base is `""`, empty repeated elements, and a
  `required` field holding the base default. All must be written, matching
  ScalaPB's bytes.

### 4.2 Messages, fields, enums

| ScalaPB | rustpb |
|---|---|
| `case class M(..)` | `pub struct M { pub .. }`, `Default`, `Clone`, `PartialEq`, `Debug` |
| message field `Option[T]` | `Option<T>`; `Option<Box<T>>` only where the type is recursive (rustc requires it) |
| `no_box` | `T`. Missing on the wire → `T::try_default()?`, or for a mapped field `to_custom(base default)` (§4.1, defaults compose) |
| `required` | `T`; decode fails with `DecodeError` naming the field when absent |
| `repeated` → `Seq` | `Vec<T>` |
| `collection_type = "Set"` | `BTreeSet<T>` (deterministic bytes; open question 3) |
| `map<K,V>` → `Map` | `HashMap<K, V>`; `(rustpb.field).map_type` for `BTreeMap` |
| proto3 `optional` | `Option<T>` |
| enum | `pub enum E { A, B, Unrecognized(i32) }`: an unknown number survives a round trip, as ScalaPB's `Unrecognized` does |
| `int64`, `uint64`, `fixed64` | `i64`, `u64` (BigInt under rust-js) |
| `bytes` | `Vec<u8>` |
| `preserve_unknown_fields` (default `true`) | `pub unknown_fields: rustpb::UnknownFields`, re-emitted after known fields, as ScalaPB does |

### 4.3 Oneofs

- Plain `oneof x` → `pub x: Option<m::X>` with `enum X { A(..), B(..) }`.
- **Sealed oneofs** come in ScalaPB's two styles, chosen by the oneof's
  name (`DescriptorImplicits.sealedOneofStyle`). Both are kept, because
  emptiness must survive in *every* position: top level, singular field,
  repeated element, map value (review #1, finding 1).

  **`oneof sealed_value`** (default style, 108 files): the message *is* an
  enum, with `Empty` as a real variant, as ScalaPB's `case object Empty`:

  ```rust
  #[derive(Default)]
  pub enum ApiParams {
      #[default]
      Empty,
      GetItemParams(GetItemParams),
      UpdateItemParams(UpdateItemParams),
      // ..
  }
  ```

  Every position uses the enum itself, never `Option<ApiParams>`, as
  ScalaPB uses `ApiParams`: a field is `ApiParams`, a repeated field is
  `Vec<ApiParams>`, a map value is `ApiParams`. An empty message on the
  wire, at any of those positions, decodes to `ApiParams::Empty`, and
  `Empty` encodes as an empty message. `Option` would only cover singular
  fields, and would give `Vec<Option<ApiParams>>` a second meaning of empty.

  **`oneof sealed_value_optional`** (16 files): no `Empty` variant. Every
  position is `Option<S>`, as ScalaPB types it `Option[S]` everywhere
  (`sealedOneofScalaType`): the field is `Option<S>`, repeated is
  `Vec<Option<S>>`, a map value is `Option<S>`, and decoding a top-level
  message gives `Option<S>`.

  Fixtures for both styles, in binary and JSON: empty top level, an empty
  singular field, an empty element in the middle of a repeated field, and an
  empty map value. Each must decode to `Empty` or `None` and re-encode to the
  bytes ScalaPB writes.

### 4.4 Options with no Rust counterpart: dropped, with reasons

| ScalaPB option | Why Rust does not need it |
|---|---|
| `(message).extends` (504), `companion_extends` | Rust impls are open: the app writes `impl ApiParamsTrait for X { .. }` in its own source. Scala needs the mixin at class definition; Rust does not |
| `sealed_oneof_extends` | The same: the app implements the trait for the generated enum |
| `lenses` | `&mut msg.field` *is* the lens |
| `getters`, `no_default_values_in_constructor` | `Default` + struct update syntax `M { a, ..Default::default() }` |
| `preamble` (Scala source) | Rust items live in ordinary source files next to the module |
| `companion_annotations` | No companion objects in Rust; associated items go in `impl` blocks |

### 4.5 Options translated to Rust form

| ScalaPB | rustpb |
|---|---|
| `package_name`, `flat_package`, `single_file` | `(rustpb.file).module`: the Rust module path. One file per proto package |
| `import: "com.example.id.OrderId"` | `(rustpb.file).use: "crate::id::OrderId"`, emitted as `use` |
| `annotations`, field `annotations` | `(rustpb.message).attributes`, `(rustpb.field).attributes`: Rust attributes |
| `derives` | `(rustpb.message).derive: "Hash"`, added to the derive list |

#### Migration: both option sets live side by side (review #1, finding 2)

Scala and Rust services share the same `.proto` files while both exist,
and Scala generation still needs every `(scalapb.*)` option. So the
migration tool **adds** `(rustpb.*)` options next to the ScalaPB ones and
removes nothing:

```protobuf
import "scalapb/scalapb.proto";
import "rustpb/rustpb.proto";

string created_by = 4 [(scalapb.field).type = "Option[UserId]",
                       (rustpb.field).type  = "Option<UserId>",
                       (rustpb.field).mapper = "OptionalUserId"];
```

Each generator ignores the other's options, as protoc plugins do with
unknown extensions. One schema stays the single source of truth;
generated Rust-only copies were rejected because they would drift
from what Scala reads.

- **`rustpb-migrate add`** derives the `rustpb` options from the ScalaPB
  ones. It translates Scala type syntax (`Option[UserId]` → `Option<UserId>`,
  `java.time.Instant` → its mapped Rust type, through a mapping table) and
  lists every ScalaPB option with no Rust form (§4.4), file by file, for a
  human to check.
- **`rustpb-migrate check`** runs in CI during coexistence. It fails when the
  two sets disagree, for example a field mapped for Scala but not for
  Rust, or a `required` or collection type on one side only. A schema edit
  can't silently update only one language.
- **ScalaPB options are removed only after the last Scala consumer is
  gone**, by `rustpb-migrate drop-scalapb`. rustpb itself never reads
  `(scalapb.*)` options: only the migration tool knows ScalaPB.

### 4.6 Wire policies, and limits

The Rust ecosystem's worst bugs were not in new features. They were small
policies copied by default, and missing limits. See
[the prior-art report](prior-art/report.md) (notes in `prior-art/notes/`). Each rule below is rustpb's
own, read from ScalaPB's source where ScalaPB decides it, and pinned by a
fixture.

**Binary policies:**

| Policy | Rule (verified in ScalaPB) | Prior-art failure it avoids |
|---|---|---|
| Packedness | Computed from the raw `packed` option, with the proto3 default `true`, never from "has any options". Nearly every rustpb field carries `(rustpb.field)` options | prost wrote such fields unpacked until 0.14.5 (#976) |
| A known field with the wrong wire type | **Treated as unknown**: ScalaPB matches the full tag `(number << 3) \| wire_type`, so a mismatch falls to `case tag =>` (`ParseFromGenerator.scala:215–222`). Packable repeated fields accept both packed and unpacked tags. This holds for map fields too | prost returns `UnexpectedWireType`; its maps skipped the check until 0.14.6 (#1473) |
| Oneof placement | Each member at its own field number (§3 dec. 4) | prost-derive writes a oneof at its lowest tag |
| `-0.0` and defaults | A singular `float`/`double` is omitted when `v != 0.0` is false under **IEEE** comparison (`ProtobufGenerator.isNonEmpty`), so **`-0.0` is omitted**, as ScalaPB does. NaN is written | prost #1177; buffa compares bits. rustpb follows ScalaPB, not either of them |
| Unknown fields, kept | Stored per field number; within a number, values grouped by wire type and written **varint, fixed32, fixed64, length-delimited** (`UnknownFieldSet.Field.writeTo`), after the known fields. Across numbers, ScalaPB's order is a Scala hash map's, so rustpb writes ascending field numbers, and §6 compares unknown fields by value | rust-protobuf loses order too, undocumented |
| Unknown **group** (wire type 3/4) | Message **preserves** unknown fields: decode fails, as ScalaPB's `UnknownFieldSet.Field.Builder.parseField` throws `InvalidProtocolBufferException`. Message **doesn't preserve**: skipped (with the depth budget below), as `skipField` does | RUSTSEC-2024-0437: skipping groups bypassed the recursion limit |
| Size and write | One owner computes both; a generated assertion in debug builds and the first fuzz oracle check `encoded_len == bytes written`. A negative `int32` is 10 bytes, a packed `Unrecognized(-1)` too | rust-protobuf #766: size and write disagree for a negative unknown enum |
| Size strategy | No cached size in the struct (§3 dec. 5). Encoding computes nested sizes once per message, into an external size buffer, giving linear time (buffa's approach) | prost recomputes nested sizes per level (#1112); rust-protobuf's in-struct atomic size can't lower to JS |
| Equality | Generated `PartialEq` includes `unknown_fields`, compared by value per field number, as ScalaPB's case-class equality includes `unknownFields` | Google's crate omits `PartialEq` entirely |

**Limits** (every advisory against a Rust protobuf crate is a denial of
service):

- **One depth budget**, default 100 as in C++, shared by nested-message
  decode, map-entry decode, unknown-group skipping, JSON input nesting and
  default construction. Exceeding it is a `DecodeError`, never a stack
  overflow (RUSTSEC-2020-0002, RUSTSEC-2024-0437). Encoding checks the same
  depth, so rustpb never writes a message it cannot read back (prost #736).
- **Never reserve from a declared length.** A length prefix or packed byte
  count is checked against the bytes remaining in its parent before any
  read. Collections grow as elements actually arrive (RUSTSEC-2019-0003,
  quick-protobuf #273).
- **JSON input** is bounded the same way: depth, string and number sizes;
  no `arbitrary_precision` (buffa #482).
- **Limits are passed explicitly, in two profiles.** Every decode entry
  point takes a `DecodeOptions` value (depth, total message size, element
  count and memory), so a server sets them from its own configuration;
  there is no global or thread-local state, which would not carry over to
  JS. `DecodeOptions::untrusted()` is the default for wire input. rustpb-build
  decodes the compiler's own descriptor output with
  `DecodeOptions::tooling()`, which has looser size limits but the same
  depth limit, as connect-rust does with buffa's `tooling_decode_options()`.

Fixtures: depth bombs through messages, maps and unknown groups; a huge
declared length on a tiny input; a packed field whose byte count runs past
its parent; and one fixture per advisory, also seeded into the fuzz corpus.

## 5. JSON

The browser speaks JSON today. The Scala app's endpoints send proto models
through a codec built on scalapb-jsoniter. Its `Printer` is created with
only `includingDefaultValueFields = false`; everything else is the default.
So the format is **canonical proto3 JSON**:

| Rule | Value |
|---|---|
| Field names | lowerCamelCase (`json_name`) |
| Default values | omitted (`0`, `""`, `false`, empty lists and maps, unset messages) |
| `int64`, `uint64`, `fixed64` | strings, `"123"` |
| Enum *fields* | names, `"ACTIVE"` |
| Maps | JSON objects, keys as strings |
| Mapped (`type`) fields | the base value, e.g. the id's string |
| Sealed oneof | the wrapper message's JSON: `{"getItemParams": {..}}` |

Two app-specific rules sit on top, and rustpb must match them too:

- **A standalone enum** (an enum that is the whole value, not a field)
  is written as `{"value": 3}`. When read, it accepts that object, the
  enum's name as a string, or a bare number. An unknown name is an error;
  an unknown number becomes `Unrecognized(n)`.
- **One extra writer** prints the same messages with defaults included.
  It is used explicitly, never as the default. rustpb-json offers it as a
  function, `to_json_including_defaults`, not as a second codec.
- `google.protobuf.Struct` / `Value` fields map to a raw-JSON type, which
  is written as the JSON itself.

How rustpb-json does it:

- `rustpb-json` generates `impl Serialize` / `impl Deserialize` per message,
  written out (as pbjson does), not `#[derive(Serialize)]`, because proto3
  JSON is not serde's default shape. rust-js already compiles hand-written
  serde impls (rust-js ADR 0459).
- Golden tests pin every rule above against the Scala printer's output
  (§6). The pinned scalapb-jsoniter version matters: if the Scala side
  upgrades, the goldens are re-made and compared.
- **rustpb owns its number and time formatting.** It writes floats,
  doubles, `Timestamp` and `Duration` with its own writers, matched
  digit-for-digit to scalapb-jsoniter. It never uses serde_json's float
  path, which writes NaN as `null` (pbjson #106) and changed digits when it
  moved from ryu to zmij. It never uses a time crate's RFC 3339 output, which
  gives `+00:00` where ProtoJSON wants `Z` (pbjson's own test asserts the
  wrong form). Negative `Duration`s with zero seconds keep their sign: two
  Rust crates lose it.
- **A mapped field's JSON goes only through `to_base` and the base type's
  writer, in every position.** A custom type never needs `Serialize`, so
  foreign types such as time types work unchanged.
- **Enum names come from a generated table** holding every value and every
  alias, separate from the Rust variant names (open question 4). pbjson
  can't parse aliases (#123).
- **JSON is one switch, per crate.** `rustpb_build::Config::json(false)`
  leaves out the generated serde code, and turning off the runtime's
  `json` cargo feature drops the matching bounds. A crate that only speaks
  binary then pays nothing for JSON in compile time or code size, which
  matters most for the JS output. This follows connect-rust's
  `generate_json`. It is the only way to turn JSON off: there is no per-file
  or per-message variant.
- **Options are fixed at build time, never ambient.** buffa passes its runtime
  parse options through a thread-local, which wouldn't carry over to the JS
  output.

#### The reference parser's contract (review #1, finding 4)

The reference is **scalapb-jsoniter 0.1.2**: the version the app's build
and its shared codec library both pin. Its parser was read at tag
`v0.1.2` (`JsonFormat.scala`, `Parser.parseObject`), not inferred from the
printer settings. It does **not** follow canonical ProtoJSON on reading:

| Input | scalapb-jsoniter 0.1.2 | Canonical ProtoJSON | rustpb |
|---|---|---|---|
| Proto field name `my_field` | **Skipped silently**: only the `json_name` is indexed | Accepted as an alias | **Skipped, as 0.1.2 does** |
| Unknown key | Skipped | Error by default | Skipped |
| `"field": null` | Treated as absent (default) | Treated as absent | Treated as absent |
| Same key twice | The last **non-null** one wins, a repeated field's array included. A later `null` is skipped, so the earlier value stays | Error | The same, `null` rule included |
| Two members of one oneof | Both reach the reader; the result is fixture-defined | Error | Whatever the fixture records 0.1.2 doing |

Matching 0.1.2 exactly is the decision: a JSON value must decode the same
in a Scala and a Rust service during coexistence, even where 0.1.2 is
lenient. Silently dropping `my_field` is a data-loss trap, but changing it
is a behaviour change for the Scala side too. It is listed as open
question 7, not fixed quietly in rustpb.

Parser fixtures, each checked against 0.1.2 with its expected value or
error: the proto-name alias, an unknown key, `null` for each kind of field,
a duplicate scalar key, a duplicate repeated key, a duplicate key whose
last value is `null` (the earlier value stays), two oneof members, a
standalone enum as `{"value": n}` / `"NAME"` / `n` / an unknown name, and
int64 given as a number versus a string.

## 6. Proof: how we know the bytes and JSON are the same

How rustpb is tested is what decides whether anyone can trust it. The
evidence and the reasoning behind every rule here are in
[the test-harness report](test-harness/report.md), with notes in
`test-harness/notes/`.

```
                   one protocol: 4-byte little-endian length + ConformanceRequest/Response
                   (conformance.proto, the framing Google's runner already uses)

   ScalaPB worker (JVM)          rustpb native              rustpb via rust-js (Bun, Node)
   ScalaPB 0.11.x +              testee binary              same testee, compiled to JS
   scalapb-jsoniter 0.1.2
          │                            │                              │
          │ recorded once,             │ every case, every PR         │ every case, every PR
          │ re-asked nightly           ▼                              ▼
          └──────────► fixtures/*.outcome ◄──── canonical outcome records, compared
```

**ScalaPB is the authority, but it is slow and external, so its answers
are recorded as data and replayed everywhere.** Every pull request
compares rustpb, native and JS, against recorded ScalaPB outcomes,
without starting a JVM. A nightly job asks the live ScalaPB worker again
and fails if any recorded outcome drifts.

### 6.1 Oracles, and where each runs

| Oracle | Judges | Runs |
|---|---|---|
| **ScalaPB worker** | Everything: bytes, JSON, values, errors | Recorded when a fixture is added or changed; all outcomes re-asked nightly |
| **Native vs JS** | That the rust-js build equals the native build | Every case, every PR |
| **Google's conformance runner** | Spec conformance, and deviations ScalaPB and rustpb *share*, which a differential test cannot see | Per PR, native and JS, binary and JSON |
| **Single-sided properties** | Never panics; `encoded_len` equals the bytes written; re-encoding is stable; the merge rule; unknown fields preserved | Inside the fuzz loop and the property tests |
| **Reference models** | Defaultability over the message graph (§4.1) | Fast loop: exhaustive small graphs; nightly: random graphs |
| **Kani** | `rustpb::wire` arithmetic, in Rust | Per PR |
| **Numbers grid** | 64-bit arithmetic in the *emitted* JavaScript | Per PR at M5, nightly in full |

### 6.2 The testee, and canonical outcomes

- **One testee crate**, built natively and by rust-js, answers
  `ConformanceRequest`s for any message type of the schema under test.
- **The judge is never the defendant.** The testee and the driver parse
  the conformance envelope by hand, as buffa does: rustpb's own decoder
  never decides whether rustpb's bytes are right.
- **Every answer is a canonical outcome record:** `ok` with the re-encoded
  bytes, the JSON text (numbers kept as raw text, never re-parsed) and a
  value dump that prints 64-bit numbers as decimal strings (the JS testee
  prints BigInts itself, since `JSON.stringify` throws on them); or `err`
  with an error **class**, never a message text. Resource-limit errors are
  a class of their own.
- Sets, maps and unknown-field order are canonicalized before comparison
  (§4.6). Everything else is compared exactly.

### 6.3 Fixtures

- **Names state the expectation**, as JSONTestSuite does: `y_` must decode,
  `n_` must fail, `i_` either is acceptable, whatever ScalaPB records. Each
  name carries the design rule it pins (e.g. `y_4.6-wrong-wire-type.bin`),
  so a rule with no fixture can be listed by a script.
- **Each fixture has a committed `.outcome`** and a `PROVENANCE` line naming
  the ScalaPB, jsoniter and protobuf versions that recorded it. A record
  job runs only when fixtures change.
- **Every bug gets a fixture**, every fuzz finding included, before it is
  fixed.

Hand-built decoding fixtures (review #1, finding 5). Random values only
produce bytes a well-behaved writer emits. A decoder must also handle
bytes no ScalaPB writer produces. The right column is what we expect;
ScalaPB's recorded outcome is the authority, and a disagreement is fixed in
rustpb:

| Case | What it proves |
|---|---|
| A singular scalar field twice | The last one wins |
| A singular message field twice | The occurrences merge, as base values before mapping (§4.1) |
| A mapped message field twice | It merges as base, then `to_custom` runs once |
| A repeated scalar field, packed and unpacked chunks mixed | Both forms accepted, order kept |
| Oneof: the same variant twice | The message variant merges; a scalar variant's last wins |
| Oneof: two different variants | The last one wins |
| Sealed oneof empty in each position | `Empty` / `None` (§4.3) |
| A `required` field absent | `DecodeError` naming the field |
| A map entry with key or value missing | The default is used for the missing part |
| Unknown fields of each wire type, preserved and not | Kept, re-written grouped per number as varint, fixed32, fixed64, length-delimited (§4.6), or dropped |
| A known field number with the wrong wire type | Treated as an unknown field, as ScalaPB does (§4.6) |
| A packable field with options, encoded | Packed, the proto3 default (§4.6) |
| `-0.0` in a singular `double` | Omitted on write, as ScalaPB's IEEE check does (§4.6) |
| A oneof member between two plain fields | Written at its own field number (§3 dec. 4) |
| An enum number not in the enum | `Unrecognized(n)` and back |
| A mapping failure | `DecodeError::Map` |
| Truncated: inside a varint, a length prefix, a fixed64 | `DecodeError`, not a panic |
| Invalid: a length past the end, wire type 6 or 7, tag 0, a varint over 10 bytes | `DecodeError`, not a panic |
| A 10-byte varint whose last byte sets bits past 64 | Whatever ScalaPB records |
| An unknown group | `DecodeError` where unknown fields are preserved; skipped where they are not (§4.6) |
| Depth at limit − 1, at the limit, at limit + 1: messages, maps, groups, JSON | Decodes, decodes, `DecodeError`; never a stack overflow (§4.6) |
| Invalid UTF-8 in a `string` | `DecodeError`, as ScalaPB's parser rejects it |

### 6.4 Golden corpus and real data

1. **Golden corpus from ScalaPB.** The ScalaPB worker generates random
   values for every message type (ScalaCheck generators) and records them
   as bytes and JSON fixtures.
2. **Byte equality** where the encoding is deterministic; **canonical value
   equality** where it is not: sets, maps, unknown-field order.
3. **Real data and the full schema set.** A private worker runs in the Scala
   app's own environment over all 858 protos and a sample of stored
   records and queue messages, decoded by both implementations. That
   proves the mappers too, the default-on-bad-id case included. Only counts
   are published, never the data.

### 6.5 Google's conformance runner

- **Pinned runner version**, proto3 binary and JSON, native and JS.
- **The failure lists are measured, not written.** The same runner is run
  against the ScalaPB worker, and the list it produces is committed.
  rustpb's native list, rustpb's JS list and ScalaPB's list must be
  **identical**, and that equality is a test of its own. Known entries
  include `UnknownOrdering` (ScalaPB groups unknown fields by wire type,
  §4.6) and `OriginalProtoFieldName` (scalapb-jsoniter 0.1.2 skips proto
  names, §5, open question 7).
- **Skips can't hide failures.** The runner never fails on a skipped test,
  so the harness asserts the *answered* set: the testee counts requests,
  and the count must equal the pinned runner's proto3 binary and JSON
  total. A runner upgrade is then a reviewed change to a committed count.
- Message types rustpb doesn't support yet, such as `Any` and the
  wrappers that `TestAllTypesProto3` imports, are answered as listed
  failures, never as skips, so they stay visible and fail once fixed.

### 6.6 Fuzzing

- **Coverage-guided fuzzing runs natively only**, with the single-sided
  oracles (§6.1) inside the loop. A JVM call on every execution would make
  it thousands of times slower.
- **The corpus is replayed through ScalaPB and JS in batches.** New corpus
  entries are sent to the ScalaPB worker nightly, their outcomes committed,
  and the whole corpus replays as a normal test against native and JS. The
  per-PR cross-language check stays deterministic; fuzzing time only buys
  new inputs. When a disagreement is shrunk, the *same* disagreement must
  survive, not just any failure.
- **CI fuzzing uses ClusterFuzzLite** (`code-change` 300–600 s per PR,
  nightly `batch`, `prune` and `coverage`). prost's per-PR CIFuzz relies
  on OSS-Fuzz membership, which a new project won't have.
- **Seeds:** one input per Rust protobuf security advisory, plus the
  golden corpus.
- **Per-format JSON targets** for int64 strings, floats, `Timestamp`,
  `Duration` and base64. Byte mutation of whole JSON documents almost never
  reaches those parsers.
- **Anomaly sweeps**, nightly, native and JS: every prefix and every
  single-bit flip of every golden message. The expectation comes from
  invariants: an error or a value, never a panic; a value re-encodes
  stably; and peak allocation, under a counting allocator, stays within a
  small multiple of the input length. That last check is the codec's
  version of SQLite's out-of-memory loop, since Rust aborts on allocation
  failure.

### 6.7 Properties and models

- `rustpb-build` generates `Arbitrary` impls (proptest) for every message.
  A mapped field is generated from its base value and mapped, so custom
  types need no generator of their own.
- Properties: `decode(encode(x)) == x` by canonical value; `encoded_len`
  equals the bytes written; re-encoding a decoded value is stable;
  `decode(a ++ b) == merge(decode(a), decode(b))`, on base values before
  mapping; unknown fields survive a round trip.
- Seeds are fixed and printed in CI (`PROPTEST_RNG_SEED`). Failures are
  saved as values, not just as seeds, because seeds stop reproducing when a
  generator changes.
- **Per-mapper round trip:** `to_base(to_custom(b)?) == b`. A mapper that is
  lossy on purpose, such as default-on-bad-id, declares it with
  `(rustpb.field).lossy = true`, and its test then checks only the
  non-lossy subset. One Rust crate's UUID mapper silently rewrites
  hyphenated ids on re-encode, which this catches.
- **Defaultability model:** a deliberately naive reference implementation of
  §4.1's fixed point, compared with rustpb-build's on every graph of up to
  four messages, exhaustively, and on random larger graphs. Plus
  properties: adding `default = false` never makes a message defaultable;
  adding an optional edge never creates a cycle.

### 6.8 The generator

Code generation is the risk no fixed corpus reaches: nothing written by
hand covers the combinations of 7,785 mapped fields.

- **Diagnostics are data:** unit tests on in-memory descriptors match the
  error type and its spans.
- **Goldens** of generated code (insta), reviewed as diffs.
- **A compile matrix** over every fixture schema, run in CI.
- **Generate twice, compare bytes:** generation must be deterministic.
- **API guarantees are compile-time checks:** for example, a
  non-defaultable message has no `Default`
  (`static_assertions::assert_not_impl_any!`), which avoids trybuild's
  error-text drift between rustc versions.
- **Random valid schemas** (ScalaPB's own `proptest` model: random schemas
  and values, compiled, compared with another implementation). Nightly,
  through all three testees.
- **Random adversarial schemas** hunting collisions, keywords, cycles and
  mapper edge cases. Expectation: never panics, deterministic output, and
  either compiles or gives a diagnostic.

### 6.9 Mutation testing

- **cargo-mutants** over the runtime, derive and build crates: `--in-diff`
  per PR, the full set sharded nightly. Survivors are triaged; equivalent
  mutants are skipped by name.
- **cargo-mutants never visits generated code**, so a hand-written list of
  codegen mutations (swap a tag, drop zigzag, skip an unknown field, write
  a oneof at the wrong position) is applied to the generator's templates
  and must turn the fixture suite red.
- **The harness is mutated too.** A testee patched to answer `skipped`,
  and a canonicalizer patched to sort every field, must each turn CI red.

### 6.10 Proofs

- **Kani proves `rustpb::wire`, in Rust:** varint encode/decode round trip
  and size, zigzag, the length-prefix check against the bytes remaining,
  and a Rust reference of the two-32-bit-halves algorithm used for 64-bit
  fields in JS, proved equal to the `u64` path. prost has no such proofs:
  its Kani harnesses cover only prost-types' `Timestamp` and `Duration`.
- **Kani cannot see the JavaScript rust-js emits.** The emitted 64-bit
  logic is held to native by the **numbers grid**: every varint length from
  1 to 10, crossed with boundary values near 2^31, 2^32, 2^53, 2^63 and
  2^64−1. The same grid technique found real wrong answers in rust-js on
  its first run.
- **Miri** runs nightly on a codec subset. The runtime has no `unsafe`
  (§3 dec. 5), so Miri's value is in dependencies.

### 6.11 The JS build, from the first milestone

- **The JS testee runs in shadow from M1**: every fixture and the corpus,
  through rust-js, on Bun and Node, non-blocking. Wrong answers appear
  where new shapes first compile, so drift must surface per feature, not
  in one batch at M5. It becomes blocking at M5, with its own conformance
  list.
- **A batch where nothing ran fails.** If every case is refused or skipped,
  the run fails, not passes.
- **End to end in a real browser.** The same generated types, built
  natively for a small Rust server and by rust-js for the browser, talk
  over HTTP. The server binds port 0 and prints it, and echoes the raw
  body in hex beside the decoded value. Playwright drives Chromium per PR
  and Chromium, Firefox and WebKit nightly, with `retries: 0`, so a flake is
  investigated, not hidden. The test asserts byte equality of what crossed
  the wire, for binary and JSON: a mapped id, a sealed oneof, an
  `Unrecognized` enum and unknown fields carried through a round trip.
  connect-rust's `wasm-client` example is the model for its shape.
- A **bundle-size budget** gates the JS output; speed is a tracked trend,
  never a pass or fail.

### 6.12 Tiers

Each tier contains the one before it. Every step has a hard timeout of
about twice its usual time.

| Tier | Budget | Adds | JVM |
|---|---|---|---|
| Fast loop (`cargo test`) | ≤ 60 s | Unit tests, every fixture against its outcome, corpus replay, properties at 256 cases, goldens, diagnostics, exhaustive small-graph model | No |
| Per PR | ≤ 15 min, sharded (nextest `hash:`) | JS testee on Bun and Node; conformance native and JS with answered counts and list equality; compile matrix; determinism; adversarial schemas; Kani; ClusterFuzzLite `code-change`; `cargo mutants --in-diff`; e2e on Chromium; semver, MSRV, 32-bit, minimal versions; bundle size; protox vs protoc; record job when fixtures change | Only the record job |
| Nightly | ≤ 6 h, parallel | Re-asking ScalaPB for every outcome; corpus replay through ScalaPB; fuzz `batch`, `prune`, `coverage`; anomaly sweeps; random valid schemas through all three; high property counts; full mutation runs; Miri; coverage; three browsers; full numbers grid; schema-evolution matrix; the private real-data worker | Yes |
| Release | ≤ 1 day | Fuzz and property soak; re-verified pins; regenerated `TESTING.md`; release provenance | Yes |

### 6.13 Evidence published to users

- **`TESTING.md`, generated by CI**: what is tested, how, and the current
  numbers (fixtures, corpus size, conformance counts, fuzz hours, mutation
  score, worker throughput), never written by hand.
- **A results matrix**: rustpb native, rustpb via rust-js, ScalaPB, and for
  context prost and buffa, on the conformance runner, with every
  deviation from the spec listed and explained. rustpb's number can't be
  zero, because faithfulness to ScalaPB requires measured deviations; the
  matrix shows the same list for ScalaPB.
- **Written compatibility promises, each held by a CI check:** wire and
  JSON equality with a pinned ScalaPB and jsoniter version (the recorded
  outcomes); Rust API semver (cargo-semver-checks on a generated-API
  fixture crate); and the supported skew between generator and runtime
  versions.
- **Hygiene:** a security policy, MSRV policy, pinned tools, signed
  releases with provenance, and supply-chain checks (cargo-deny,
  cargo-audit).

## 7. The rust-js side (separate ADR in rust-js)

Two ways to get `encode`/`decode` into JavaScript:

| | Compile the derive's expansion | Compiler-owned lowering (serde's way, rust-js ADR 0077) |
|---|---|---|
| How | rust-js compiles the `rustpb::wire` calls as ordinary Rust | rustc checks the derive. rust-js skips its expansion, reads the `#[rustpb(..)]` attributes, and emits readable code on a small runtime |
| JS readability | Poor: generic `Buf` code | Good: `writer.string(3, value.orderId)` |
| Speed | Every varint is u64 BigInt math | Number for ≤32-bit fields, BigInt only for 64-bit fields |
| Risk | `bytes` crate's unsafe code may be refused | Two implementations of one rule (native derive, rust-js lowering), held equal by running both on §6's corpus |

Recommendation: start with compile-the-expansion for correctness, and move
to compiler-owned lowering, as serde did. JSON needs neither: it is
serde, already supported.

For 64-bit fields, the JS path decodes a varint as two 32-bit halves and
builds the BigInt once at the field boundary, as protobufjs's `long`
handling does, never a BigInt operation per byte. Kani proves a Rust
reference of that algorithm equal to the `u64` path, and the numbers grid
holds the emitted JavaScript to native (§6.10). No other pure-Rust protobuf runtime targets
JavaScript, so rustpb's JS build runs the same conformance and fuzz loops
as the native build, not just the corpus.

## 8. Milestones

The harness arrives **before** the feature it judges, or a milestone's
"done when" can't be checked. M0 exists for that reason.

| # | Proves | Harness that lands with it | Done when |
|---|---|---|---|
| 0 | The harness itself | Testee crate with hand-parsed envelope; driver and canonical outcomes (§6.2); public ScalaPB worker (§6); fixture layout, `PROVENANCE` and record job (§6.3); harness self-mutations (§6.9) | The worker and both testees answer the same request; a deliberately broken testee and canonicalizer turn CI red |
| 1 | Scalars, strings, typed ids, `no_box`, enums, unknown fields | Single-sided properties and generated `Arbitrary` (§6.7); Kani over `rustpb::wire` (§6.10); native fuzz targets with advisory seeds, corpus replay, ClusterFuzzLite (§6.6); generator diagnostics, goldens, compile matrix, determinism, adversarial schemas (§6.8); **JS testee in shadow** (§6.11) | One record type with typed ids matches ScalaPB's recorded outcomes, native and in shadow JS |
| 2 | Oneofs, sealed oneofs, maps with `key_type`/`value_type`, sets, `required` | Conformance runner, binary, native, with answered-count assertion (§6.5); defaultability model (§6.7); corpus replay through ScalaPB nightly (§6.6) | A large sealed oneof (~1,800-line API params file) matches; rustpb's binary list equals ScalaPB's measured list |
| 3 | JSON | JSON conformance with the list measured from ScalaPB and three-way equality (§6.5); per-format JSON fuzz targets (§6.6) | Golden JSON equal to the Scala app's printer for 1 and 2, digit for digit |
| 4 | Scale | Private real-data worker over all 858 protos (§6.4); random valid schemas through all three testees (§6.8); schema-evolution matrix; full mutation runs (§6.9); anomaly sweeps and depth boundaries (§6.6); coverage, Miri | All converted protos generate and compile; the full corpus and real-data sample match |
| 5 | rust-js | JS testee **blocking**, with its own conformance list on Bun and Node; full numbers grid (§6.10); end-to-end browser test on three engines; bundle-size gate (§6.11) | Native and JS outcomes identical on every fixture and the corpus; the browser test passes against a native server |
| 6 | Descriptors | Embedded `FileDescriptorSet` compared with protoc's | Embedded `FileDescriptorSet` per module, for a future record-store layer (FoundationDB's Record Layer is Java-only) |
| — | First public release | Generated `TESTING.md`, results matrix, compatibility promises, semver checks, MSRV, hygiene (§6.13) | Every claim on the testing page links to the CI run that produced it |

## 9. Open questions

1. ~~Name?~~ Resolved: `rustpb`. Check that the crate names are free on
   crates.io before the first publish.
2. ~~Which printer settings does the Scala app's JSON codec use?~~ Resolved
   in §5: canonical proto3 JSON, plus the `{"value": n}` standalone enum.
3. `Set` → `BTreeSet` (deterministic, needs `Ord` on ids) or `HashSet`
   (closer to Scala, nondeterministic bytes)?
4. Generated-enum naming: keep proto's `SCREAMING_CASE` values with prefix
   stripping, as prost does?
5. Do the 2 `service` declarations need code (gRPC or Connect), or only
   messages? Either way, **services are never part of rustpb.** As with buffa
   and connect-rust, a service generator is a separate layer that calls
   rustpb-build's library API and refers to rustpb's message types by path
   (`extern_paths`). tonic-build is codec-agnostic (`codec_path`), so a
   rustpb codec plugs in; tonic-prost-build would tie services to prost.
6. Time types: which Rust crate replaces `java.time.Instant`/`LocalDate`
   (`jiff` or `chrono`), and do the mappers live in rustpb or in the app?
7. scalapb-jsoniter 0.1.2 silently skips proto-name keys (`my_field`).
   Fix it on both sides together (an alias in the Scala parser and in
   rustpb, by one decision), or keep matching it? rustpb matches 0.1.2
   until this is decided. Prior art argues for fixing it: the ProtoJSON
   spec requires parsers to accept both names, and every Rust JSON reader
   surveyed does.
