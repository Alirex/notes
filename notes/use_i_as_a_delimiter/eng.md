# Use "i" as a delimiter

Why does it make sense to use "i" in the middle of names in classes, functions, modules, etc.?

In different name cases:

- CamelCase: `MyIClass`, `SomeBigNameIWithSeveralParts`
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
with
inheritance in classes, you often want to reflect some base name in the derived class.

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
