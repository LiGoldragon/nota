# nota

A text data format. Four delimiter pairs, two sigils, no
keywords. Serializes typed structured values. The Rust
implementation is
[nota-serde](https://github.com/LiGoldragon/nota-serde).

`nota` is the data-layer format in the sema ecosystem.
[nexus](https://github.com/LiGoldragon/nexus) is the superset
messaging protocol — every valid nota text is also valid nexus;
nexus adds sigils and delimiter pairs for query / mutate / bind
/ negate actions.

This repo is spec-only. Grammar and examples below.

---

## Delimiters

Four pairs. Role is fixed at the opening token — the grammar is
first-token-decidable at every choice point. No interior scanning.

| Pair | Role | Example |
|---|---|---|
| `( )` | Record (named composite value) | `(Point horizontal=3.0 vertical=4.0)` |
| `[ ]` | String (inline) | `[hello world]` |
| `[\| \|]` | String (multiline, auto-dedented) | `[\| line one / line two \|]` |
| `< >` | Sequence / tuple (heterogeneous) | `<1 2 3>`, `<(k v) (k2 v2)>` |

Strings are delimiter-bounded, not quote-bounded. No `"..."`.

## Sigils

Two. Each has a fixed lexical position — no interior scanning.

| Sigil | Role | Position |
|---|---|---|
| `;;` | Line comment | to end of line |
| `#` | Byte-literal prefix | `#a1b2c3` (3 bytes) |

## Identifiers

Three lexer classes, distinguished by first character:

| Class | Shape | Role |
|---|---|---|
| PascalCase | First char uppercase | Type / variant names (structural) |
| camelCase | First char lowercase, no hyphens | Field names, instance names |
| kebab-case | Lowercase, with `-` | Titles, hash-IDs, tags |

The parser dispatches on class. Classes are disjoint — no
identifier is valid in more than one class.

## Literals

| Form | Type |
|---|---|
| `42`, `-7`, `0xFF`, `0b1010`, `0o755`, `1_000_000` | Integer |
| `3.14`, `-0.5` | Float (always `.` in canonical form) |
| `[text]`, `[\| multiline \|]` | String |
| `true`, `false` | Bool |
| `#<even-hex>` (e.g. `#a1b2c3`) | Bytes |
| `#<64 hex chars>` | Blake3 hash (canonical length) |

Hex bytes are lowercase and even-length. No quoted-string form.
No typed numeric suffixes (`42u32`) — the receiving type
determines the value type.

## Path syntax

`:` separates nested names:

```
Char:Upper:A
Schema:Type:Field
example:item
```

## Records

A record is a named composite with positional name followed by
named fields:

```nota
(TypeName field1=value1 field2=value2)
```

Named fields use `=`. Positional variants (tuple structs, tuple
variants, enum variants carrying unnamed data) use whitespace:

```nota
(TupleVariant value1 value2)
```

## Sequences

Heterogeneous allowed. Serves both `Vec<T>` and tuples:

```nota
<1 2 3>
<[hello] 42 true>
```

## Maps

A sequence of `(key value)` pairs:

```nota
<([host] [localhost]) ([port] 8080)>
```

Canonical form sorts entries by serialized key bytes.

## Examples

### A record with a nested record field

```nota
(Line
  start=(Point horizontal=0.0 vertical=0.0)
  end=(Point horizontal=10.0 vertical=10.0))
```

### An enum variant

```nota
Fire
```

(bare PascalCase name — a unit variant in the receiving type)

### Strings

```nota
(Greeting message=[hello, world])

(Document body=[\|
  first line
  second line
\|])
```

### Bytes

```nota
(Frame data=#a1b2c3d4)
```

### Blake3 hash

```nota
(Ref target=#e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855)
```

### Comments

```nota
(Point
  ;; horizontal coordinate
  horizontal=3.0
  vertical=4.0)
```

## Canonical form

For stable hashing (e.g. content-addressing):

- **Field order**: source-declaration order in the Rust type.
- **Map entries**: sorted by serialized key bytes.
- **Integers**: decimal, no separators, minimum digits.
- **Floats**: shortest round-trip; `.` always present.
- **Strings**: inline `[...]` unless content contains `]` or a
  newline, otherwise `[\| \|]`.
- **Bytes**: lowercase hex, no separator, `#` prefix.
- **Whitespace**: single space within one expression; newline
  between top-level items; no indentation.

`to_string_pretty` is a separate output mode with indentation
rules.

## Forbidden constructs

These are explicitly *not* part of nota. Using them is a syntax
error:

- Unit type (serde's `()`). Use a named variant (`Nil`, `None`)
  where absent-value semantics are needed.
- `null` keyword. Use `None` (PascalCase).
- Quoted strings (`"..."`, `'...'`). Use `[ ]` / `[\| \|]`.
- Sigils `~`, `@`, `!` — reserved for the [nexus](https://github.com/LiGoldragon/nexus)
  messaging layer, not valid in pure nota.
- Delimiter pairs `(\| \|)`, `{ }`, `{\| \|}` — also nexus-only.

## Implementation

[nota-serde](https://github.com/LiGoldragon/nota-serde) implements
`serde::Serializer` and `serde::Deserializer`. Any type
implementing serde's `Serialize` + `Deserialize` can round-trip
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
