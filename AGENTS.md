# Agent instructions — nota

You **MUST** read AGENTS.md at `github:ligoldragon/lore` — the workspace contract.

## Repo role

Spec-only repo. The text grammar at the bottom of the language stack — the **family of grammars** (tokeniser shape + parsing discipline) that nexus and other dialects extend.

The grammar lives in `README.md`; an example file is `example.nota`.

nota-codec implements decode/encode against this spec; nota-derive provides the proc-macro derives.

---

## Carve-outs worth knowing

- Spec-only — no Rust source bodies in this repo.
- Grammar changes are coordinated with `nota-codec` to keep the parser kernel in sync.
