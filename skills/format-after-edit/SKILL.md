---
name: format-after-edit
description: After writing or editing source files, run the project's formatter (Biome or Prettier) instead of manually formatting code. Saves Claude tokens by delegating formatting to the toolchain.
---

# Format After Edit

After completing all writes and edits to source files in a task, run the project's formatter on those files. Do NOT manually reformat code — let the formatter handle it.

## When to Apply

After every `Edit` or `Write` tool call on a `.ts`, `.tsx`, `.js`, `.jsx`, `.json`, `.css`, `.html` file. Run once at the end of a task covering all touched files, not after each individual edit.

## Detection & Commands

Detect the formatter by looking for config files in the project root, then run the appropriate command.

### 1. Biome (check for `biome.json`)

```bash
# Single file
./node_modules/.bin/biome format --write <file>

# Multiple files
./node_modules/.bin/biome format --write <file1> <file2> ...

# If no local binary
npx @biomejs/biome format --write <file>
```

### 2. Prettier (check for `.prettierrc`, `.prettierrc.json`, `.prettierrc.js`, `.prettierrc.cjs`, `prettier.config.js`, `prettier.config.cjs`)

```bash
# Single or multiple files
./node_modules/.bin/prettier --write <file>

# If no local binary
npx prettier --write <file>
```

### 3. Package.json `format` script (check if `scripts.format` exists)

```bash
yarn format <file>
# or
npm run format -- <file>
```

Prefer local binaries (`./node_modules/.bin/`) over `npx` to avoid network calls.

## Priority Order

1. `biome.json` exists → use Biome
2. `.prettierrc*` or `prettier.config.*` exists → use Prettier
3. `package.json` has a `format` script → use it
4. None found → skip formatting, note it to the user

## Example Workflow

```
# After editing src/components/Foo.tsx and src/utils/bar.ts:
./node_modules/.bin/biome format --write src/components/Foo.tsx src/utils/bar.ts
```

## Rules

- Run formatter AFTER all edits are complete, not between edits.
- Do NOT reformat files you did not touch.
- If the formatter exits with an error, show the output to the user but do not retry by hand-editing.
- Skip binary files, Swift/Kotlin/Java/Objective-C files — only format JS/TS/JSON/CSS/HTML.
- This replaces any manual indentation, quote normalization, or import sorting Claude would otherwise do.
