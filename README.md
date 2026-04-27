# nota

A text data format. Two delimiter pairs, two string forms, two
sigils, no parser keywords beyond the literals `true` / `false`
/ `None`. Records are positional; field names live in the Rust
schema, not the text. The Rust implementation is
[nota-serde](https://github.com/LiGoldragon/nota-serde).

The "no keywords" rule applies to the **parser** — there are no
reserved words like `SELECT` or `IF` that the parser dispatches
on. **Schemas are typed by the consumer**: kind names, enum
variants (e.g. `RelationKind { DependsOn, Contains, … }`), and
field types are part of the schema, not the parser. Adding new
typed kinds and variants is exactly what consumers like
[signal](https://github.com/LiGoldragon/signal) do; the parser
stays small.

`nota` is the data-layer format in the sema ecosystem.
[nexus](https://github.com/LiGoldragon/nexus) is the superset
messaging protocol — every valid nota text is also valid nexus.

This repo is spec-only.

---

## Delimiters

Two pairs. Role is fixed at the opening token — the grammar is
first-token-decidable at every choice point. No interior scanning.

| Pair | Role | Example |
|---|---|---|
| `( )` | Record (named composite value) | `(Point 3.0 4.0)` |
| `[ ]` | Sequence (heterogeneous list) | `[1 2 3]`, `[("k" v) ("k2" v2)]` |

## Strings

Two forms. Quote-bounded.

| Form | Role | Example |
|---|---|---|
| `" "` | String (inline) | `"hello world"` |
| `""" """` | String (multiline, auto-dedented) | `"""line one\nline two"""` |

Inline strings allow `\"`, `\\`, `\n`, `\t` escapes. Multiline
strings take their content verbatim — escapes pass through
unchanged. Dedent strips the shortest common leading-whitespace
prefix from every non-empty line.

## Sigils

Two. Each has a fixed lexical position — no interior scanning.

| Sigil | Role | Position |
|---|---|---|
| `;;` | Line comment | to end of line |
| `#` | Byte-literal prefix | `#a1b2c3` (3 bytes) |

**Comments carry no load-bearing data.** The parser discards them.
Information that must be communicated has a typed home in the
schema — never use comments, whitespace, ordering accidents, or
any other parser-discarded channel to carry meaning. If a design
wants to "put it in a comment for the reader to find," the design
is missing a field; fix the design.

## Identifiers

Three lexer classes, distinguished by first character. `_` is
treated as a letter in all classes, including as a leading
character — `_foo` counts as camelCase-kindred (the conventional
class for Rust field names starting with `_`).

| Class | Shape | Role |
|---|---|---|
| PascalCase | First char uppercase | Type / variant names (structural) |
| camelCase | First char lowercase or `_`; alnum / `_` body (no `-`) | Field names (in schema), instance names |
| kebab-case | First char lowercase or `_`; alnum / `_` / `-` body, at least one `-` | Titles, hash-IDs, tags |

The parser dispatches on class. Classes are disjoint — no
identifier is valid in more than one class. The presence of a
`-` distinguishes kebab from camel; otherwise a lowercase-or-`_`
leader is camelCase.

**The PascalCase rule is enforced at parse time** in record /
unit struct / newtype / variant *head* positions — the
identifier immediately after `(` in `(Foo …)`, the bare token
of a unit `Foo`, or the variant name. A bare lowercase
identifier in head position is rejected with a "name must be
PascalCase" error rather than failing later as a schema-mismatch.

The rule applies to head positions only. In *value* positions,
every ident-class token (PascalCase, camelCase, kebab-case)
remains a valid bare-identifier string when the schema expects
`String` — `(Tag User)`, `(Person nexus)`, `(Tag lojix-schema)`
all work.

## Literals

| Form | Type |
|---|---|
| `42`, `-7`, `0xFF`, `0b1010`, `0o755`, `1_000_000` | Integer |
| `3.14`, `-0.5` | Float (always `.` in canonical form) |
| `"text"`, `"""multiline"""`, or a bare identifier | String |
| `true`, `false` | Bool |
| `#<even-hex>` (e.g. `#a1b2c3`) | Bytes |
| `#<64 hex chars>` | Blake3 hash (canonical length) |

Hex bytes are lowercase and even-length. No typed numeric
suffixes (`42u32`) — the receiving type determines the value
type.

## Bare-identifier strings

Where a string is expected by the schema, a bare identifier
may be written in place of `"identifier"`. This keeps config
files readable for the common case where string values follow
identifier rules:

```nota
;; these three forms are equivalent when the schema says String
(Package "nota-serde")
(Package nota-serde)
(Package """nota-serde""")
```

A bare identifier qualifies when its content is a non-empty
ident-class token (PascalCase / camelCase / kebab-case) and is
*not* one of the reserved keywords `true`, `false`, `None` —
those always mean the bool / Option::None they name.

Bare form is **ASCII-only**. Non-ASCII content (e.g. `café`,
emoji, RTL text) always emits through `" "` or `""" """`. The
path separator `:` never appears in a bare string — use `" "`
for content containing colons.

Single-character strings and `char` values follow the same rule:
`'a'` emits as bare `a`; both bare and quoted forms deserialize
back to `char`.

**Canonical form emits bare when eligible.** Serialising the
string `"nota-serde"` through `to_string` produces `nota-serde`,
not `"nota-serde"`. Strings containing spaces, `"`, newlines,
digits as the first char, or reserved-word content always round-
trip through the `" "` / `""" """` forms.

## Path syntax

`:` separates nested names:

```
Char:Upper:A
Schema:Type:Field
example:item
```

## Records

A record is a named composite value. The first token is a
PascalCase type name; the rest are the fields, **in
source-declaration order from the Rust schema**. No field names
appear in the text — positions determine field identity.

```nota
(Point 3.0 4.0)
```

Where `Point` is defined `struct Point { horizontal: f64, vertical: f64 }`,
position 1 is `horizontal`, position 2 is `vertical`.

### Nested records

```nota
(Line
  (Point 0.0 0.0)
  (Point 10.0 10.0))
```

### Newtype structs

Rust single-field unnamed structs (`struct Id(u32)`) are allowed
and serialize *wrapped* by default, with one positional value:

```nota
(Id 42)
```

The wrapped form preserves the type marker on the wire — useful
for newtypes whose appearance in text shouldn't be confused with
the underlying primitive.

**Newtypes of primitives may opt into a bare form** with serde's
`#[serde(transparent)]` attribute. The inner value emits bare,
and bare integer (or string, etc.) literals in matching schema
positions are accepted as the newtype:

```rust
#[serde(transparent)]
pub struct Slot(pub u64);
```

```nota
;; with transparent: bare integer in slot positions
(Edge 100 101 Flow)
;; without transparent the same record would read:
(Edge (Slot 100) (Slot 101) Flow)
```

This is the same schema-position-determines-type discipline as
bare-identifier strings (above): when the schema expects `Slot`
at a position, the parser accepts a bare `42` and constructs
`Slot(42)`; the serializer emits `42` for canonical output. Use
`#[serde(transparent)]` for primitive-shaped newtypes that
appear in heavily-used positions where wrapping hurts
readability — `Slot`, `Revision`, and content-hash types are the
canonical cases. Domain newtypes whose type marker carries
meaning to a human reader (e.g. `EmailAddress(String)`,
`Url(String)`) should stay wrapped.

### Multi-field unnamed structs are forbidden

`struct Pair(i32, i32)` (tuple struct with more than one field)
has no field names in the schema — position cannot be mapped to
meaning. nota rejects this at serialize time. Use a named-field
struct instead.

Single-field enum variants (`Square(f64)`) are allowed and treated
like newtypes: `(Square 5.0)`. Multi-field tuple variants are
forbidden for the same reason as multi-field tuple structs.

## Sequences

Heterogeneous values. Serves `Vec<T>`, `&[T]`, tuples of any
length, and tuple variants carrying multiple values.

```nota
[1 2 3]
["hello" 42 true]
```

Empty sequence: `[]`.

## Maps

A sequence of `(key value)` pairs:

```nota
[("host" "localhost") ("port" 8080)]
```

Canonical form sorts entries by serialized key bytes.

## Reserved tokens

These tokens have meaning only in [nexus](https://github.com/LiGoldragon/nexus)
(the messaging superset) and are syntax errors in pure nota:

- Sigils `~` `@` `!` `?` `*` — verb sigils for messaging actions.
- Delimiter pairs `(| |)` `[| |]` `{ }` `{| |}` — pattern,
  atomic batch, shape, constrain.
- Single-character `=` between non-bind tokens. (In nexus, `=`
  appears only between two `@bind` names for aliasing.)
- `<` `>` `<=` `>=` `!=` — reserved for comparison operators in
  nexus pattern positions. (Comparison-operator design is
  deferred; the tokens are reserved.)

## Examples

### A record containing every literal form

```nota
(Sample
  true
  42
  3.14
  hello
  """
  multi
  line
  """
  #a1b2c3
  None
  Active
  [1 2 3]
  [("name" nota)]
  (Id 99))
```

The string `hello` is bare (ident-shaped); `"hello"` is the same
value.

### Comments

```nota
(Point
  ;; horizontal coordinate
  3.0
  ;; vertical coordinate
  4.0)
```

## Canonical form

For stable hashing (e.g. content-addressing):

- **Field order**: source-declaration order in the Rust type.
- **Map entries**: sorted by serialized key bytes.
- **Integers**: decimal, no separators, minimum digits.
- **Floats**: shortest round-trip; `.` always present.
- **Strings**: bare when content is a non-empty ASCII ident
  (non-reserved — not `true`, `false`, `None`); otherwise inline
  `"..."` unless content contains `"` or a newline; otherwise
  `""" """`.
- **Bytes**: lowercase hex, no separator, `#` prefix.
- **Whitespace**: single space within one expression; newline
  between top-level items; no indentation.

`to_string_pretty` is a separate output mode with indentation
rules.

## Forbidden constructs

These are *not* part of nota. Using them is a syntax error:

- Unit type (serde's `()`). Use a named variant (`Nil`, `None`)
  where absent-value semantics are needed.
- `null` keyword. Use `None` (PascalCase).
- Field-assignment `=` inside records. Records are positional.
- Multi-field unnamed structs (`struct Pair(i32, i32)`). Use
  named fields.
- `#[serde(flatten)]` on a struct field. Flattening requires
  map semantics (key-based field routing); positional records
  have no such routing. Prefer composition: keep the nested
  struct visible as a positional field.

## Implementation

[nota-serde](https://github.com/LiGoldragon/nota-serde) implements
`serde::Serializer` and `serde::Deserializer`. Any type
implementing serde's `Serialize` + `Deserialize` (with the
constraint above on unnamed multi-field structs) can round-trip
through nota text:

```rust
let value: MyType = nota_serde::from_str(text)?;
let text = nota_serde::to_string(&value)?;
```

## Repo layout

```
nota/
  README.md       ;; this file — grammar spec
  example.nota    ;; example records
  flake.nix       ;; dev-shell
  LICENSE.md
```
