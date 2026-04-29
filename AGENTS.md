# Agent Bootstrap — nota

Spec-only repo. The text grammar at the bottom of the language
stack — the **family of grammars** (tokeniser shape + parsing
discipline) that nexus and other dialects extend.

The grammar lives in [README.md](README.md); an example file is
[`example.nota`](example.nota). [ARCHITECTURE.md](ARCHITECTURE.md)
describes nota's role in the workspace.

[nota-codec](https://github.com/LiGoldragon/nota-codec) implements
decode/encode against this spec; [nota-derive](https://github.com/LiGoldragon/nota-derive)
provides the proc-macro derives (`NotaRecord`, `NotaEnum`,
`NotaTransparent`, `NotaTryTransparent`, `NexusPattern`,
`NexusVerb`).

For project-wide rules: [mentci/AGENTS.md](https://github.com/LiGoldragon/mentci/blob/main/AGENTS.md).

For project-wide architecture: [criome/ARCHITECTURE.md](https://github.com/LiGoldragon/criome/blob/main/ARCHITECTURE.md).

## Process

- Jujutsu only (`jj`).
- Push immediately after every change.
- Spec-only — no Rust source bodies in this repo.
- Grammar changes are coordinated with `nota-codec` to keep the
  parser kernel in sync.
