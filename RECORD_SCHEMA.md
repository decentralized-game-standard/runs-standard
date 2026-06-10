# RUNS Record Schema

> **Layer**: Data (Records and Fields)
> **File extension**: `.runs`
> **Status**: Design stage — no reference implementation yet. Precise enough to
> implement from; the first implementation is expected to correct it.

## Purpose

This document defines the data layer of RUNS: the declaration syntax for **Records**,
**Fields**, **enums**, and the references that tie them together. Everything a game
knows about itself at rest — entity state, configuration, constants, boundary data —
is declared here.

A **Record** is a named collection of typed **Fields** — a data store, a noun. It
holds Fields and has no input/output signature of its own. A Record stores its value
until a Processor produces a new one: **write once, never overwrite**. A "change" is
a fresh value, never in-place mutation. The [Network Topology](./NETWORK_TOPOLOGY.md)
specification defines how that law plays out in wiring; this document defines the
shapes themselves.

Records, enums, and game-defined types are declared in `.runs` files. Processor bodies
(computation) are a different layer with a different extension (`.runs-prim`, defined
in [DIGS](./DIGS_EXPRESSION_LANGUAGE.md)). Networks (wiring) share the `.runs`
extension with schemas because both declare structure, not computation.

Every decision in this specification is driven by one question: *can a solo developer
in a century implement this from scratch, with nothing but this document?*

---

## Lexical Structure

Schema files follow the same lexical rules as DIGS:

- UTF-8 encoding; ASCII-only keywords, operators, and identifiers.
- Logical lines end with a newline (U+000A); a file ends with a newline.
- Block structure by indentation: exactly **two spaces** per level, no tabs.
- Comments begin with `#` and run to end of line.
- Identifiers: a letter or underscore, then letters, digits, underscores.
  Case-sensitive.
- Qualified names: an optional namespace prefix, a colon, and an identifier —
  `spacewar:object`, `runs:transform`.

## Namespaces and Identity

The prefix on a qualified name (`spacewar:`, `doom:`, `pong:`) is an **umbrella
prefix** — a human-readable convenience, never identity. Real identity is
cryptographic: the author's public key roots authorship, and the content hash of the
published artifact roots the artifact itself. References between published artifacts
resolve by ID, never by prefix string, so two authors publishing `pong:object` is
harmless — different authors, different IDs, two distinct schemas.

One prefix is special: **`runs:`** is reserved. Only the standard defines `runs:`
keys, and a `runs:` key always carries exactly the standard's keys, types, and
semantics. Using `runs:` shapes is optional — it buys interoperability, not
compliance. The `runs:` palette lives in the
[RUNS Library](https://github.com/enduring-game-standard/runs-library).

## Publication

Every schema artifact — each Record, enum, and type declaration — is an
**independently published, content-addressed event**, referenced by ID. This holds
regardless of reuse: a shared shape designed for interop and a game-specific schema
nobody else imports are equally published. Publication is what makes the source of a
game *complete* on the commons — a Network that referenced an unresolvable
`spacewar:object` would be a fragment, not source.

Publication and baking are different lifecycle stages, not alternatives. At build
time, a manifest pins the exact closure of referenced artifacts; the build tool
resolves them, flattens, and bakes everything into a self-contained Build. Nothing
reads the commons at play time.

---

## Field Types

A Field's type is one of:

| Type kind | Examples | Defined in |
|-----------|----------|-----------|
| Primitive | `int`, `int32`, `uint16`, `bool`, `float`, `fixed16` | [DIGS §Type System](./DIGS_EXPRESSION_LANGUAGE.md#type-system) |
| Game-defined numeric | `spacewar:fixed18`, `quake:x87_double` | Declared with `type` (below); semantics in [DIGS §Game-Defined Types](./DIGS_EXPRESSION_LANGUAGE.md#game-defined-types) |
| Enum | `spacewar:object_state` | Declared with `enum` (below) |
| Record | `runs:vec3`, `spacewar:spawn_request` | Declared with `record` — Records nest as Field types |
| Optional | `doom:mobj?` | Trailing `?`: the Field may hold `none` |
| List | `spacewar:object[24]`, `int[]` | Trailing `[...]`: a homogeneous list |

### Lists and the memory contract

A list type is written three ways:

| Notation | Meaning |
|----------|---------|
| `T[N]` | Exactly N elements, always. |
| `T[max: N]` | At most N elements. |
| `T[]` | Unbounded — any length, determined at runtime. |

A declared bound is an **opt-in memory contract**: it gives the build a bake-time
memory ceiling, which is what lets the game reach allocator-less and
memory-constrained targets. It is *not* required for compliance, and it is not
required for determinism — an unbounded list whose length is computed
deterministically is exactly as deterministic as a bounded one. Spacewar declares
`spacewar:object[24]` and can reach a machine with kilobytes of RAM; a factory game
declaring `entities[]` runs anywhere with enough memory, and a constrained target is
honestly out of its reach. The declaration is a true, portable statement of the
game's memory needs. Changing a declared bound changes the game's rules — see the
RUNS Library's patterns document for the Variant/Port consequences.

There are no pointers and no addresses anywhere in this type system. Values have
sizes and contents; they never have observable locations.

---

## Record Declaration

```
record spacewar:object
  fields:
    state:        spacewar:object_state
    position_x:   spacewar:fixed18
    position_y:   spacewar:fixed18
    velocity_x:   spacewar:fixed18
    velocity_y:   spacewar:fixed18
    angle:        spacewar:fixed18
    lifetime:     int32              default: 0
    fuel:         spacewar:fixed18
    torpedoes:    int32
```

A `record` declaration names the Record and lists its Fields. Each Field is
`name: type`, optionally followed by `default: literal`.

### Defaults and zero values

Every type has a defined **zero value**. A Field without a `default:` carries its
type's zero value in a fresh instance:

| Type | Zero value |
|------|-----------|
| Numeric (primitive or game-defined) | `0` (the +0 pattern, for representations with two zeros) |
| `bool` | `false` |
| Enum | The first declared variant |
| Optional | `none` |
| `T[N]` | N elements, each the zero value of T |
| `T[max: N]`, `T[]` | The empty list |
| Record | Every Field at its default (which may itself be declared) |

A `default:` value must be a literal of the Field's type, or an enum variant name.
Fresh-instance construction is therefore fully deterministic from the schema alone.

### Nesting

A Record may use another Record as a Field type:

```
record runs:vec3
  fields:
    x: float
    y: float
    z: float

record runs:transform
  fields:
    position: runs:vec3
    rotation: runs:quat
```

Nesting is by value: a nested Record is part of its container, not a reference to a
separately-living instance. (Published *schemas* are referenced by ID; *values* at
runtime are plain data, with no identity beyond their contents.)

---

## Enum Declaration

```
enum spacewar:object_state
  empty
  ship
  torpedo
  exploding
  hyperspace_in
  hyperspace_out
```

An enum is a **closed list of named variants**. Variants have no numeric encoding
visible to the language: comparison is by identity (`==`, `!=`), never by ordinal.
In expressions and guards, variants are referenced by qualified name:
`spacewar:exploding`.

Enums earn their keep at Network dispatch: a guard set switching on a closed enum can
be checked for exhaustiveness — every variant handled, none twice. An open-typed
discriminant (an int, a string) can never be proven complete. Prefer closed enums for
anything the wiring routes on. (See
[Network Topology §Guards](./NETWORK_TOPOLOGY.md#guards-and-the-two-laws).)

---

## Game-Defined Type Declaration

Games declare custom numeric types to pin platform-specific arithmetic:

```
type spacewar:fixed18
  storage: int32
  width: 18
  complement: ones
  binary_point: 17
  range: -131071 to 131071
```

The declaration syntax lives in `.runs` schema files; the **arithmetic semantics** —
what `+`, `>>`, `==` mean on these types, including ones-complement behavior, the
two zeros, and floating-point format pinning — are normative in
[DIGS §Game-Defined Types](./DIGS_EXPRESSION_LANGUAGE.md#game-defined-types). The
declaration is the portability contract: a compiler on any host must reproduce the
declared arithmetic exactly.

---

## Formal Grammar

EBNF. Terminals in double quotes; `{ }` is zero-or-more; `[ ]` is optional. INDENT
denotes one two-space indentation level relative to the enclosing line.

```ebnf
schema_file       = { declaration } ;
declaration       = record_decl | enum_decl | type_decl ;

record_decl       = "record" qualified_name NEWLINE
                    INDENT "fields:" NEWLINE
                    { INDENT INDENT field_decl NEWLINE } ;

field_decl        = identifier ":" type_expr [ "default:" default_value ] ;

type_expr         = qualified_name [ "?" ] [ list_spec ] ;
list_spec         = "[" integer_literal "]"
                  | "[" "max:" integer_literal "]"
                  | "[" "]" ;

default_value     = literal | qualified_name ;   (* qualified_name: enum variant *)

enum_decl         = "enum" qualified_name NEWLINE
                    { INDENT identifier NEWLINE } ;

type_decl         = "type" qualified_name NEWLINE
                    { INDENT type_property NEWLINE } ;
type_property     = identifier ":" type_property_value ;
type_property_value = literal | identifier
                    | integer_literal "to" integer_literal ;

literal           = integer_literal | float_literal | fixed_literal
                  | bool_literal ;
qualified_name    = [ identifier ":" ] identifier ;
```

Integer, float, fixed, and bool literal forms are identical to
[DIGS §Literals](./DIGS_EXPRESSION_LANGUAGE.md#literals). The recognized
`type_property` keys and their legal values are normative in DIGS §Game-Defined
Types.

---

## What the Data Layer Deliberately Excludes

| Excluded | Rationale |
|----------|-----------|
| Methods, computed fields | Records are nouns. All computation lives in Processors. |
| References / pointers between runtime values | Values are plain data. Cross-entity relationships are expressed as indices or keys in Fields, interpreted by Processors. |
| Inheritance, subtyping | Composition by nesting only. A shape either matches or it doesn't. |
| Visibility modifiers | Whether a Record crosses the platform boundary is declared where it is *used* (a Network's `requires:`/`produces:`/`state:`), not on the schema. |
| String manipulation types | Strings exist only as enum-comparison and diagnostic literals in DIGS. The data layer has no string Field type in version 1.0. |

---

## Versioning

Schema files carry no per-file version declaration in version 1.0. Evolution rules
match DIGS: future versions are additive only; a schema valid under 1.0 remains valid
and means the same thing forever. Published artifacts are content-addressed, so any
revision is a *new* artifact with a new ID — old references continue to resolve to
exactly what they referenced.

---

*MIT License*
