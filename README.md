# nota

A text data format. Four delimiter pairs, two sigils, no
keywords. Records are positional; field names live in the Rust
schema, not in the text. The Rust implementation is
[nota-serde](https://github.com/LiGoldragon/nota-serde).

`nota` is the data-layer format in the sema ecosystem.
[nexus](https://github.com/LiGoldragon/nexus) is the superset
messaging protocol — every valid nota text is also valid nexus;
nexus adds sigils, delimiters, and the `=` token for pattern /
mutate / bind / negate / alias actions.

This repo is spec-only. Grammar and examples below.

---

## Delimiters

Four pairs. Role is fixed at the opening token — the grammar is
first-token-decidable at every choice point. No interior scanning.

| Pair | Role | Example |
|---|---|---|
| `( )` | Record (named composite value) | `(Point 3.0 4.0)` |
| `[ ]` | String (inline) | `[hello world]` |
| `[\| \|]` | String (multiline, auto-dedented) | `[\| line one / line two \|]` |
| `< >` | Sequence (heterogeneous) | `<1 2 3>`, `<([k] v) ([k2] v2)>` |

Strings are delimiter-bounded, not quote-bounded. No `"..."`.

## Sigils

Two. Each has a fixed lexical position — no interior scanning.

| Sigil | Role | Position |
|---|---|---|
| `;;` | Line comment | to end of line |
| `#` | Byte-literal prefix | `#a1b2c3` (3 bytes) |

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

## Literals

| Form | Type |
|---|---|
| `42`, `-7`, `0xFF`, `0b1010`, `0o755`, `1_000_000` | Integer |
| `3.14`, `-0.5` | Float (always `.` in canonical form) |
| `[text]`, `[\| multiline \|]`, or a bare identifier | String |
| `true`, `false` | Bool |
| `#<even-hex>` (e.g. `#a1b2c3`) | Bytes |
| `#<64 hex chars>` | Blake3 hash (canonical length) |

Hex bytes are lowercase and even-length. No quoted-string form.
No typed numeric suffixes (`42u32`) — the receiving type
determines the value type.

## Bare-identifier strings

Where a string is expected by the schema, a bare identifier
may be written in place of `[identifier]`. This keeps config
files readable for the common case where string values follow
identifier rules:

```nota
;; these three forms are equivalent when the schema says String
(Package [nota-serde])
(Package nota-serde)
(Package [|nota-serde|])
```

A bare identifier qualifies when its content is a non-empty
ident-class token (PascalCase / camelCase / kebab-case) and is
*not* one of the reserved keywords `true`, `false`, `None` —
those always mean the bool / Option::None they name.

Bare form is **ASCII-only**. Non-ASCII content (e.g. `café`,
emoji, RTL text) always emits through `[ ]` or `[| |]`. The
path separator `:` never appears in a bare string — use `[...]`
for content containing colons.

Single-character strings and `char` values follow the same rule:
`'a'` emits as bare `a`; both bare and bracketed forms deserialize
back to `char`.

**Canonical form emits bare when eligible.** Serialising the
string `"nota-serde"` through `to_string` produces `nota-serde`,
not `[nota-serde]`. Strings containing spaces, `]`, newlines,
digits as the first char, or reserved-word content always round-
trip through the `[ ]` / `[| |]` forms.

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
and serialize *wrapped*, with one positional value:

```nota
(Id 42)
```

Not transparently — `(Id 42)` is the canonical form, not bare
`42`. This preserves structural integrity across the wire.

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
length, and tuple variants carrying multiple values. (The latter
also appears inside records.)

```nota
<1 2 3>
<[hello] 42 true>
```

## Maps

A sequence of `(key value)` pairs:

```nota
<([host] [localhost]) ([port] 8080)>
```

Canonical form sorts entries by serialized key bytes. (Note:
"sorted by bytes" is deterministic but not arithmetic —
`(IntKey 10)` sorts before `(IntKey 2)` because `"1"` < `"2"`
lexicographically. Fine for string keys; use with care for
struct keys.)

## Examples

### A record containing every literal form

```nota
(Sample
  true
  42
  3.14
  hello
  [\|
    multi
    line
  \|]
  #a1b2c3
  None
  Active
  <1 2 3>
  <(name nota)>
  (Id 99))
```

The string `hello` is bare (ident-shaped); `[hello]` is the same
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
  `[...]` unless content contains `]` or a newline; otherwise
  `[\| \|]`.
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
- Quoted strings (`"..."`, `'...'`). Use `[ ]` / `[\| \|]`.
- Field-assignment `=` inside records. Records are positional.
  The `=` token itself is reserved and appears only in nexus
  (for bind aliasing).
- Sigils `~`, `@`, `!` — reserved for the
  [nexus](https://github.com/LiGoldragon/nexus) messaging layer,
  not valid in pure nota.
- Delimiter pairs `(\| \|)`, `{ }`, `{\| \|}` — also nexus-only.
- Multi-field unnamed structs (`struct Pair(i32, i32)`). Use
  named fields.
- `#[serde(flatten)]` on a struct field. Flattening requires
  map semantics (key-based field routing); positional records
  have no such routing. A flattened field currently serialises
  through the map path and emits `<(k v) …>` instead of a
  `(Name …)` record — not what you want. Prefer composition:
  keep the nested struct visible as a positional field.

## Canonical-form assumptions

Canonical form assumes `Serialize` is injective on distinct
values — that two different values never produce identical
serialised bytes. Derived `Serialize` is injective over the
visible fields by construction. A hand-rolled `Serialize` that
drops or lossily encodes data breaks this assumption, which in
turn makes map-key sort order undefined between keys that
collide.

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
  example.nota   ;; example records
  flake.nix       ;; dev-shell
  LICENSE.md
```
