---
name: shared-types-in-types-folder
description: Enforces that TypeScript types used across more than one component live in a dedicated types module, never exported from a component file. Triggers whenever you are about to write `export type` / `export interface` in a `.tsx`/component file, or when an existing component-file-exported type is imported by another component.
---

# Shared Types Belong in a Types Folder

Component files should export components, not types that other modules consume. If a `type` or `interface` is used in more than one file, it must live in a dedicated types module and be imported from there. Component files may still declare types that are purely local (used only inside that one file).

## When to Apply

Trigger this rule in any of these situations:

1. You are about to write `export type Foo = ...` or `export interface Foo { ... }` inside a component file (`*.tsx`, or a `.ts` file in a `components/` directory).
2. You are about to import a type from a component file in another component or module — `import type { Foo } from "../components/foo"`.
3. You notice during a refactor that a type previously declared inside a component is now needed by a second caller.
4. A code review flags a type as exported from the wrong place.

## The Rule

- **Local type, single file** → declare it in the component file, do **not** export it. Keep it as `type Foo = ...` (no `export`).
- **Type used in 2+ files** → move it to a types module:
  - Look for an existing types folder, in this order of preference:
    1. The current module's types folder: `src/modules/<module>/types/index.ts`
    2. A sibling `types/` folder one level up
    3. A repo-wide `src/types/` folder
  - If none exists, create `src/types/index.ts` (repo root) — but prefer a module-local folder whenever the type is clearly scoped to one module.
- **Never** add `export` to a type in a component file just to share it. Move the type instead.

## How to Apply

When you detect a violation (you are about to write a cross-file `export type` in a component, or you spot one already there):

1. Identify the correct types module using the priority list above.
2. Move the `type` / `interface` declaration into that module. Preserve any JSDoc.
3. Update every importer to import from the types module.
4. Remove the now-unused `export` (and the declaration itself) from the component file.
5. If the type was only used in one file after all, delete the `export` keyword — don't move it.

## Examples

### Violation

```tsx
// src/modules/journey/components/questions/form-fields.tsx
export type FormFieldsNav = {  // ← used by chapter-shell.tsx too
  onContinue: () => void;
  onBack: () => void;
};

export function FormFieldsQuestion({ nav }: { nav: FormFieldsNav }) { ... }
```

```tsx
// src/modules/journey/components/chapter-shell.tsx
import { FormFieldsQuestion, type FormFieldsNav } from "./questions/form-fields";
```

### Fixed

```ts
// src/modules/journey/types/index.ts
export type FormFieldsNav = {
  onContinue: () => void;
  onBack: () => void;
};
```

```tsx
// src/modules/journey/components/questions/form-fields.tsx
import type { FormFieldsNav } from "@/modules/journey/types";

export function FormFieldsQuestion({ nav }: { nav: FormFieldsNav }) { ... }
```

```tsx
// src/modules/journey/components/chapter-shell.tsx
import type { FormFieldsNav } from "@/modules/journey/types";
import { FormFieldsQuestion } from "./questions/form-fields";
```

### Not a violation (single-file use)

```tsx
// src/modules/journey/components/chapter-shell.tsx
type QuestionInputProps = {  // ← no export, used only in this file
  question: Question;
  answer: AnswerValue;
};
```

## Rationale

- Component files export UI; mixing type exports creates accidental coupling and makes it hard to track where shared shapes live.
- A single canonical location for shared types makes them discoverable and easy to refactor.
- Importing types from a `types/` module instead of a sibling component also avoids accidental runtime-import cycles when a component happens to need the type.

## What NOT to Do

- Do not create a new `types/` folder when an appropriate one already exists at a higher level — reuse the closest one.
- Do not move purely local types into the types module just for consistency. The rule applies to types used in 2+ files.
- Do not re-export the type from the component file as a "convenience" — importers should reach for the types module directly.
