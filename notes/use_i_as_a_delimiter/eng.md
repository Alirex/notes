# Use "i" as a delimiter

Why does it make sense to use "i" in the middle of names in classes, functions, modules, etc.?

In different name cases:

- PascalCase: `MyIClass`, `SomeBigNameIWithSeveralParts`
- snake_case: `my_i_function`, `some_big_name_i_with_several_parts`
- kebab-case: `my-env-i-variable`, `some-big-name-i-with-several-parts`

The letter "i" is visually distinct.
It looks like a simple line ("|"), making it unobtrusive yet noticeable.
It works in similar ways both in `PascalCase` and `snake_case`, making it versatile across naming conventions.
It is also compatible with file names, environment variables, and other contexts where names are used.

This can enhance the readability and maintainability of code,
especially in complex names where traditional delimiters
like underscores or hyphens might be less effective.

---

# Reasons to use

## General

**Visual separator**: gives a quick, minimal boundary inside long names. It's easy to spot when scanning code.

**Language-agnostic marker**: works across languages/contexts that differ in their allowed separators.

**Compactness**: separates semantic chunks without adding long words, keeping names shorter. Even if slightly longer
than no delimiter, it's often shorter than alternatives.

**Linter-friendly**: avoids issues with special characters that some linters or languages may flag.

**Preserves flat naming**: encodes hierarchical or composite structure where modules/namespaces aren’t available. Even
with inheritance in classes, you often want to reflect some base name in the derived class.

**Avoids ambiguous merging**: prevents two parts from running together and forming misleading words.

**Semantic clarity**: indicates distinct segments in compound names (e.g., role, phase, variant).

**Better readability**: improves scanning and comprehension of long identifiers by breaking them into meaningful parts.
It helps to clearly separate different parts of a name
without causing confusion with common words or abbreviations.

**Controlled extension point**: marks where new segments can be appended in stable naming schemes.

## Codegen

**Tokenization for codegen/tools**: provides a stable marker scripts/regex can split on when generating or parsing
names.

**Reduced key collisions**: consistent delimiter reduces accidental identical names when composing parts
programmatically. For example, with cookiecutter templates or codegen tools.

---

# Trade-offs and alternatives

## Costs

**Clash with the interface prefix**: in C#, TypeScript and Java, `IClass` means an interface, so `MyIClass` can read as
"My" + "IClass". This hits PascalCase, where a delimiter is needed most.

**"i" already has meanings**: it is the classic loop index and an English word. Without knowing the convention,
`user_i_profile` reads as "user, index, profile".

**Look-alike characters**: in many sans-serif fonts `I`, `l`, `1` and `|` are hard to tell apart (`MyIClass` vs
`MylClass`). Prefer a font that distinguishes them.

**Ambiguous splitting in PascalCase**: a single capital `I` cannot be told apart from words like `IO`, `IP`, `ID`
(`GetIOStream`). In snake_case and kebab-case (`_i_`, `-i-`) splitting is reliable.

**A private convention**: every new reader, and every AI agent, has to learn it. State it in the project's README or
style guide.

## Established alternatives

**Double underscore `__`**: Django lookups (`author__name`), nested env vars in pydantic-settings (`APP__DB__HOST`),
BEM in CSS (`block__element`). Widely recognized and easy to split.

**Double dash `--`** in kebab-case: BEM modifiers, file names (`report-export--v2`).

**Real structure**: modules, namespaces, nested classes or packages instead of encoding the hierarchy in a name.
The best option where the language allows it.

## When "i" is still a good choice

- One delimiter must work the same way in every case style (PascalCase, snake_case, kebab-case, env vars, file names),
  and `__` is not allowed or already taken in one of them.
- Generated names, where the same tool produces and parses them.
- Personal projects, or teams that agreed on the convention and wrote it down.

For shared code without such a need, prefer `__` and real namespaces.

---
