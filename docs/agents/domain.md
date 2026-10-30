# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

## Before exploring, read these

- **`docs/internals/glossary.md`**: the single glossary for T3 Code. Also respect the
  glossary in `AGENTS.md` ("A small glossary") for how to address people.
- **`docs/internals/*.md`**: architectural decisions and their reasons. Start from
  `docs/internals/overview.md`, then read the pages that touch the area you're working in.

There is no root `GLOSSARY.md` or `docs/adr/`. Don't create them. When `/domain-modeling`
resolves a term, it updates `docs/internals/glossary.md`. When it records a decision, it
rewrites the relevant `docs/internals/` page, following the Documentation rules in `AGENTS.md`.

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in `docs/internals/glossary.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal: either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag decision conflicts

If your output contradicts a decision recorded in `docs/internals/`, surface it explicitly
rather than silently overriding:

> _Contradicts docs/internals/providers.md (capabilities, not driver kind), but worth reopening because…_
