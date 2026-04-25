# ARCHITECTURE — nota

The text grammar spec at the bottom of the language stack. nota
is the **family of grammars**: a tokeniser shape (delimiters,
identifiers, literals, sigils) and a parsing discipline that
nexus and other dialects extend.

This repo is **spec-only**. The grammar lives in
[README.md](README.md); an example file is
[`example.nota`](example.nota). No Rust code.

## Role in the sema-ecosystem

Layer 0 of the project (per [mentci-next/docs/architecture.md
§8](https://github.com/LiGoldragon/mentci-next/blob/main/docs/architecture.md)).
The kernel that
[nota-serde-core](https://github.com/LiGoldragon/nota-serde-core)
implements; the foundation that
[nexus](https://github.com/LiGoldragon/nexus)'s grammar refines.

## What this repo defines

- The token shape (Pascal-named records, `:keyword` syntax,
  `@bind` syntax, literal forms, sigil reservation).
- The parsing discipline (delimiter balancing, comment forms,
  whitespace handling).
- The dialect-extensibility points where downstream grammars
  layer additional rules.

## What this repo does not define

- The delimiter-family matrix (lives in
  [nexus](https://github.com/LiGoldragon/nexus)).
- Edit / query semantics (live in nexus + criomed).
- Records (live in [nexus-schema](https://github.com/LiGoldragon/nexus-schema)).

## Status

**Spec, stable.** Changes are coordinated with
[nota-serde-core](https://github.com/LiGoldragon/nota-serde-core)
to keep the parser kernel in sync.
