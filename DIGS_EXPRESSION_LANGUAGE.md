# DIGS — Deterministic Inspectable Game Syntax

> **Layer**: Computation (Processor bodies)
> **File extension**: `.runs-prim`
> **Status**: Design stage — no reference parser or evaluator exists yet. Precise
> enough to implement from; the first implementation is expected to correct it.

## Purpose

DIGS is the language RUNS Processor bodies are written in — the formal syntax in
which game logic is expressed, parsed, and compiled. It is a **named sub-language of
RUNS**, not a separate protocol: RUNS has three layers — data
([Records](./RECORD_SCHEMA.md)), wiring ([Networks](./NETWORK_TOPOLOGY.md)), and
computation — and DIGS is the language of the third. Its expression sub-grammar is
also reused for Network guard expressions. DIGS is governed and versioned here,
inside RUNS, and is scoped to the RUNS world.

DIGS is the enduring artifact in the RUNS model. Runtimes compile it into
platform-specific execution; the source outlives every one of them. The language must
be readable, parseable, and compilable by any future implementer without dependence
on any existing programming language, toolchain, or platform.

Every decision in this specification is driven by one question: *can a solo developer
in a century implement this from scratch, with nothing but this document?*

---

## The Core and the Reach-Contracts

DIGS makes a small set of **core guarantees** that hold for every Processor, always.
Separately, it offers three **reach-contracts** — optional, type-declared properties
a game opts into, each trading some expressible games for some hardware reach. Fusing
these together is the single most common way to misread DIGS, so the boundary is
stated first.

### Core — every DIGS Processor, always

1. **Purity.** A Processor's outputs are a function of its declared inputs. No I/O,
   no observable state between invocations, no external calls. A body may use any
   number of intermediate computations; its semantics are **single-assignment
   dataflow** — the law of the whole RUNS system, *write once, never overwrite*,
   applied inside the body. Imperative-looking reassignment is versioning; a loop is
   a bounded fold. (See §Statements.)

2. **Per-tick totality.** Every Processor terminates on every input. This is a
   *structural* guarantee: every loop's bound is fixed before the loop begins and
   immutable while it runs, and the call graph of Processors is a DAG (no recursion).
   Totality means *per-input termination* — it does **not** mean a static,
   compile-time ceiling on steps. A fold over a list whose length is a runtime value,
   including an unbounded `int`, is total: it terminates for that input, whatever the
   input is.

3. **Evaluation-order determinism.** The grammar and operator precedence fix the
   order of evaluation completely. There is no reassociation, no fused
   multiply-add contraction, no unspecified argument order, and a fixed
   short-circuit order for `and`/`or`. Two conforming implementations evaluate any
   expression identically.

### Reach-contracts — opt-in, declared in types

1. **Arithmetic determinism** (cross-platform bit-exactness). Numeric types whose
   behavior is *fully pinned* — every DIGS primitive numeric type, and every
   game-defined type — produce bit-identical results on every conforming platform.
   This is what makes lockstep multiplayer, replays, and bit-exact ports possible
   across machines. The cost is that the compiler must *emulate* the declared
   arithmetic where the host's native arithmetic differs. A build that instead
   substitutes hardware-native arithmetic is a **divergent compilation**
   (§Divergent Compilation) — a different game, honestly labeled.

2. **Static execution bound.** An optional ceiling on steps per tick, for hard
   real-time targets. *Not implied by totality* — totality says "terminates,"
   not "terminates within N." Where a game wants this property, it follows from
   declaring compile-time list bounds and literal loop counts; tooling can then
   compute a worst-case. This is a tooling concern, never a compliance gate.

3. **Static memory footprint.** Optional list bounds
   ([Record Schema §Lists](./RECORD_SCHEMA.md#lists-and-the-memory-contract)) give a
   bake-time RAM ceiling for allocator-less and constrained targets. *Not implied by
   purity, determinism, or totality.* Unbounded lists are legal DIGS; a game that
   uses them simply cannot reach targets that cannot hold it, and the type says so
   honestly.

The unifying rule: **the declaration IS the contract.** A type declaration pins
arithmetic; a list bound pins memory; a literal loop count pins execution. The core
is mandatory; each contract is reach you opt into by declaration.

A validator checks the **core only**. There is no "too slow," "too big," or
"unbounded" compliance diagnostic — those are target-fit questions answered by the
declarations themselves.

---

## Lexical Structure

### Character Set

Source files are UTF-8 encoded. The language uses only ASCII for keywords,
operators, and identifiers. UTF-8 characters outside the ASCII range may appear only
in comments.

### Line Structure

Logical lines are terminated by a newline character (U+000A). Carriage return
(U+000D) preceding a newline is ignored. A source file must end with a newline.

### Indentation

Block structure is expressed through indentation. Each level of nesting increases
indentation by exactly **two spaces**. Tab characters (U+0009) are illegal in
indentation. A block begins after a colon (`:`) at the end of a line and consists of
all subsequent lines indented deeper than the introducing line.

```
if x > 0:
  let y = x + 1      # indented two spaces: inside the if-block
  output result = y
```

### Comments

A comment begins with `#` and extends to the end of the line. There are no
multi-line comment delimiters. Comments carry no semantic meaning.

### Keywords

The following identifiers are reserved and may not be used as variable names:

```
let  output  if  elif  else  then  for  in  from  with  not  and  or
true  false  none  range  len  processor  inputs  outputs  preconditions
```

### Identifiers

An identifier begins with a letter (a–z, A–Z) or underscore, followed by zero or more
letters, digits (0–9), or underscores. Identifiers are case-sensitive.

### Qualified Names

A qualified name is an optional namespace prefix followed by a colon and an
identifier:

```
spacewar:fixed18      # namespace "spacewar", name "fixed18"
runs:transform        # namespace "runs", name "transform"
ship                  # no namespace (local scope)
```

---

## Literals

### Integer Literals

Decimal, hexadecimal, octal, or binary. Underscores may appear between digits for
readability and carry no semantic meaning.

| Format | Prefix | Example | Value |
|--------|--------|---------|-------|
| Decimal | (none) | `65536` | 65536 |
| Hexadecimal | `0x` | `0x1_0000` | 65536 |
| Octal | `0o` | `0o200000` | 65536 |
| Binary | `0b` | `0b1_0000_0000_0000_0000` | 65536 |

Negative integer literals are written with the unary minus operator: `-65536`.

### Fixed-Point Literals

Decimal numbers with a fractional part, suffixed with `fx`:

```
3.5fx            # 3.5 in the target fixed-point type
-0.25fx
```

The exact binary representation depends on the target fixed-point type. The compiler
rounds to the nearest representable value; ties round toward zero.

### Float Literals

Decimal numbers with a fractional part or an exponent, no suffix:

```
3.14159
1.0e-6
-0.5
```

A float literal denotes the nearest representable value of the target float type
(round-to-nearest, ties-to-even).

### Boolean Literals

```
true
false
```

### List Literals

Square brackets, comma-separated:

```
[1, 2, 3]
[]                # empty list
```

All elements must have the same type.

There are no string literals in version 1.0. Enum values are compared by qualified
name (§Enum Types); the language has no string type.

---

## Type System

### Primitive Types

| Type | Width | Semantics | Cross-platform bit-exact |
|------|-------|-----------|--------------------------|
| `int` | Unbounded | Arbitrary-precision signed integer | Yes — exact |
| `int8` / `int16` / `int32` | 8/16/32 bits | Signed two's-complement, wrapping | Yes — exact |
| `uint8` / `uint16` / `uint32` | 8/16/32 bits | Unsigned, wrapping | Yes — exact |
| `bool` | 1 bit | `true` or `false` | Yes — exact |
| `float` | 64 bits | IEEE 754 binary64, fully pinned (§Float Determinism) | Yes — exact under strict evaluation |
| `fixed16` | 32 bits | 16.16 signed fixed-point, two's-complement | Yes — exact |

**Wrapping** means arithmetic overflow wraps modularly: `int32` max is
`2147483647`; adding 1 yields `-2147483648`.

**Arbitrary-precision** means `int` never overflows. It is the default integer type
when bit-width behavior is not itself part of the game's rules.

Every primitive type is fully pinned — there is no implementation-defined behavior
anywhere in this table. Using these types *is* the arithmetic-determinism contract.

### Fixed-Point Arithmetic (`fixed16`)

`fixed16` stores a 32-bit signed two's-complement value where the lower 16 bits are
the fractional part. The value represented is `raw_bits / 65536`.

| Operation | Semantics |
|-----------|-----------|
| `a + b` | Add raw bits. Wraps on overflow. |
| `a - b` | Subtract raw bits. Wraps on overflow. |
| `a * b` | Multiply to 64-bit intermediate, arithmetic right shift by 16, truncate to 32 bits. |
| `a / b` | Extend `a` to 64 bits by left-shifting 16, signed divide by `b`'s raw bits truncating toward zero, truncate to 32 bits. |
| `a >> n` | Arithmetic right shift of raw bits by `n`. |
| `a << n` | Left shift of raw bits by `n`. |

### Game-Defined Types

Games declare custom numeric types
([Record Schema §Game-Defined Type Declaration](./RECORD_SCHEMA.md#game-defined-type-declaration))
to pin arithmetic that no primitive captures — historical machines, console FPUs,
era-specific float behavior. This section is normative for what those declarations
mean.

#### Integer and Fixed-Point Types

```
type spacewar:fixed18
  storage: int32
  width: 18
  binary_point: 17
  complement: ones
  range: -131071 to 131071
```

| Property | Meaning |
|----------|---------|
| `storage` | The primitive type holding the value in memory. |
| `width` | Significant bits. All arithmetic is performed within this width. |
| `binary_point` | Bit position of the radix point, counted from the right. `binary_point: 17` on an 18-bit type means integer (point to the right of all bits); `binary_point: 9` means 9 integer and 9 fraction bits. |
| `complement` | `ones` or `twos` — the signed representation. |
| `range` | Inclusive range of representable values. |

**Two's-complement types** behave like the fixed-width primitives: modular wrapping,
one zero, arithmetic right shift sign-extends.

**Ones-complement types** are defined as follows:

- **Negation** is bitwise complement within `width`. There are two zero patterns:
  `+0` (all bits clear) and `-0` (all bits set). The literal `0` denotes `+0`; the
  expression `-0` (unary minus on the zero literal) denotes the all-bits-set pattern.
- **Addition** is binary addition within `width` with **end-around carry**: a carry
  out of the top bit is added back into bit 0.
- **Subtraction** `a - b` is `a + (~b)` under the addition rule above.
- **Multiplication and division** are performed on magnitudes; the result's sign is
  the XOR of the operand signs, applied by complement. Division truncates toward
  zero. (Where a historical machine's multiply/divide differ from this — partial
  products, normalization quirks — express the routine explicitly as a Processor
  over the raw bits, exactly as the original did. The declaration covers the common
  contract; the bit-level body is the universal fallback.)
- **Shifts and rotates** operate on the stored bit pattern within `width`. `>>` is
  arithmetic (fills with the sign bit), `<<` fills with zeros, `>>>` rotates within
  `width`.
- **Equality** (`==`, `!=`) on a game-defined type is **bit-pattern equality**.
  `+0 == -0` is therefore `false`. This is deliberate: ports of ones-complement
  software routinely use `-0` as a distinguished sentinel, and the language must be
  able to express that test. A "numerically zero" test is written explicitly:
  `x == 0 or x == -0`.
- **Ordering** (`<`, `<=`, `>`, `>=`) compares numeric values; `+0` and `-0` are
  numerically equal, so `-0 < 0` is `false`.
- A negative-sign test that must match the hardware's sign-bit behavior (where `-0`
  counts as negative) is written on the bits: `(x >> 17) & 1 == 1` for an 18-bit
  type. Bitwise operators give complete access to the stored pattern, so every
  historical test is expressible exactly.

The `binary_point` property is *documentation with teeth*: it does not change the
arithmetic (which operates on raw bit patterns exactly as historical code did), but
it declares the intended interpretation, and fixed-point literals (`1.5fx`) are
encoded according to it. Algorithms that move the binary point between conventions —
as 1960s code constantly does — express the moves as explicit shifts, exactly as the
original source did.

#### Floating-Point Types

```
type quake:x87_double
  format: ieee754
  storage_width: 64
  compute_width: 80
  significand: 64
  exponent: 15
  rounding: nearest_ties_to_even
  intermediate_rounding: on_store
  subnormals: enabled
```

| Property | Meaning |
|----------|---------|
| `format` | The floating-point standard. Only `ieee754` is defined. |
| `storage_width` | Bit width when stored (64 for binary64). |
| `compute_width` | Bit width of intermediates. When it exceeds `storage_width`, intermediates keep extra precision until stored — x87-style behavior. |
| `significand` | Significand bits, including the implicit leading 1 (53 for binary64, 64 for 80-bit extended). |
| `exponent` | Exponent bits (11 for binary64, 15 for 80-bit extended). |
| `rounding` | `nearest_ties_to_even` (default), `toward_zero`, `toward_positive`, `toward_negative`. |
| `intermediate_rounding` | `per_operation` (round after every operation, SSE-style) or `on_store` (defer until stored, x87-style). |
| `subnormals` | `enabled` (IEEE 754 compliant) or `flush_to_zero`. |

The primitive `float` is sugar for: `format: ieee754, storage_width: 64,
compute_width: 64, significand: 53, exponent: 11, rounding: nearest_ties_to_even,
intermediate_rounding: per_operation, subnormals: enabled`.

#### The Portability Principle

Arithmetic on game-defined types follows the declared rules, not the host machine's.
A compiler targeting a two's-complement machine emulates ones-complement for types
that declare `complement: ones`. A compiler targeting ARM emulates x87
extended-precision intermediates for types that declare `compute_width: 80`. The
type declaration does not require the original hardware — it requires the compiler
to reproduce the declared behavior on whatever hardware exists. A game written
against any platform's native arithmetic runs on any future platform through this
mechanism alone.

### Enum Types

Enums are declared in schema files
([Record Schema §Enum Declaration](./RECORD_SCHEMA.md#enum-declaration)). In
expressions, variants are referenced by qualified name:

```
if object.state == spacewar:exploding:
  # ...
```

Enum values have no numeric encoding visible to the language. Comparison is by
identity (`==`, `!=`) only — there is no ordering on enum values.

### Optional Types

A field declared with a trailing `?` may hold `none`:

```
inputs:
  inflictor: doom:mobj?     # may be absent
```

An optional value must be narrowed before its fields are accessed. Within the
`then`/taken branch of a test against `none`, the value's type narrows to its
non-optional type:

```
let damage = if inflictor != none then inflictor.damage else 0
```

Accessing a field on an un-narrowed optional is a compile-time error.

### List Types

A trailing `[...]` declares a homogeneous list. Bounds (`[N]`, `[max: N]`, `[]`) and
their meaning — including that **unbounded lists are legal** and a declared bound is
an opt-in memory contract — are defined in
[Record Schema §Lists](./RECORD_SCHEMA.md#lists-and-the-memory-contract). Lists are
pure values: they have lengths and contents, never observable addresses. `len(xs)`
returns a list's length as `int`.

### Record Types

Record types are declared in schema files. In expressions, Records are accessed via
field paths (`object.position_x`), constructed with a record constructor
(§Record Construction), and updated with `with` (§Record Update).

---

## Expressions

### Arithmetic Operators

| Operator | Meaning | Types |
|----------|---------|-------|
| `+` `-` `*` `/` | Add, subtract, multiply, divide | All numeric types |
| `%` | Modulo (remainder) | Integer types only |
| `-` (unary) | Negation | All numeric types |

Integer division truncates toward zero: `7 / 2 = 3`, `-7 / 2 = -3`. The remainder
takes the dividend's sign: `-7 % 2 = -1`. Division and modulo require a nonzero
divisor — `divisor != 0` is an implicit precondition (§Preconditions).

### Comparison Operators

`==` `!=` `<` `>` `<=` `>=` produce `bool`. Comparison is valid between values of
the same type only; comparing different types is a compile-time error. Comparisons
do not chain (`a < b < c` is a syntax error). On game-defined types, `==`/`!=` are
bit-pattern equality and the ordering operators are numeric (§Game-Defined Types).

### Logical Operators

`and`, `or`, `not`, on `bool` operands only. `and` evaluates its right operand only
if the left is `true`; `or` only if the left is `false`. Short-circuit order is part
of the language's evaluation-order determinism — no implementation may reorder it.

### Bitwise Operators

| Operator | Meaning | Types |
|----------|---------|-------|
| `&` `\|` `^` | AND, OR, XOR | Integer, fixed-point, and game-defined numeric types |
| `~` (unary) | Complement | Same |
| `>>` | Arithmetic right shift (sign-extending) | Same |
| `<<` | Left shift (zero-filling) | Same |
| `>>>` | Rotate right | Fixed-width types only |

On fixed-width and game-defined types, bitwise operators act on the stored bit
pattern within the declared width, and the result is a value of the same type. On
unbounded `int`, `&`/`|`/`^`/`~` follow the infinite two's-complement convention
(`~x = -x - 1`); `<<` is exact multiplication by a power of two and `>>` is flooring
division. Shift counts must be non-negative `int` values; a shift count `>= width`
on a fixed-width type yields the pattern fully shifted out (all sign bits for `>>`,
zero for `<<`). `>>>` rotates within the declared width; rotating an unbounded `int`
is a compile-time error.

### Operator Precedence

From highest to lowest binding:

| Level | Operators | Associativity |
|-------|-----------|---------------|
| 1 | `not`, `-` (unary), `~` | Right |
| 2 | `*`, `/`, `%` | Left |
| 3 | `+`, `-` | Left |
| 4 | `<<`, `>>`, `>>>` | Left |
| 5 | `&` | Left |
| 6 | `^` | Left |
| 7 | `\|` | Left |
| 8 | `==`, `!=`, `<`, `>`, `<=`, `>=` | None (no chaining) |
| 9 | `and` | Left |
| 10 | `or` | Left |

Parentheses override precedence. The formal grammar (§Formal Grammar) encodes
exactly this table; if a discrepancy is ever found between the two, **this table
governs** and the grammar is in error. Note that comparisons bind *looser* than
bitwise operators: `x & 3 == 1` parses as `(x & 3) == 1`.

### Field Access

```
object.position_x
config.heavy_star
```

Field access on a non-Record value, or of a field name absent from the Record's
schema, is a compile-time error.

### Indexed Access

```
entities[0]
objects[i]
```

The index must be an integer. `0 <= index < len(list)` is an implicit precondition
(§Preconditions).

### Sub-Processor Calls

A Processor may invoke another Processor as a pure function call:

```
let sin_result = spacewar:sin(angle)
let product = spacewar:multiply(a, b)
```

Arguments are positional, matching the callee's declared `inputs:` order. The call
returns the callee's declared outputs as a **record-shaped value** — a plain value
whose fields are the outputs, accessed with the dot operator:

```
let sqrt_out = spacewar:sqrt(value)
let root = sqrt_out.root
```

It is a *value*, not a Record: it lives as a body-local, it is never wired, never
persisted, never dispatched on. Composition by call is **body-plane** composition —
a pure DAG of calls whose intermediates are anonymous locals — and it collapses into
one opaque Network transition when viewed from outside. Promoting such an
intermediate into an actual Record is exactly the act of moving it to the Network
plane, where it becomes inspectable, dispatchable, and able to persist across ticks.
(See [Network Topology §The Two Composition Planes](./NETWORK_TOPOLOGY.md#the-two-composition-planes).)

**Recursion prohibition**: the call graph of all Processors must form a DAG. A
Processor may not call itself, directly or through any chain. The compiler verifies
this by topological sort; a cycle is a compile-time error. This applies to the
body-plane call graph only — the Network may invoke the same Processor at many
points per tick (that is iteration, not recursion).

### Record Construction

A record constructor builds a fresh record value of a declared type:

```
let request = spacewar:spawn_request {
  position_x = ship.position_x,
  position_y = ship.position_y,
  velocity_x = launch_vx,
  velocity_y = launch_vy
}
```

Fields not named take their schema defaults
([Record Schema §Defaults](./RECORD_SCHEMA.md#defaults-and-zero-values)). Naming a
field not in the schema, or assigning a value of the wrong type, is a compile-time
error.

### Record Update (`with`)

`with` produces a new record value with the named fields changed:

```
let updated = object with { velocity_x = 0, velocity_y = 0, state = spacewar:exploding }
```

The original value is not modified — there is no in-place mutation anywhere in the
language. Fields not mentioned keep their values from the original. A compiler may
implement `with` as in-place mutation when it can prove the original is never read
afterward; that is invisible to the semantics.

### Conditional Expression (Inline)

```
let max_val = if a > b then a else b
```

Both branches must produce the same type. `else` is required — a conditional
expression always produces a value.

### List Concatenation

```
let extended = existing_list ++ [new_element]
```

`++` concatenates two lists of the same element type. If the result is ultimately
written to an output whose declared bound it exceeds, that is a precondition
violation of the write (§Preconditions).

### `len`

`len(xs)` returns the length of a list as `int`. Total for every list.

---

## Statements

### The Single-Assignment Law

Every binding in a DIGS body is written once. What looks like mutation is
**versioning**: a new binding for a new value, the old value unchanged. What looks
like a stateful loop is a **bounded fold**: an accumulator passed hand-to-hand
through a fixed number of steps. This is the same law the Network obeys at the
wiring scale — *write once, never overwrite; pass shared things hand-to-hand* — and
it is what makes a body trivially lowerable to SSA form and checkable by the same
machinery that checks the Network.

### `let` Binding

Introduces an immutable name bound to a value:

```
let velocity = old_velocity + acceleration
```

A `let` name is never reassigned. A subsequent `let` of the same name in the same
scope **shadows** it — a new, independent binding; the previous value is unchanged
and no longer referenceable:

```
let x = 5
let x = x + 1    # new 'x' is 6; the binding holding 5 is out of scope
```

Shadowing exists for sequential refinement of a value through a pipeline of steps
without inventing a fresh name per step. It is semantically identical to fresh
temporaries.

### `output` Statement

Assigns a value to a declared output:

```
output position_x = new_x
output object = object with { state = spacewar:ship }
output result.x = a.x + b.x          # per-field assignment
output entities[i] = updated_entity  # indexed assignment into a list output
```

Every declared output must be fully assigned **exactly once along every execution
path** — whole-value assignment, or per-field/per-index assignments that jointly
cover it. An output left unassigned on some path is a compile-time error: the
Processor would have undefined output for those inputs, violating purity. Assigning
the same output (or the same field of it) twice on one path is equally an error —
write once.

### `if` / `elif` / `else` Statement

```
if health <= 0:
  output state = spacewar:dead
elif health < 20:
  output state = spacewar:critical
else:
  output state = spacewar:alive
```

The condition must be `bool`. When outputs are assigned inside conditional blocks,
the compiler verifies every output is assigned on all paths.

A body `if` may freely *compute* facts — including writing a state tag that the
Network will later route on. Producing a discriminant is a Processor's job; only
*reading* one to choose between Processors is routing, and routing belongs to
Network guards, not bodies. (See
[Network Topology §Guards](./NETWORK_TOPOLOGY.md#guards-and-the-two-laws).)

### `for` Statement

Bounded iteration — a fold over a finite list:

```
for entity in entities:
  ...body...
```

The loop variable binds to each element in order, index 0 to `len - 1`, exactly once
per element. The bound — the list value being iterated — is **fixed before the loop
begins and immutable while it runs**. There is no `break`, no `continue`, no early
return; the body may not modify the list being iterated. These structural rules are
what make totality checkable by inspection: the loop terminates because the bound
was fixed and nothing in the body can move it.

The bound may be a runtime value. Iterating a list whose length is unknown until the
tick runs is total — it terminates for every input. A *compile-time* iteration
ceiling is the optional execution-bound reach-contract, never a language
requirement.

**`range(n)`**: for counted iteration, `range(n)` produces the integers `0` to
`n-1`:

```
for step in range(18):
  # step = 0, 1, ..., 17
```

`n` is any `int` expression, evaluated once, before iteration; `range(n)` for
`n <= 0` is empty. `n` may be a runtime value — `range` is total for every input.

**Accumulators (`from` clause)**: when the body carries state across iterations, a
`from` clause declares accumulated names with initial values:

```
# Binary digit-by-digit square root: 18 iterations
for step in range(18) from result = 0, remainder = 0, num = value:
  let result = result << 1
  let t = (remainder << 2) | ((num >> 16) & 3)
  let num = (num << 2) & 262143
  let d = (result << 1) + 1
  let result = if t != 0 and d <= t then result + 1 else result
  let remainder = if t == 0 then remainder elif d <= t then t - d else t

output root = result
```

Semantics:

1. Before iteration 0, each accumulated name is bound to its initial value.
2. During each iteration, accumulated names hold their values from the end of the
   previous iteration.
3. `let` bindings in the body may shadow accumulated names; the final binding of
   each name at the end of the body becomes its value for the next iteration.
4. A name not shadowed in an iteration keeps its start-of-iteration value.
5. After the loop, the final values of all accumulated names are in scope.

The `from` clause is the bounded fold made explicit — the accumulator passed
hand-to-hand. It is syntactic sugar for the unrolled chain of `let` shadows and adds
no expressive power, which is why totality survives it untouched.

**Producing a transformed collection**: a body never structurally edits a
collection in place. It folds the input into a fresh output:

```
for entity in entities from survivors = []:
  let survivors = if entity.health > 0
    then survivors ++ [doom:move_entity(entity).entity]
    else survivors

output entities = survivors
```

Removal, insertion, and reordering are all expressible this way as pure folds. No
deferred mutation, no hidden post-pass: the output list is the new truth, produced
by the Processor like any other value.

A body fold that routes per-element by a state tag —
`for entity in entities: if entity.state == spacewar:torpedo: ...` — is a legal,
honest **mid-decomposition idiom**: the Processor is a monolith and the Network sees
one opaque transition, so there is no Network routing being usurped. As the monolith
is decomposed and the per-state behaviors become distinct Processors, that routing
belongs on Network arcs as guards. Both stations are fully compliant; granularity is
a spectrum, not a compliance gate.

---

## Processor Structure

A complete Processor definition:

```
#! runs-prim 1.0

processor spacewar:gravity
  inputs:
    object:   spacewar:object
    config:   spacewar:game_config
    consts:   spacewar:game_constants
  outputs:
    gravity_x: spacewar:fixed18
    gravity_y: spacewar:fixed18
    object:    spacewar:object
  preconditions:
    object.state == spacewar:ship

  if config.disable_gravity:
    output gravity_x = 0
    output gravity_y = 0
    output object = object
  else:
    let xn = object.position_x >> 11
    let yn = object.position_y >> 11
    let r_sq = xn * xn + yn * yn - consts.star_capture_radius

    if r_sq <= 0:
      let zeroed = object with { velocity_x = 0, velocity_y = 0 }
      output gravity_x = 0
      output gravity_y = 0
      if config.star_teleport:
        output object = zeroed with { position_x = 131071, position_y = 131071 }
      else:
        output object = zeroed with { state = spacewar:exploding, lifetime = -8 }
    else:
      let r_sq_full = r_sq + consts.star_capture_radius
      let sqrt_result = spacewar:sqrt(r_sq_full)
      let r_scaled = sqrt_result.root >> 9
      let product = spacewar:multiply(r_scaled, r_sq_full)
      let divisor = product.low >> 2
      let div = if config.heavy_star then divisor else divisor >> 2

      if div == 0:
        output gravity_x = 0
        output gravity_y = 0
        output object = object
      else:
        output gravity_x = (0 - object.position_x) / div
        output gravity_y = (0 - object.position_y) / div
        output object = object
```

### Version Declaration

The first line of every Processor body:

```
#! runs-prim 1.0
```

This declares the language version the body is written in. Version 1.0 is the
"forever" version — it will never be removed from the specification.

### Inputs and Outputs

`inputs:` declares the read-only parameters; `outputs:` declares the values the
Processor must produce. Both are typed, named fields. An input may share a name with
an output: the Processor receives a value and produces a successor version of it.
The input is never mutated — the same-name pair is just the coarsest one-link
thread, `object_v0 → object_v1`, written without the subscripts.

### Preconditions

The `preconditions:` block declares what must hold when the Processor is invoked:

```
preconditions:
  divisor != 0
  object.state == spacewar:ship
```

A precondition is the body-side half of a contract whose other half lives on the
Network: in a well-formed Network, the **guards on the arcs routing into this
Processor discharge its preconditions** — the guard asserts the fact on the wiring
plane, the precondition assumes it on the body plane. DIGS's determinism contract is
therefore conditional: *same inputs, same outputs, given the preconditions hold*.

A violated precondition is not a runtime feature and has no defined runtime
behavior, because it cannot occur in a conforming game: it means the Network is
malformed — its guards failed to discharge an assumption. Detecting that is a
**tooling** concern: a validator proves (or a test suite checks) that every arc into
a Processor implies its preconditions. Two preconditions are implicit on every
body: divisors are nonzero, and list indices are in range.

The compiler may exploit preconditions for optimization — if `divisor != 0` is
contracted, the emitted code may skip a zero check.

---

## Determinism

This section is normative. It defines the contract between the language and its
compilers.

### The Strict Evaluation Rule

The compiler MUST produce code that computes the **exact same output** as the
reference evaluation of the body, given the same inputs. Formally: let
`E(body, inputs)` be the result of evaluating the body by the rules in this
specification. For any compiler `C` and any valid inputs `I` satisfying the
preconditions:

```
C(body)(I) == E(body, I)
```

This is not approximate. It is bit-exact for all types, including float, under
strict evaluation. The body is the specification — a compiler never needs to
understand *why* an algorithm scales a value; it needs only to execute the algorithm
exactly.

### Permitted Optimizations

A compiler MAY:

- Inline sub-Processor calls
- Unroll loops
- Reorder independent `let` bindings (no data dependence between them)
- Eliminate dead code (unreachable branches, unused bindings)
- Replace an algorithm with a different one **if and only if** the replacement
  produces identical output for every input within the declared types' ranges that
  satisfies the preconditions

A compiler MAY NOT:

- Substitute a platform-native function for a Processor body unless it can prove
  bit-exact equivalence over all valid inputs
- Reorder operations with data dependencies
- Change the evaluation order of `and` / `or`
- Reassociate arithmetic or contract `a * b + c` into a fused multiply-add

### Float Determinism

All float operations in DIGS (`+`, `-`, `*`, `/`, negation, comparison) are IEEE 754
basic operations, required by that standard to be correctly rounded. DIGS has no
built-in transcendental functions — games that need `sin`, `sqrt`, `exp` implement
them as Processors out of basic arithmetic, which keeps them exactly as
deterministic as everything else. There is no fused multiply-add and no compiler
latitude to reassociate.

The historical sources of cross-platform float divergence — intermediate precision,
subnormal handling, rounding mode — are exactly the properties a float type
declaration pins (§Game-Defined Types). The primitive `float` is itself fully
pinned. Under strict evaluation, every DIGS numeric type produces bit-identical
results on every conforming platform; the emulation cost on hardware whose native
arithmetic differs is the price of that reach, and divergent compilation
(§below) is the honest escape hatch where a build chooses not to pay it.

### Divergent Compilation

A build MAY deviate from strict evaluation for specific Processors — substituting a
hardware sqrt, native floating point, a smaller table — to fit a constrained target
or buy speed. Such a build is a **divergent compilation**. Because its Rules-level
output differs from strict evaluation, it is by definition a **variant of the game,
not the game itself** — the honest antonym of strict, declared rather than hidden.

Every divergence must be recorded in a machine-readable **deviation manifest**
shipped with the build:

```yaml
deviation_manifest:
  game: <qualified name of the root Network>
  source_closure: <content hash of the pinned manifest the build was baked from>
  target: <platform identifier>
  deviations:
    - processor: spacewar:sqrt
      substitution: platform.native_sqrt
      justification: "Strict 18-iteration sqrt exceeds the per-frame budget on this target"
      impact: "Results may differ by ±1 in the low bit from strict evaluation"
```

Required fields per deviation: `processor` (the qualified name deviated from),
`substitution` (what ran instead), `justification` (why), `impact` (the honest
statement of how output can differ). The manifest is the divergent variant's diff
from canon — the build-time face of the asserted-vs-verified discipline: a build
either matches strict evaluation bit-for-bit, or it says exactly where it doesn't.

Divergent compilation is never the default, and a manifest does not make the
divergence "still the same game." It makes it an honestly labeled variant.

---

## Formal Grammar

EBNF. Terminals in double quotes; `{ }` is zero-or-more; `[ ]` is optional. INDENT
denotes one two-space level relative to the enclosing line. This grammar encodes the
precedence table in §Operator Precedence; the table governs if they ever disagree.

```ebnf
processor_file    = version_decl processor_decl ;

version_decl      = "#!" "runs-prim" version_number NEWLINE ;
version_number    = DIGIT+ "." DIGIT+ ;

processor_decl    = "processor" qualified_name NEWLINE
                    inputs_block
                    outputs_block
                    [ preconditions_block ]
                    body ;

inputs_block      = INDENT "inputs:" NEWLINE
                    { INDENT INDENT field_decl NEWLINE } ;

outputs_block     = INDENT "outputs:" NEWLINE
                    { INDENT INDENT field_decl NEWLINE } ;

preconditions_block = INDENT "preconditions:" NEWLINE
                      { INDENT INDENT expression NEWLINE } ;

field_decl        = identifier ":" type_expr ;
type_expr         = qualified_name [ "?" ] [ "[" [ "max:" ] [ integer_literal ] "]" ] ;

body              = { statement NEWLINE } ;
statement         = let_stmt | output_stmt | if_stmt | for_stmt ;

let_stmt          = "let" identifier "=" expression ;

output_stmt       = "output" output_target "=" expression ;
output_target     = field_path [ "[" expression "]" ] ;

if_stmt           = "if" expression ":" NEWLINE INDENT body
                    { "elif" expression ":" NEWLINE INDENT body }
                    [ "else" ":" NEWLINE INDENT body ] ;

for_stmt          = "for" identifier "in" expression
                    [ "from" accum_init { "," accum_init } ]
                    ":" NEWLINE INDENT body ;
accum_init        = identifier "=" expression ;

expression        = or_expr ;
or_expr           = and_expr { "or" and_expr } ;
and_expr          = comparison_expr { "and" comparison_expr } ;
comparison_expr   = bitor_expr [ ( "==" | "!=" | "<" | ">" | "<=" | ">=" ) bitor_expr ] ;
bitor_expr        = bitxor_expr { "|" bitxor_expr } ;
bitxor_expr       = bitand_expr { "^" bitand_expr } ;
bitand_expr       = shift_expr { "&" shift_expr } ;
shift_expr        = additive_expr { ( "<<" | ">>" | ">>>" ) additive_expr } ;
additive_expr     = multiplicative_expr { ( "+" | "-" ) multiplicative_expr } ;
multiplicative_expr = unary_expr { ( "*" | "/" | "%" ) unary_expr } ;
unary_expr        = [ "-" | "not" | "~" ] postfix_expr ;

postfix_expr      = primary_expr
                    { "." identifier
                    | "[" expression "]"
                    | "(" [ arg_list ] ")"
                    | "with" "{" field_assign { "," field_assign } "}" } ;

arg_list          = expression { "," expression } ;

primary_expr      = qualified_name [ record_ctor ]
                  | integer_literal
                  | float_literal
                  | fixed_literal
                  | bool_literal
                  | "none"
                  | list_literal
                  | "(" expression ")"
                  | "if" expression "then" expression "else" expression
                  | "range" "(" expression ")"
                  | "len" "(" expression ")" ;

record_ctor       = "{" field_assign { "," field_assign } "}" ;
field_assign      = identifier "=" expression ;

list_literal      = "[" [ expression { "," expression } ] "]" ;

field_path        = identifier { "." identifier } ;
qualified_name    = [ identifier ":" ] identifier ;

integer_literal   = DECIMAL_DIGITS
                  | "0x" HEX_DIGITS
                  | "0o" OCTAL_DIGITS
                  | "0b" BINARY_DIGITS ;
float_literal     = DECIMAL_DIGITS "." DECIMAL_DIGITS
                    [ ( "e" | "E" ) [ "+" | "-" ] DECIMAL_DIGITS ] ;
fixed_literal     = DECIMAL_DIGITS "." DECIMAL_DIGITS "fx" ;
bool_literal      = "true" | "false" ;
```

List concatenation `++` is additive: it parses at level 3 with `+`/`-`, restricted
to list operands.

---

## Deliberate Exclusions

The language does not and will never express:

| Excluded | Rationale |
|----------|-----------|
| Unbounded loops (`while`, `loop`) | Totality is structural: every loop's bound is fixed before it runs. Unbounded computation across time lives in the Network's tick feedback, where it is visible in the topology — never inside a body. |
| General recursion | Same: the call graph is a DAG, so termination is inspectable. |
| I/O of any kind | A body computes. Reaching hardware — render, audio, input, transport — is the business of platform-side Processors whose bodies are not DIGS (see Network Topology §The Platform Boundary). |
| Observable memory | No pointers, no addresses, no manual allocation or free. Values have sizes and contents, never locations. Pure values of runtime-determined size are legal; whatever allocator realizes them is an invisible implementation detail. |
| Global mutable state | All data flows through declared inputs and outputs. |
| Platform intrinsics | No OS calls, no hardware-specific operations. Platform-specific *arithmetic* is reached through game-defined types, which any platform must emulate exactly. |
| Strings | The language is not a text processor. No string type, no string literals in 1.0. |
| Dynamic dispatch | All calls are statically resolved. Routing is the Network's job, on guarded arcs. |
| Exceptions | No `throw`/`catch`. Contracts are preconditions, discharged by Network guards; everything else is total. |

---

## Versioning

### The `#!` Declaration

Every Processor body begins with `#! runs-prim <major>.<minor>`. An implementation
claiming support for version `X.Y` must support all `X.Z` with `Z < Y` and all
`W.0` with `W < X`.

### Evolution Rules

1. **Additive only** — future versions may add constructs; they never remove or
   change the semantics of existing ones.
2. **Version 1.0 is forever** — a program valid under 1.0 is valid under every
   future version and never changes meaning.
3. **Major version for breaking changes** — none is promised or planned; if one ever
   happens, 1.0 source remains readable and archivable.
4. **No feature flags** — the version number is the only gate. No pragmas, no
   compiler flags, no conditional compilation.

---

## The Oracle: Reference Tooling and Test Vectors

Until output is checked against an oracle, every claim that a Processor "is
faithful" or "is correct" is **asserted** — someone read it and judged it. The
claim becomes **verified** only when a reference evaluator runs the body against
test vectors derived from ground truth and matches bit-for-bit. The minimum tooling
that makes verification possible:

1. **Reference grammar** — this document's EBNF, published as a machine-readable
   grammar file. If the two ever disagree, the published grammar file governs the
   syntax (and the precedence table in §Operator Precedence governs both).
2. **Reference parser** — a single-file parser from `.runs-prim` text to an AST. No
   dependencies beyond its host language's standard library.
3. **Reference evaluator** — a tree-walking interpreter evaluating a Processor's AST
   against given inputs. The evaluator is the oracle for every compiler: an
   optimizing compiler's output must match it for all valid inputs.
4. **Test vectors** — per Processor, input/output pairs derived from ground truth
   (for a port: the original platform's emulated behavior). Example vectors for the
   first conversion:

   | Processor | Vector |
   |-----------|--------|
   | `spacewar:sqrt` | `sqrt(2048) → root = 23168` (result carries a ×512 scale; binary point between bits 8 and 9) |
   | `spacewar:sin` | `sin(0) → 0`; `sin(0o62210) → 0o377777` |
   | `spacewar:random` | the known rotate-XOR-add sequence from each seed |
   | `spacewar:multiply` | `multiply(32, 32) → high = 0, low = 1024` |

A Processor without a body is a promise without proof; a body without vectors is
asserted, not verified. Both halves are deliverables.

---

*MIT License*
