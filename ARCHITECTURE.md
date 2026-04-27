# ARCHITECTURE — nota

The text grammar spec at the bottom of the language stack. nota
is the **family of grammars**: a tokeniser shape (delimiters,
identifiers, literals, sigils) and a parsing discipline that
nexus and other dialects extend.

This repo is **spec-only**. The grammar lives in
[README.md](README.md); an example file is
[`example.nota`](example.nota). No Rust code.

## Role in the sema-ecosystem

Layer 0 of the project (per [criome/ARCHITECTURE.md
§8](https://github.com/LiGoldragon/criome/blob/main/ARCHITECTURE.md)).
The kernel that [nota-codec](https://github.com/LiGoldragon/nota-codec)
+ [nota-derive](https://github.com/LiGoldragon/nota-derive)
implement; the foundation that
[nexus](https://github.com/LiGoldragon/nexus)'s grammar refines.

## What this repo defines

- The token shape (Pascal-named records, positional fields, `:`
  as the path separator (`Char:Upper:A`), literal forms, the
  reserved-sigil set that downstream dialects can claim).
- The parsing discipline (delimiter balancing, comment forms,
  whitespace handling).
- The dialect-extensibility points where downstream grammars
  layer additional rules.

## What this repo does not define

- The full delimiter-family matrix (lives in
  [nexus](https://github.com/LiGoldragon/nexus)).
- Edit / query semantics (live in nexus + criome).
- Record kinds (live in [signal](https://github.com/LiGoldragon/signal)).

## Status

**Spec, stable.** Changes are coordinated with
[nota-codec](https://github.com/LiGoldragon/nota-codec) to keep
the parser kernel in sync.
