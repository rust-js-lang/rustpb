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

### 4.2 Messages, fields, enums

| ScalaPB | rustpb |
|---|---|
| `case class M(..)` | `pub struct M { pub .. }`, `Default`, `Clone`, `PartialEq`, `Debug` |
| message field `Option[T]` | `Option<T>`; `Option<Box<T>>` only where the type is recursive (rustc requires it) |
| `no_box` | `T` (missing on the wire → `T::default()`) |
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
- **`sealed_oneof`** (`oneof sealed_value`) → the message *is* the enum:

  ```rust
  pub enum ApiParams {
      GetItemParams(GetItemParams),
      UpdateItemParams(UpdateItemParams),
      // ..
  }
  ```

  Where it appears as a field, it is `Option<ApiParams>`. `None` plays the
  role of ScalaPB's `Empty` case. This is the most natural Rust of the
  whole mapping.

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

**Migration converts options mechanically:** a script rewrites
`(scalapb.*)` to `(rustpb.*)` in every `.proto` file. It translates Scala
type syntax (`Option[UserId]` → `Option<UserId>`, `java.time.Instant` → its
mapped Rust type) and lists every option it dropped, file by file, for a
human to check.

## 5. JSON

The browser speaks JSON today. The Scala app's endpoints send proto models
through a codec built on scalapb-jsoniter. Its `Printer` is created with
only `includingDefaultValueFields = false`; everything else is the default.
So the format is **canonical proto3 JSON**:

| Rule | Value |
|---|---|
| Field names | lowerCamelCase (`json_name`); the parser takes proto names too |
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
4. **Mutations.** Break the derive (swap a tag, drop zigzag, skip an unknown
   field) and require a test to fail.
5. **rust-js.** The same corpus is run through the JS output, compared
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
