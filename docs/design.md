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
 │                     wire primitives (re-used from prost::encoding)
 ├── rustpb-derive     #[derive(rustpb::Message)]: encode/decode/size/merge, Default
 ├── rustpb-build      .proto → Rust source, called from build.rs
 ├── rustpb-json       proto3 JSON (serde impls), matching the Scala app's printer
 ├── rustpb-migrate    coexistence tool: add / check / drop-scalapb (§4.5)
 └── proto/rustpb.proto   the custom options, like scalapb/scalapb.proto

 rust-js/                     (separate repo, its own ADR)
 └── compiler-owned lowering of #[derive(rustpb::Message)] → readable encode/decode
```

### Decisions and why

1. **Wire primitives from prost, not re-written.** Varints, zigzag, length
   prefixes and skipping unknown fields are exactly where subtle bugs hide.
   `prost::encoding` is proven. rustpb adds what prost lacks; it never
   re-implements the wire format.

2. **Our own derive, not prost's.** prost-derive's field kinds are a closed
   list (`scalar`, `string`, `bytes`, `message`, `enumeration`, `map`,
   `oneof`, `group`): no custom types, no sealed oneofs, no unknown fields.
   7,785 typed fields cannot go through it. `rustpb-derive` writes the same
   kind of code, calling `prost::encoding`, and adds those three features.

3. **build.rs, with a pure-Rust parser (`protox`).** It is the Rust
   convention (prost-build, tonic-build). It needs no `protoc` binary on any
   machine and runs the same under Cargo for native builds and for rust-js.
   Generated code is not committed. ScalaPB's protoc-plugin form is not
   offered as a second way.

4. **Fields are written in field-number order, as both ScalaPB and prost
   do** (`ProtobufGenerator.scala:676`, `prost-derive/src/lib.rs:91`).
   So for messages without sets, maps or unknown fields, the output is
   byte-identical to ScalaPB's, which makes the proof in §6 simple.

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

```
 Scala app (ScalaPB)                     rustpb (Rust)
 ───────────────────                     ─────────────
 generate N random values per message ─► golden/*.bin, *.json
                                                │
                     ┌──────────────────────────┤
                     ▼                          ▼
       Rust decode → re-encode       Rust decode → Scala decode
       == golden bytes?              (round trip back through ScalaPB)
```

1. **Golden corpus from ScalaPB.** A small Scala program in the Scala app
   writes random values for every message type (ScalaCheck generators), as
   bytes and JSON. Committed here as test data.
2. **Byte equality** where the encoding is deterministic. **Value equality**
   where it is not: sets, maps, and unknown-field order, compared after
   decoding on both sides.
3. **Real data.** A sample of stored records and queue messages is decoded
   by both implementations and the results compared. That proves the
   mappers too, the default-on-bad-id case included.
4. **Decoding fixtures** (review #1, finding 5). Random values only
   produce bytes a well-behaved writer emits. A decoder must also handle
   bytes no ScalaPB writer produces, so these are hand-built byte strings,
   each decoded by ScalaPB to record its expected value or failure. The
   right column is what we expect; ScalaPB's recorded result is the
   authority, and a disagreement is fixed in rustpb:

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
   | Unknown fields of each wire type, preserved and not | Kept in order, or dropped |
   | An enum number not in the enum | `Unrecognized(n)` and back |
   | A mapping failure | `DecodeError::Map` |
   | Truncated: inside a varint, a length prefix, a fixed64 | `DecodeError`, not a panic |
   | Invalid: a length past the end, wire type 6 or 7, tag 0, a varint over 10 bytes, a group | `DecodeError`, not a panic |
   | Invalid UTF-8 in a `string` | `DecodeError`, as ScalaPB's parser rejects it |

5. **Mutations.** Break the derive (swap a tag, drop zigzag, skip an unknown
   field) and require a test to fail.
6. **rust-js.** The same corpus is run through the JS output, compared
   against native rustpb (§7).

## 7. The rust-js side (separate ADR in rust-js)

Two ways to get `encode`/`decode` into JavaScript:

| | Compile the derive's expansion | Compiler-owned lowering (serde's way, rust-js ADR 0077) |
|---|---|---|
| How | rust-js compiles the `prost::encoding` calls as ordinary Rust | rustc checks the derive. rust-js skips its expansion, reads the `#[rustpb(..)]` attributes, and emits readable code on a small runtime |
| JS readability | Poor: generic `Buf` code | Good: `writer.string(3, value.orderId)` |
| Speed | Every varint is u64 BigInt math | Number for ≤32-bit fields, BigInt only for 64-bit fields |
| Risk | `bytes` crate's unsafe code may be refused | Two implementations of one rule (native derive, rust-js lowering), held equal by running both on §6's corpus |

Recommendation: start with compile-the-expansion for correctness, and move
to compiler-owned lowering, as serde did. JSON needs neither: it is
serde, already supported.

## 8. Milestones

| # | Proves | Done when |
|---|---|---|
| 1 | Scalars, strings, typed ids, `no_box`, enums, unknown fields | One record type with typed ids round-trips ScalaPB's golden bytes |
| 2 | Oneofs, sealed oneofs, maps with `key_type`/`value_type`, sets, `required` | A large sealed oneof (~1,800-line API params file) round-trips |
| 3 | JSON | Golden JSON equal to the Scala app's printer for 1 and 2 |
| 4 | Scale | All converted protos generate and compile; full corpus passes |
| 5 | rust-js | Corpus passes in the browser build, compared to native |
| 6 | Descriptors | Embedded `FileDescriptorSet` per module, for a future record-store layer (FoundationDB's Record Layer is Java-only) |

## 9. Open questions

1. ~~Name?~~ Resolved: `rustpb`. Check that the crate names are free on
   crates.io before the first publish.
2. ~~Which printer settings does the Scala app's JSON codec use?~~ Resolved
   in §5: canonical proto3 JSON, plus the `{"value": n}` standalone enum.
3. `Set` → `BTreeSet` (deterministic, needs `Ord` on ids) or `HashSet`
   (closer to Scala, nondeterministic bytes)?
4. Generated-enum naming: keep proto's `SCREAMING_CASE` values with prefix
   stripping, as prost does?
5. Do the 2 `service` declarations need code (gRPC via tonic), or only messages?
6. Time types: which Rust crate replaces `java.time.Instant`/`LocalDate`
   (`jiff` or `chrono`), and do the mappers live in rustpb or in the app?
7. scalapb-jsoniter 0.1.2 silently skips proto-name keys (`my_field`).
   Fix it on both sides together (an alias in the Scala parser and in
   rustpb, by one decision), or keep matching it? rustpb matches 0.1.2
   until this is decided.
