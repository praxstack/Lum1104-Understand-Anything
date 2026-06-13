# Dart Language Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land deep Dart support in `@understand-anything/core` at parity with the recently merged Kotlin extractor (PR #347), producing structural graph + call-graph edges for `.dart` files.

**Architecture:** Vendor a freshly-built `tree-sitter-dart.wasm` as a workspace-internal package (`@understand-anything/tree-sitter-dart-wasm`); register a `dartConfig` `LanguageConfig` referencing it; add a `DartExtractor` class implementing `LanguageExtractor`; cover with ~22 vitest cases driven by the real WASM grammar. No changes to shared schemas, registries, or the dashboard.

**Tech Stack:** TypeScript 5 (strict), pnpm 10 workspaces, vitest 3, `web-tree-sitter@^0.26.6`, `tree-sitter-cli@0.26.x` (build-time only).

---

## File structure

| File | Responsibility |
|---|---|
| `understand-anything-plugin/pnpm-workspace.yaml` | Add `packages/tree-sitter-dart-wasm/*` so pnpm sees the new package |
| `.../packages/tree-sitter-dart-wasm/package.json` | Minimal package metadata — name, version, main pointing at the wasm |
| `.../packages/tree-sitter-dart-wasm/tree-sitter-dart.wasm` | Vendored wasm binary (built from `tree-sitter-dart@1.0.0` grammar.js) |
| `.../packages/tree-sitter-dart-wasm/BUILD.md` | Provenance + rebuild instructions for future maintainers |
| `.../packages/core/package.json` | Add `@understand-anything/tree-sitter-dart-wasm: workspace:*` to dependencies |
| `.../packages/core/src/languages/configs/dart.ts` | Single `LanguageConfig` object for Dart |
| `.../packages/core/src/languages/configs/index.ts` | Import + register `dartConfig` |
| `.../packages/core/src/plugins/extractors/dart-extractor.ts` | `DartExtractor` class — structural + call-graph extraction |
| `.../packages/core/src/plugins/extractors/index.ts` | Import + register `DartExtractor` |
| `.../packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts` | ~22 vitest cases — real WASM parse + assertions |

---

## Working directory & branch assumption

All paths in this plan are relative to the repository root `/Users/thejesh/Git/Understand-Anything`. The implementation branch `feat/dart-language-support` already exists (the spec was committed to it in commits `2bb5233` and `c447b69`).

Verify before starting:

```bash
cd /Users/thejesh/Git/Understand-Anything
git status                              # clean
git branch --show-current               # feat/dart-language-support
git log --oneline -3                    # top: c447b69 docs: revise Dart spec...
```

---

## Task 1: Vendor the freshly-built tree-sitter-dart wasm

**Why first:** Every downstream task depends on this wasm loading correctly via `require.resolve("@understand-anything/tree-sitter-dart-wasm/tree-sitter-dart.wasm")`. Build + commit it before writing the config so dependent tasks can run their tests.

**Files:**
- Create: `understand-anything-plugin/packages/tree-sitter-dart-wasm/package.json`
- Create: `understand-anything-plugin/packages/tree-sitter-dart-wasm/tree-sitter-dart.wasm` (binary, ~745 KB)
- Create: `understand-anything-plugin/packages/tree-sitter-dart-wasm/BUILD.md`
- Modify: `understand-anything-plugin/pnpm-workspace.yaml`

- [ ] **Step 1: Inspect existing workspace config**

Run:
```bash
cat understand-anything-plugin/pnpm-workspace.yaml
```

Expected output:
```yaml
packages:
  - packages/*
  - src
```

Confirm `packages/*` is already a glob — that means our new `packages/tree-sitter-dart-wasm/` will be picked up automatically and no edit is required. If the file does NOT use a glob, add a line `  - packages/tree-sitter-dart-wasm`.

- [ ] **Step 2: Build the wasm from upstream grammar source**

Prerequisites — install once if absent:
```bash
npm install -g tree-sitter-cli@latest
tree-sitter --version    # expect: tree-sitter 0.26.x or newer
```

Build:
```bash
cd /tmp && rm -rf dart-build && mkdir dart-build && cd dart-build
npm pack tree-sitter-dart@1.0.0     # downloads the upstream tarball
tar xzf tree-sitter-dart-1.0.0.tgz
cd package
tree-sitter build --wasm            # ~30 s; downloads wasi-sdk-29 on first run
ls -la tree-sitter-dart.wasm        # expect: ~745 KB
head -c 30 tree-sitter-dart.wasm | xxd | head -1
```

Expected last line:
```
00000000: 0061 736d 0100 0000 0011 0864 796c 696e  .asm.......dylin
```

(The `\0asm` magic followed by a custom section named `dylink.0` — the byte after `dylink` must be `2e 30`, NOT a length byte for an old-format `dylink` section.)

If the byte after `dylink` is `c8 9b 2c` (the broken upstream wasm), the build did NOT regenerate — verify your `tree-sitter --version` is current.

- [ ] **Step 3: Vendor the wasm into the workspace package**

```bash
cd /Users/thejesh/Git/Understand-Anything
mkdir -p understand-anything-plugin/packages/tree-sitter-dart-wasm
cp /tmp/dart-build/package/tree-sitter-dart.wasm \
   understand-anything-plugin/packages/tree-sitter-dart-wasm/tree-sitter-dart.wasm
ls -la understand-anything-plugin/packages/tree-sitter-dart-wasm/
```

Expected: the wasm file is present, ~745 KB.

- [ ] **Step 4: Write the package metadata**

Create `understand-anything-plugin/packages/tree-sitter-dart-wasm/package.json`:

```json
{
  "name": "@understand-anything/tree-sitter-dart-wasm",
  "version": "0.1.0",
  "description": "Vendored tree-sitter-dart WASM grammar built with the modern dylink.0 ABI for use with web-tree-sitter@^0.26.",
  "main": "tree-sitter-dart.wasm",
  "files": ["tree-sitter-dart.wasm", "BUILD.md"],
  "license": "MIT"
}
```

- [ ] **Step 5: Write the BUILD provenance note**

Create `understand-anything-plugin/packages/tree-sitter-dart-wasm/BUILD.md`:

```markdown
# tree-sitter-dart WASM (vendored)

This directory ships a pre-built `tree-sitter-dart.wasm` because the upstream
npm release does not.

## Why vendored

The published `tree-sitter-dart@1.0.0` (2023-02-24) tarball does include a
`tree-sitter-dart.wasm`, but it was built with a pre-`dylink.0` tree-sitter
CLI. `web-tree-sitter@0.26.x` — the loader this project uses — expects the
newer `dylink.0` custom-section name and refuses to load the older format
(failure surfaces in `getDylinkMetadata`).

Rebuilding the same upstream grammar.js with a current
`tree-sitter-cli@0.26.x` produces a `dylink.0` wasm that loads cleanly.

## How to rebuild

```bash
npm install -g tree-sitter-cli@latest
cd /tmp && npm pack tree-sitter-dart@1.0.0
tar xzf tree-sitter-dart-1.0.0.tgz
cd package
tree-sitter build --wasm
cp tree-sitter-dart.wasm \
   /path/to/understand-anything-plugin/packages/tree-sitter-dart-wasm/
```

Verify the resulting wasm:

```bash
head -c 30 tree-sitter-dart.wasm | xxd | head -1
# Expect: ...dylin / k.0...
```

## Provenance

- Grammar source: `tree-sitter-dart@1.0.0` (publisher: amaanq) — `grammar.js`
  unchanged, only the wasm artifact is regenerated.
- Built with: `tree-sitter-cli@0.26.x`, `wasi-sdk-29-arm64-macos`.

## When to remove this package

If amaanq publishes a refreshed `tree-sitter-dart` with a `dylink.0` wasm,
this workspace package can be deleted and the dependency in
`@understand-anything/core` flipped to the upstream package.
```

- [ ] **Step 6: Run pnpm install to wire the workspace package**

```bash
cd understand-anything-plugin
pnpm install 2>&1 | tail -5
```

Expected: `Done in <Ns>` with no errors mentioning the new package. The package is now resolvable via `require.resolve("@understand-anything/tree-sitter-dart-wasm/tree-sitter-dart.wasm")` from any workspace member that depends on it (which we'll wire in Task 3).

- [ ] **Step 7: Commit**

```bash
cd /Users/thejesh/Git/Understand-Anything
git add understand-anything-plugin/packages/tree-sitter-dart-wasm/ \
        understand-anything-plugin/pnpm-lock.yaml \
        understand-anything-plugin/pnpm-workspace.yaml
git commit -m "$(cat <<'EOF'
feat(tree-sitter-dart-wasm): vendor freshly-built dart WASM grammar

The upstream tree-sitter-dart@1.0.0 ships a pre-`dylink.0` wasm that
fails to load in web-tree-sitter@0.26.x. The grammar source itself is
sound — rebuilding with the current tree-sitter-cli + wasi-sdk produces
a working dylink.0 wasm. Vendor that artifact as a workspace-internal
package so @understand-anything/core can depend on it via workspace:*.

BUILD.md documents the provenance and rebuild instructions.
EOF
)"
```

---

## Task 2: Add the dartConfig LanguageConfig

**Files:**
- Create: `understand-anything-plugin/packages/core/src/languages/configs/dart.ts`
- Modify: `understand-anything-plugin/packages/core/src/languages/configs/index.ts`
- Modify: `understand-anything-plugin/packages/core/package.json` (add workspace dep)

- [ ] **Step 1: Add the workspace dependency to core**

Edit `understand-anything-plugin/packages/core/package.json` — add **one line** inside the `"dependencies"` block, in alphabetical position (before `fuse.js`):

```json
"dependencies": {
  "@understand-anything/tree-sitter-dart-wasm": "workspace:*",
  "@tree-sitter-grammars/tree-sitter-kotlin": "1.1.0",
  ...
}
```

Note: pnpm's workspace protocol uses `workspace:*` — same as how core would reference any other internal package.

Run:
```bash
cd understand-anything-plugin
pnpm install 2>&1 | tail -3
```

Expected: clean install, no warnings mentioning the workspace package.

- [ ] **Step 2: Create the dart.ts config file**

Create `understand-anything-plugin/packages/core/src/languages/configs/dart.ts`:

```ts
import type { LanguageConfig } from "../types.js";

export const dartConfig = {
  id: "dart",
  displayName: "Dart",
  extensions: [".dart"],
  treeSitter: {
    wasmPackage: "@understand-anything/tree-sitter-dart-wasm",
    wasmFile: "tree-sitter-dart.wasm",
  },
  concepts: [
    "null safety",
    "mixins",
    "extensions",
    "isolates",
    "async/await",
    "streams",
    "factory constructors",
    "named constructors",
    "records",
    "sealed classes",
  ],
  filePatterns: {
    entryPoints: ["lib/main.dart", "bin/*.dart"],
    barrels: ["lib/*.dart"],
    tests: ["test/**/*_test.dart"],
    config: ["pubspec.yaml", "analysis_options.yaml"],
  },
} satisfies LanguageConfig;
```

- [ ] **Step 3: Register dartConfig in the configs index**

Edit `understand-anything-plugin/packages/core/src/languages/configs/index.ts`. Three places to edit:

(a) Add the import alongside the other code-language imports (alphabetical-ish, between `cppConfig` and `csharpConfig` is fine):

```ts
import { dartConfig } from "./dart.js";
```

(b) Add `dartConfig` to the `builtinLanguageConfigs` array, inside the "Code languages" block (place between `cppConfig` and `csharpConfig`):

```ts
  // Code languages
  typescriptConfig,
  javascriptConfig,
  pythonConfig,
  goConfig,
  rustConfig,
  javaConfig,
  rubyConfig,
  phpConfig,
  swiftConfig,
  kotlinConfig,
  luaConfig,
  cConfig,
  cppConfig,
  dartConfig,
  csharpConfig,
```

(c) Add `dartConfig` to the named re-export block in the same position.

- [ ] **Step 4: Build core to verify TypeScript compiles**

```bash
cd understand-anything-plugin
pnpm --filter @understand-anything/core build 2>&1 | tail -5
```

Expected: `Done` with no tsc errors.

- [ ] **Step 5: Write a smoke test that the config is registered and the grammar loads**

This test is a sanity check — it doesn't exercise the extractor (Task 3 onwards does that). Append it to the existing test file
`understand-anything-plugin/packages/core/src/languages/__tests__/language-registry.test.ts` (look at the existing tests there for style; if no test file exists, the build step's import of `dartConfig` is enough sanity for this task).

Run:
```bash
pnpm --filter @understand-anything/core test 2>&1 | tail -10
```

Expected: all existing tests still pass. No regressions.

- [ ] **Step 6: Verify the wasm actually loads via the existing TreeSitterPlugin**

Write a one-off Node script at `/tmp/verify-dart-wasm.mjs`:

```js
import { createRequire } from "node:module";
import * as ts from "web-tree-sitter";

const require = createRequire(import.meta.url);
await ts.Parser.init();
const wasmPath = require.resolve(
  "@understand-anything/tree-sitter-dart-wasm/tree-sitter-dart.wasm",
);
const Lang = await ts.Language.load(wasmPath);
const p = new ts.Parser();
p.setLanguage(Lang);
const tree = p.parse("void main() { print('hi'); }");
console.log("rootType:", tree.rootNode.type);
console.log("firstChild:", tree.rootNode.namedChild(0)?.type);
```

Run from inside core:
```bash
cd understand-anything-plugin/packages/core
cp /tmp/verify-dart-wasm.mjs ./verify-dart-wasm.mjs
node verify-dart-wasm.mjs
rm verify-dart-wasm.mjs
```

Expected output:
```
rootType: program
firstChild: function_signature
```

If you instead see an `Error: ... at getDylinkMetadata`, the wasm is the wrong ABI — go back to Task 1, Step 2 and verify the build produced a `dylink.0` artifact.

- [ ] **Step 7: Commit**

```bash
git add understand-anything-plugin/packages/core/package.json \
        understand-anything-plugin/packages/core/src/languages/configs/dart.ts \
        understand-anything-plugin/packages/core/src/languages/configs/index.ts \
        understand-anything-plugin/pnpm-lock.yaml
git commit -m "$(cat <<'EOF'
feat(core): register dart LanguageConfig

Adds the Dart language config and wires it into builtinLanguageConfigs
so .dart files are recognized by the language registry. References the
vendored @understand-anything/tree-sitter-dart-wasm package for grammar
loading.

No extractor yet — structural extraction lands in the next commit.
EOF
)"
```

---

## Task 3: Scaffold DartExtractor + register it

**Why before TDD steps:** Subsequent TDD tasks need an importable `DartExtractor` class to add tests against. This task creates the empty shell + registration; the next tasks fill in extraction logic test-first.

**Files:**
- Create: `understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts`
- Modify: `understand-anything-plugin/packages/core/src/plugins/extractors/index.ts`

- [ ] **Step 1: Create the skeleton DartExtractor**

Create `understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts`:

```ts
import type { StructuralAnalysis, CallGraphEntry } from "../../types.js";
import type { LanguageExtractor, TreeSitterNode } from "./types.js";
import { findChild, findChildren } from "./base-extractor.js";

/**
 * Whether a Dart name is exported.
 *
 * Dart's visibility rule is name-based and the INVERSE of Kotlin's: names
 * starting with `_` are library-private, everything else is exported. There
 * is no `public` / `private` keyword to inspect — only the leading character.
 */
function isExported(name: string): boolean {
  return !name.startsWith("_");
}

/**
 * Dart extractor for tree-sitter structural analysis + call graph.
 *
 * Approach (matching `KotlinExtractor` convention): mixin / extension / enum
 * declarations are folded into `StructuralAnalysis.classes[]` because the
 * shared schema does not have a first-class slot for them. Extension
 * declarations without a name surface as `"on <TargetType>"` so they aren't
 * silently dropped.
 */
export class DartExtractor implements LanguageExtractor {
  readonly languageIds = ["dart"];

  extractStructure(rootNode: TreeSitterNode): StructuralAnalysis {
    const functions: StructuralAnalysis["functions"] = [];
    const classes: StructuralAnalysis["classes"] = [];
    const imports: StructuralAnalysis["imports"] = [];
    const exports: StructuralAnalysis["exports"] = [];

    // Implementation lands in subsequent tasks.
    void rootNode;
    void findChild;
    void findChildren;
    void isExported;

    return { functions, classes, imports, exports };
  }

  extractCallGraph(rootNode: TreeSitterNode): CallGraphEntry[] {
    // Implementation lands in a later task.
    void rootNode;
    return [];
  }
}
```

- [ ] **Step 2: Register DartExtractor in the extractors index**

Edit `understand-anything-plugin/packages/core/src/plugins/extractors/index.ts`. Three edits:

(a) Add the named re-export beside the others:

```ts
export { DartExtractor } from "./dart-extractor.js";
```

(b) Add the import beside the others:

```ts
import { DartExtractor } from "./dart-extractor.js";
```

(c) Add `new DartExtractor()` to `builtinExtractors` (place between `CppExtractor` and `CSharpExtractor`):

```ts
export const builtinExtractors: LanguageExtractor[] = [
  new TypeScriptExtractor(),
  new PythonExtractor(),
  new GoExtractor(),
  new RustExtractor(),
  new JavaExtractor(),
  new RubyExtractor(),
  new PhpExtractor(),
  new CppExtractor(),
  new DartExtractor(),
  new CSharpExtractor(),
  new KotlinExtractor(),
];
```

- [ ] **Step 3: Build + test (must still pass)**

```bash
pnpm --filter @understand-anything/core build 2>&1 | tail -3
pnpm --filter @understand-anything/core test 2>&1 | tail -5
```

Expected: tsc clean, all existing tests pass. (Skeleton extractor returns empty results, no behavior change for non-Dart files.)

- [ ] **Step 4: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/index.ts
git commit -m "feat(core): scaffold DartExtractor + register in builtinExtractors

Empty extractor that satisfies the LanguageExtractor interface so the
plugin pipeline can load it. Real extraction logic lands in subsequent
TDD commits.
"
```

---

## Task 4: TDD — top-level function extraction

From here through Task 12 follow strict TDD: write failing test, verify it fails for the right reason, implement minimum to pass, verify pass, commit. Each task corresponds to a roughly-coherent slice of extractor behavior.

**Reference test setup** — every test file in `__tests__/` uses the same `beforeAll` + `parse()` helper shape. Establish it once in Step 1, then re-use across Tasks 4–12.

**Files (all of Tasks 4–12):**
- Create: `understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts`
- Modify: `understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts`

- [ ] **Step 1: Create the test-file scaffold**

Create `understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts`:

```ts
import { describe, it, expect, beforeAll } from "vitest";
import { createRequire } from "node:module";
import { DartExtractor } from "../dart-extractor.js";

const require = createRequire(import.meta.url);

let Parser: any;
let Language: any;
let dartLang: any;

beforeAll(async () => {
  const mod = await import("web-tree-sitter");
  Parser = mod.Parser;
  Language = mod.Language;
  await Parser.init();
  const wasmPath = require.resolve(
    "@understand-anything/tree-sitter-dart-wasm/tree-sitter-dart.wasm",
  );
  dartLang = await Language.load(wasmPath);
});

function parse(code: string) {
  const parser = new Parser();
  parser.setLanguage(dartLang);
  const tree = parser.parse(code);
  const root = tree.rootNode;
  return { tree, parser, root };
}

describe("DartExtractor", () => {
  const extractor = new DartExtractor();

  it("has correct languageIds", () => {
    expect(extractor.languageIds).toEqual(["dart"]);
  });
});
```

Run:
```bash
pnpm --filter @understand-anything/core test src/plugins/extractors/__tests__/dart-extractor.test.ts 2>&1 | tail -10
```

Expected: 1 test passes (the `languageIds` assertion), no errors. If `beforeAll` errors, the wasm path is wrong — fix before continuing.

- [ ] **Step 2: Write the failing function-extraction tests**

Append inside the `describe("DartExtractor", …)` block:

```ts
  describe("extractStructure - functions", () => {
    it("extracts a simple top-level function with params and return type", () => {
      const { tree, parser, root } = parse(`int add(int a, int b) => a + b;\n`);
      const result = extractor.extractStructure(root);

      expect(result.functions).toHaveLength(1);
      expect(result.functions[0].name).toBe("add");
      expect(result.functions[0].params).toEqual(["a", "b"]);
      expect(result.functions[0].returnType).toBe("int");

      tree.delete();
      parser.delete();
    });

    it("extracts a function with no params and void return type", () => {
      const { tree, parser, root } = parse(`void noop() {}\n`);
      const result = extractor.extractStructure(root);

      expect(result.functions).toHaveLength(1);
      expect(result.functions[0].name).toBe("noop");
      expect(result.functions[0].params).toEqual([]);
      expect(result.functions[0].returnType).toBe("void");

      tree.delete();
      parser.delete();
    });

    it("extracts an async function with a generic return type", () => {
      const { tree, parser, root } = parse(`Future<String> fetch(String url) async { return ""; }\n`);
      const result = extractor.extractStructure(root);

      expect(result.functions).toHaveLength(1);
      expect(result.functions[0].name).toBe("fetch");
      expect(result.functions[0].params).toEqual(["url"]);
      expect(result.functions[0].returnType).toBe("Future<String>");

      tree.delete();
      parser.delete();
    });
  });
```

Run:
```bash
pnpm --filter @understand-anything/core test src/plugins/extractors/__tests__/dart-extractor.test.ts 2>&1 | tail -10
```

Expected: 3 new tests FAIL because the extractor returns empty `functions`. The `languageIds` test still passes.

- [ ] **Step 3: Implement function extraction**

In `dart-extractor.ts`, replace the `extractStructure` body. The AST shape (verified live):

- A top-level function appears as `program > function_signature` followed by a **sibling** `function_body`. (Not parent/child — `function_body` is a separate top-level node.)
- `function_signature` children: an optional return-type node (`type_identifier` or `void_type` or a generic `type` subtree), then `identifier` (the name), then `formal_parameter_list`.

Add this helper at the top of the file (after the existing `isExported` helper):

```ts
/**
 * Extract the identifier name from a function_signature / method_signature
 * node. The name is the first `identifier` child after any return-type
 * subtree.
 */
function extractFunctionName(sig: TreeSitterNode): string | null {
  const id = findChild(sig, "identifier");
  return id ? id.text : null;
}

/**
 * Extract parameter names from a `formal_parameter_list`. Each
 * `formal_parameter` child carries the parameter name as its `identifier`
 * child; we ignore the type annotation.
 */
function extractParams(sig: TreeSitterNode): string[] {
  const params: string[] = [];
  const paramList = findChild(sig, "formal_parameter_list");
  if (!paramList) return params;
  for (const p of findChildren(paramList, "formal_parameter")) {
    const id = findChild(p, "identifier");
    if (id) params.push(id.text);
  }
  return params;
}

/**
 * Extract the return type from a function_signature. The return type is the
 * first NAMED child whose type is NOT `identifier` or `formal_parameter_list`
 * or `type_parameters`. If there is no such child, the function has no
 * declared return type (Dart infers it).
 *
 * Common shapes seen during AST probing:
 *   `int add(int a, int b)` →  type_identifier "int"
 *   `void noop()`           →  void_type
 *   `Future<String> fetch()`→  type_identifier "Future" wrapped in a type with type_arguments
 */
function extractReturnType(sig: TreeSitterNode): string | undefined {
  for (let i = 0; i < sig.childCount; i++) {
    const child = sig.child(i);
    if (!child || !child.isNamed) continue;
    if (
      child.type === "identifier" ||
      child.type === "formal_parameter_list" ||
      child.type === "type_parameters"
    ) {
      // Reached the name / params without seeing a return type.
      return undefined;
    }
    // This is the return type node. Its full text (including generics) is
    // what we want.
    return child.text;
  }
  return undefined;
}
```

Now replace `extractStructure`:

```ts
  extractStructure(rootNode: TreeSitterNode): StructuralAnalysis {
    const functions: StructuralAnalysis["functions"] = [];
    const classes: StructuralAnalysis["classes"] = [];
    const imports: StructuralAnalysis["imports"] = [];
    const exports: StructuralAnalysis["exports"] = [];

    for (let i = 0; i < rootNode.childCount; i++) {
      const node = rootNode.child(i);
      if (!node) continue;

      switch (node.type) {
        case "function_signature":
          this.extractTopLevelFunction(node, functions, exports);
          break;
      }
    }

    return { functions, classes, imports, exports };
  }

  // ---- Private helpers ----

  private extractTopLevelFunction(
    sig: TreeSitterNode,
    functions: StructuralAnalysis["functions"],
    exports: StructuralAnalysis["exports"],
  ): void {
    const name = extractFunctionName(sig);
    if (!name) return;
    functions.push({
      name,
      lineRange: [sig.startPosition.row + 1, sig.endPosition.row + 1],
      params: extractParams(sig),
      returnType: extractReturnType(sig),
    });
    if (isExported(name)) {
      exports.push({ name, lineNumber: sig.startPosition.row + 1 });
    }
  }
```

The four `void X;` lines from the Task 3 skeleton inside `extractStructure` are gone now (replaced by the real body). Leave the `void rootNode;` line in `extractCallGraph` for now — it'll be replaced when Task 12 implements call-graph extraction.

- [ ] **Step 4: Run the function tests and verify pass**

```bash
pnpm --filter @understand-anything/core test src/plugins/extractors/__tests__/dart-extractor.test.ts 2>&1 | tail -15
```

Expected: all 4 tests (including `languageIds`) pass. If one fails, inspect actual vs expected — adjust the helper if the AST shape for that case differs from what was assumed.

- [ ] **Step 5: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — top-level function extraction"
```

---

## Task 5: TDD — class extraction (plain, abstract, with inheritance)

**Files:**
- Modify: `dart-extractor.ts`, `dart-extractor.test.ts`

- [ ] **Step 1: Write the failing class tests**

Append inside `describe("DartExtractor", …)`:

```ts
  describe("extractStructure - classes", () => {
    it("extracts a class with fields and methods", () => {
      const { tree, parser, root } = parse(`class Counter {
  int count = 0;
  String? label;
  void increment() { count++; }
  int get value => count;
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Counter");
      expect(result.classes[0].methods).toContain("increment");
      // method declarations land in functions[] too (matching Kotlin convention)
      expect(result.functions.map((f) => f.name)).toContain("increment");

      tree.delete();
      parser.delete();
    });

    it("extracts an empty class", () => {
      const { tree, parser, root } = parse(`class Empty {}\n`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Empty");
      expect(result.classes[0].methods).toEqual([]);

      tree.delete();
      parser.delete();
    });

    it("extracts an abstract class with method requirements", () => {
      const { tree, parser, root } = parse(`abstract class Shape {
  double area();
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Shape");
      expect(result.classes[0].methods).toContain("area");

      tree.delete();
      parser.delete();
    });

    it("extracts a class with extends + with + implements clauses", () => {
      const { tree, parser, root } = parse(`class Square extends Shape with Comparable<Square> implements Cloneable {
  double side;
  Square(this.side);
  double area() => side * side;
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Square");
      expect(result.classes[0].methods).toContain("area");

      tree.delete();
      parser.delete();
    });
  });
```

Run tests — expect 4 new failures.

- [ ] **Step 2: Implement class extraction**

Class AST shape (verified live):

- `program > class_definition { identifier(name), class_body }`. Inheritance clauses (`extends Foo`, `with Mixin`, `implements Iface`) appear as siblings between the `identifier` and `class_body`. We ignore them for now (out of scope for this task — captured at the class node's text level if ever needed; not required for the graph).
- `class_body > method_signature { function_signature { return_type, identifier, formal_parameter_list } }` followed by a sibling `function_body` (mirroring the top-level shape).
- `class_body > declaration { type_identifier, initialized_identifier_list { initialized_identifier { identifier(name) } } }` — this is a field declaration.

Add a helper for class-body walking (after the function helpers):

```ts
/**
 * Walk a `class_body` (or `extension_body` / `enum_body`) and collect
 * `method_signature` declarations into the class's `methods` array AND the
 * top-level `functions` array, mirroring KotlinExtractor.collectClassBody.
 */
function collectClassBody(
  body: TreeSitterNode,
  methods: string[],
  properties: string[],
  functions: StructuralAnalysis["functions"],
  exports: StructuralAnalysis["exports"],
): void {
  for (let i = 0; i < body.childCount; i++) {
    const member = body.child(i);
    if (!member) continue;

    if (member.type === "method_signature") {
      const inner = findChild(member, "function_signature");
      if (!inner) continue;
      const name = extractFunctionName(inner);
      if (!name) continue;
      methods.push(name);
      functions.push({
        name,
        lineRange: [member.startPosition.row + 1, member.endPosition.row + 1],
        params: extractParams(inner),
        returnType: extractReturnType(inner),
      });
      if (isExported(name)) {
        exports.push({ name, lineNumber: member.startPosition.row + 1 });
      }
    } else if (member.type === "declaration") {
      // Field declaration — surface initialized_identifier names as properties.
      const list = findChild(member, "initialized_identifier_list");
      if (!list) continue;
      for (const init of findChildren(list, "initialized_identifier")) {
        const id = findChild(init, "identifier");
        if (id) properties.push(id.text);
      }
    }
  }
}
```

Add a case to the top-level switch:

```ts
        case "class_definition":
          this.extractClassDefinition(node, classes, functions, exports);
          break;
```

Add the private method:

```ts
  private extractClassDefinition(
    declNode: TreeSitterNode,
    classes: StructuralAnalysis["classes"],
    functions: StructuralAnalysis["functions"],
    exports: StructuralAnalysis["exports"],
  ): void {
    const nameNode = findChild(declNode, "identifier");
    if (!nameNode) return;
    const name = nameNode.text;

    const methods: string[] = [];
    const properties: string[] = [];

    const body = findChild(declNode, "class_body");
    if (body) {
      collectClassBody(body, methods, properties, functions, exports);
    }

    classes.push({
      name,
      lineRange: [declNode.startPosition.row + 1, declNode.endPosition.row + 1],
      methods,
      properties,
    });

    if (isExported(name)) {
      exports.push({ name, lineNumber: declNode.startPosition.row + 1 });
    }
  }
```

Run tests — expect all class tests to pass.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — class extraction with fields + methods"
```

---

## Task 6: TDD — constructors (default, named, factory)

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

Append:

```ts
  describe("extractStructure - constructors", () => {
    it("treats an unnamed constructor as a method named after the class", () => {
      const { tree, parser, root } = parse(`class Foo {
  int x;
  Foo(this.x);
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes[0].methods).toContain("Foo");
      tree.delete();
      parser.delete();
    });

    it("treats a named constructor as Class.named", () => {
      const { tree, parser, root } = parse(`class Foo {
  int x;
  Foo.zero() : x = 0;
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes[0].methods).toContain("Foo.zero");
      tree.delete();
      parser.delete();
    });

    it("treats a factory named constructor as Class.named", () => {
      const { tree, parser, root } = parse(`class Foo {
  int x;
  Foo(this.x);
  factory Foo.fromString(String s) => Foo(int.parse(s));
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes[0].methods).toContain("Foo.fromString");
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 3 failures.

- [ ] **Step 2: Implement constructor handling**

AST shapes (verified live):

- Unnamed: `class_body > declaration > constructor_signature { identifier(class), formal_parameter_list }` — only ONE identifier.
- Named: `class_body > declaration > constructor_signature { identifier(class), identifier(named), formal_parameter_list }` — TWO identifiers; second is the named-constructor name.
- Factory: `class_body > method_signature > factory_constructor_signature { identifier(class), identifier(named), formal_parameter_list }` — wrapped in `method_signature`.

Extend `collectClassBody`'s `for` loop. Both the `method_signature` branch and the `declaration` branch get a constructor check **at the top** that short-circuits before the existing logic. Full revised loop body:

```ts
    if (member.type === "method_signature") {
      // Factory constructor lives inside method_signature as
      // factory_constructor_signature; check that first.
      const factory = findChild(member, "factory_constructor_signature");
      if (factory) {
        const name = constructorName(factory);
        if (name) {
          methods.push(name);
          functions.push({
            name,
            lineRange: [member.startPosition.row + 1, member.endPosition.row + 1],
            params: extractParams(factory),
            returnType: undefined,
          });
          if (isExported(name)) {
            exports.push({ name, lineNumber: member.startPosition.row + 1 });
          }
        }
        continue;
      }
      // ...existing function_signature handling unchanged...
    } else if (member.type === "declaration") {
      const ctor = findChild(member, "constructor_signature");
      if (ctor) {
        const name = constructorName(ctor);
        if (name) {
          methods.push(name);
          functions.push({
            name,
            lineRange: [member.startPosition.row + 1, member.endPosition.row + 1],
            params: extractParams(ctor),
            returnType: undefined,
          });
          if (isExported(name)) {
            exports.push({ name, lineNumber: member.startPosition.row + 1 });
          }
        }
        continue;
      }
      // ...existing field-declaration handling unchanged...
    }
```

Add the helper `constructorName` at the top:

```ts
/**
 * Build a constructor's method-graph name from a constructor_signature /
 * factory_constructor_signature node:
 *   - one identifier  → unnamed constructor, name = "<Class>"
 *   - two identifiers → named constructor,   name = "<Class>.<named>"
 */
function constructorName(sig: TreeSitterNode): string | null {
  const ids = findChildren(sig, "identifier");
  if (ids.length === 0) return null;
  if (ids.length === 1) return ids[0].text;
  return `${ids[0].text}.${ids[1].text}`;
}
```

Run tests — expect all constructor tests pass; previously-passing tests remain green.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — constructor naming (default/named/factory)"
```

---

## Task 7: TDD — mixins

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractStructure - mixins", () => {
    it("extracts a plain mixin as a class-like entry", () => {
      const { tree, parser, root } = parse(`mixin Walker {
  void walk() {}
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Walker");
      expect(result.classes[0].methods).toContain("walk");
      tree.delete();
      parser.delete();
    });

    it("extracts a mixin with an `on` constraint", () => {
      const { tree, parser, root } = parse(`mixin Runner on Walker {
  void run() {}
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes[0].name).toBe("Runner");
      expect(result.classes[0].methods).toContain("run");
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 2 failures.

- [ ] **Step 2: Implement mixin extraction**

AST shape: `program > mixin_declaration { identifier(name), [type_identifier(on)], class_body }`. Same body shape as `class_definition`.

Add case to the top-level switch:

```ts
        case "mixin_declaration":
          this.extractMixinDeclaration(node, classes, functions, exports);
          break;
```

Implement (almost identical to `extractClassDefinition`; refactoring opportunity but kept separate for clarity):

```ts
  private extractMixinDeclaration(
    declNode: TreeSitterNode,
    classes: StructuralAnalysis["classes"],
    functions: StructuralAnalysis["functions"],
    exports: StructuralAnalysis["exports"],
  ): void {
    const nameNode = findChild(declNode, "identifier");
    if (!nameNode) return;
    const name = nameNode.text;

    const methods: string[] = [];
    const properties: string[] = [];

    const body = findChild(declNode, "class_body");
    if (body) {
      collectClassBody(body, methods, properties, functions, exports);
    }

    classes.push({
      name,
      lineRange: [declNode.startPosition.row + 1, declNode.endPosition.row + 1],
      methods,
      properties,
    });

    if (isExported(name)) {
      exports.push({ name, lineNumber: declNode.startPosition.row + 1 });
    }
  }
```

Run tests — expect mixin tests pass.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — mixin declarations"
```

---

## Task 8: TDD — extensions

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractStructure - extensions", () => {
    it("extracts a named extension on String", () => {
      const { tree, parser, root } = parse(`extension StringX on String {
  String shout() => toUpperCase() + '!';
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("StringX");
      expect(result.classes[0].methods).toContain("shout");
      tree.delete();
      parser.delete();
    });

    it("names an anonymous extension after its target type", () => {
      const { tree, parser, root } = parse(`extension on int {
  int squared() => this * this;
}
`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      // Anonymous extension on int → "on int" so it isn't dropped.
      expect(result.classes[0].name).toBe("on int");
      expect(result.classes[0].methods).toContain("squared");
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 2 failures.

- [ ] **Step 2: Implement extension extraction**

AST shape (verified):

- Named: `extension_declaration { identifier(name), type_identifier(on-type), extension_body }`
- Anonymous: `extension_declaration { type_identifier(on-type), extension_body }` — no leading identifier.

Add the case to the top-level switch:

```ts
        case "extension_declaration":
          this.extractExtensionDeclaration(node, classes, functions, exports);
          break;
```

Implement:

```ts
  private extractExtensionDeclaration(
    declNode: TreeSitterNode,
    classes: StructuralAnalysis["classes"],
    functions: StructuralAnalysis["functions"],
    exports: StructuralAnalysis["exports"],
  ): void {
    // Try named extension first: leading `identifier` child is the name.
    const idNode = findChild(declNode, "identifier");
    let name: string;
    if (idNode) {
      name = idNode.text;
    } else {
      // Anonymous: name the entry after the target type so the graph builder
      // doesn't drop it. The on-type is the first `type_identifier`.
      const onType = findChild(declNode, "type_identifier");
      if (!onType) return;
      name = `on ${onType.text}`;
    }

    const methods: string[] = [];
    const properties: string[] = [];

    const body = findChild(declNode, "extension_body");
    if (body) {
      collectClassBody(body, methods, properties, functions, exports);
    }

    classes.push({
      name,
      lineRange: [declNode.startPosition.row + 1, declNode.endPosition.row + 1],
      methods,
      properties,
    });

    if (isExported(name)) {
      exports.push({ name, lineNumber: declNode.startPosition.row + 1 });
    }
  }
```

Run — expect extension tests pass.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — extension declarations (named + anonymous)"
```

---

## Task 9: TDD — enums

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractStructure - enums", () => {
    it("extracts a simple enum and surfaces its constants as properties", () => {
      const { tree, parser, root } = parse(`enum Color { red, green, blue }\n`);
      const result = extractor.extractStructure(root);

      expect(result.classes).toHaveLength(1);
      expect(result.classes[0].name).toBe("Color");
      expect(result.classes[0].properties).toEqual(["red", "green", "blue"]);
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 1 failure.

- [ ] **Step 2: Implement enum extraction**

AST shape: `enum_declaration { identifier(name), enum_body { enum_constant { identifier }... } }`.

Add case to the top-level switch:

```ts
        case "enum_declaration":
          this.extractEnumDeclaration(node, classes, exports);
          break;
```

Implement:

```ts
  private extractEnumDeclaration(
    declNode: TreeSitterNode,
    classes: StructuralAnalysis["classes"],
    exports: StructuralAnalysis["exports"],
  ): void {
    const nameNode = findChild(declNode, "identifier");
    if (!nameNode) return;
    const name = nameNode.text;

    const properties: string[] = [];
    const body = findChild(declNode, "enum_body");
    if (body) {
      for (const k of findChildren(body, "enum_constant")) {
        const id = findChild(k, "identifier");
        if (id) properties.push(id.text);
      }
    }

    classes.push({
      name,
      lineRange: [declNode.startPosition.row + 1, declNode.endPosition.row + 1],
      methods: [],
      properties,
    });

    if (isExported(name)) {
      exports.push({ name, lineNumber: declNode.startPosition.row + 1 });
    }
  }
```

Run — expect enum test passes.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — enum declarations"
```

---

## Task 10: TDD — import + export directives

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractStructure - imports", () => {
    it("extracts a package import with no specifiers", () => {
      const { tree, parser, root } = parse(`import 'package:flutter/material.dart';\n`);
      const result = extractor.extractStructure(root);

      expect(result.imports).toHaveLength(1);
      expect(result.imports[0].source).toBe("package:flutter/material.dart");
      expect(result.imports[0].specifiers).toEqual([]);
      tree.delete();
      parser.delete();
    });

    it("extracts a relative import", () => {
      const { tree, parser, root } = parse(`import './foo.dart';\n`);
      const result = extractor.extractStructure(root);

      expect(result.imports[0].source).toBe("./foo.dart");
      tree.delete();
      parser.delete();
    });

    it("extracts a `show` clause as specifiers", () => {
      const { tree, parser, root } = parse(`import 'foo.dart' show Bar, Baz;\n`);
      const result = extractor.extractStructure(root);

      expect(result.imports[0].source).toBe("foo.dart");
      expect(result.imports[0].specifiers).toEqual(["Bar", "Baz"]);
      tree.delete();
      parser.delete();
    });

    it("extracts an `as` prefix as the sole specifier", () => {
      const { tree, parser, root } = parse(`import 'bar.dart' as b;\n`);
      const result = extractor.extractStructure(root);

      expect(result.imports[0].source).toBe("bar.dart");
      expect(result.imports[0].specifiers).toEqual(["b"]);
      tree.delete();
      parser.delete();
    });
  });

  describe("extractStructure - exports", () => {
    it("extracts a top-level export directive", () => {
      const { tree, parser, root } = parse(`export 'shared.dart';\n`);
      const result = extractor.extractStructure(root);

      const sharedExport = result.exports.find((e) => e.name === "shared.dart");
      expect(sharedExport).toBeDefined();
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 5 new failures.

- [ ] **Step 2: Implement import/export extraction**

AST shape (verified):

- Top-level wrapper: `import_or_export { library_import | library_export }`.
- `library_import { import_specification { configurable_uri { uri { string_literal }, [combinator { 'show', identifier, ... }], [identifier(as-prefix)] } } }`.
  - The `string_literal` text contains the surrounding quotes (`'foo.dart'`).
  - A `combinator` named child holds `show`/`hide` keyword + identifier list.
  - An `identifier` named child at the import_specification level is the `as` prefix.
- `library_export { configurable_uri { uri { string_literal } } }`.

Add a helper at top of file:

```ts
/**
 * Unwrap the string-literal text from `uri > string_literal`, stripping the
 * surrounding single or double quotes.
 */
function uriText(uriNode: TreeSitterNode): string | null {
  const lit = findChild(uriNode, "string_literal");
  if (!lit) return null;
  return lit.text.replace(/^['"]|['"]$/g, "");
}
```

Add cases to the top-level switch:

```ts
        case "import_or_export":
          this.extractImportOrExport(node, imports, exports);
          break;
```

Implement:

```ts
  private extractImportOrExport(
    declNode: TreeSitterNode,
    imports: StructuralAnalysis["imports"],
    exports: StructuralAnalysis["exports"],
  ): void {
    const libImport = findChild(declNode, "library_import");
    if (libImport) {
      this.extractLibraryImport(libImport, imports);
      return;
    }
    const libExport = findChild(declNode, "library_export");
    if (libExport) {
      this.extractLibraryExport(libExport, declNode, exports);
    }
  }

  private extractLibraryImport(
    libImport: TreeSitterNode,
    imports: StructuralAnalysis["imports"],
  ): void {
    const spec = findChild(libImport, "import_specification");
    if (!spec) return;

    const configurable = findChild(spec, "configurable_uri");
    const uri = configurable ? findChild(configurable, "uri") : null;
    if (!uri) return;
    const source = uriText(uri);
    if (!source) return;

    const specifiers: string[] = [];

    // `show Bar, Baz` — combinator has identifier children for the shown names.
    const combinators = findChildren(spec, "combinator");
    for (const c of combinators) {
      for (const id of findChildren(c, "identifier")) {
        specifiers.push(id.text);
      }
    }

    // `as Foo` — a direct `identifier` child of import_specification is the
    // alias. Has to come AFTER the configurable_uri in source order.
    const asId = findChild(spec, "identifier");
    if (asId && specifiers.length === 0) {
      specifiers.push(asId.text);
    }

    imports.push({
      source,
      specifiers,
      lineNumber: libImport.startPosition.row + 1,
    });
  }

  private extractLibraryExport(
    libExport: TreeSitterNode,
    outerNode: TreeSitterNode,
    exports: StructuralAnalysis["exports"],
  ): void {
    const configurable = findChild(libExport, "configurable_uri");
    const uri = configurable ? findChild(configurable, "uri") : null;
    if (!uri) return;
    const source = uriText(uri);
    if (!source) return;
    exports.push({
      name: source,
      lineNumber: outerNode.startPosition.row + 1,
    });
  }
```

Run — expect import + export tests pass.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — import directives (package/relative/show/as) + export directives"
```

---

## Task 11: TDD — visibility (underscore-prefix rule)

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractStructure - visibility", () => {
    it("does NOT export a top-level declaration whose name starts with _", () => {
      const { tree, parser, root } = parse(`int _helper() => 1;
class _PrivateImpl {}
`);
      const result = extractor.extractStructure(root);

      const names = result.exports.map((e) => e.name);
      expect(names).not.toContain("_helper");
      expect(names).not.toContain("_PrivateImpl");
      tree.delete();
      parser.delete();
    });

    it("DOES export a top-level declaration without an underscore prefix", () => {
      const { tree, parser, root } = parse(`int helper() => 1;
class Public {}
`);
      const result = extractor.extractStructure(root);

      const names = result.exports.map((e) => e.name);
      expect(names).toEqual(expect.arrayContaining(["helper", "Public"]));
      tree.delete();
      parser.delete();
    });

    it("does NOT export class members whose names start with _", () => {
      const { tree, parser, root } = parse(`class Counter {
  void _helper() {}
  void publicMethod() {}
}
`);
      const result = extractor.extractStructure(root);

      const names = result.exports.map((e) => e.name);
      expect(names).toContain("publicMethod");
      expect(names).not.toContain("_helper");
      tree.delete();
      parser.delete();
    });
  });
```

Run — first two tests should already pass (current implementation already uses `isExported(name)` everywhere). The class-member test should also pass thanks to `collectClassBody` calling `isExported`. If all three pass without code changes, this task is just a coverage commit.

- [ ] **Step 2: Confirm all 3 pass; if any fail, add a missing `isExported` guard at the relevant emit site**

```bash
pnpm --filter @understand-anything/core test src/plugins/extractors/__tests__/dart-extractor.test.ts 2>&1 | tail -10
```

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "test(core): DartExtractor — visibility rule (underscore prefix)"
```

---

## Task 12: TDD — call graph

**Files:** dart-extractor.{ts,test.ts}

- [ ] **Step 1: Write failing tests**

```ts
  describe("extractCallGraph", () => {
    it("attributes a top-level call to its enclosing function", () => {
      const { tree, parser, root } = parse(`int helper() => 1;
int caller() {
  return helper();
}
`);
      const entries = extractor.extractCallGraph(root);

      const helperCall = entries.find((e) => e.callee === "helper");
      expect(helperCall).toBeDefined();
      expect(helperCall!.caller).toBe("caller");
      tree.delete();
      parser.delete();
    });

    it("attributes a method call (x.foo()) to its enclosing function", () => {
      const { tree, parser, root } = parse(`void run() {
  "hi".toUpperCase();
}
`);
      const entries = extractor.extractCallGraph(root);

      const callees = entries.map((e) => e.callee);
      expect(callees).toContain("toUpperCase");
      tree.delete();
      parser.delete();
    });

    it("returns an empty array when there are no calls", () => {
      const { tree, parser, root } = parse(`int a() => 1;\n`);
      const entries = extractor.extractCallGraph(root);
      expect(entries).toEqual([]);
      tree.delete();
      parser.delete();
    });
  });
```

Run — expect 2 failures (third passes because the stub returns `[]`).

- [ ] **Step 2: Implement call-graph extraction**

AST shape for a Dart call (verified live):

```
function_body
  block
    expression_statement
      identifier "print"           ← bare-call callee
      selector
        argument_part
```

And for a method-style call:

```
expression_statement
  string_literal "'hi'"            ← receiver
  selector
    unconditional_assignable_selector
      identifier "toUpperCase"     ← method-call callee (last identifier in the selector chain)
  selector
    argument_part
```

Key insight: in Dart's grammar, a call is represented as a target expression followed by one or more `selector` siblings, with the LAST `selector` containing an `argument_part`. The callee identifier is either:

- The first identifier in the expression_statement (bare call), OR
- The last identifier appearing inside any `unconditional_assignable_selector` before the `selector` that contains `argument_part`.

Pragmatic approach: walk every node, and whenever we see a `selector` containing an `argument_part`, look for the callee as the IDENTIFIER token immediately preceding it in the parent's children. If none, look inside the previous sibling `selector` for an `identifier` (the method name in chained call).

Replace `extractCallGraph`:

```ts
  extractCallGraph(rootNode: TreeSitterNode): CallGraphEntry[] {
    const entries: CallGraphEntry[] = [];
    const functionStack: string[] = [];

    const walk = (node: TreeSitterNode) => {
      let pushed = false;

      // Push function_signature names (both top-level and inside method_signature).
      if (node.type === "function_signature") {
        const name = extractFunctionName(node);
        if (name) {
          functionStack.push(name);
          pushed = true;
        }
      }

      // Detect a call: any `selector` node containing an `argument_part`.
      if (
        node.type === "selector" &&
        findChild(node, "argument_part") &&
        functionStack.length > 0
      ) {
        const callee = this.extractCalleeName(node);
        if (callee) {
          entries.push({
            caller: functionStack[functionStack.length - 1],
            callee,
            lineNumber: node.startPosition.row + 1,
          });
        }
      }

      for (let i = 0; i < node.childCount; i++) {
        const child = node.child(i);
        if (child) walk(child);
      }

      if (pushed) functionStack.pop();
    };

    walk(rootNode);
    return entries;
  }

  /**
   * Find the callee name for a `selector` node that contains an
   * `argument_part`. We look at the parent's children: the callee identifier
   * is either the immediately-preceding `identifier` sibling (bare call) or
   * the last `identifier` inside the immediately-preceding `selector`
   * sibling's `unconditional_assignable_selector` (method call).
   */
  private extractCalleeName(callSelector: TreeSitterNode): string | null {
    const parent = callSelector.parent;
    if (!parent) return null;

    // Find this selector's index in the parent.
    let myIdx = -1;
    for (let i = 0; i < parent.childCount; i++) {
      if (parent.child(i) === callSelector) {
        myIdx = i;
        break;
      }
    }
    if (myIdx <= 0) return null;

    const prev = parent.child(myIdx - 1);
    if (!prev) return null;

    if (prev.type === "identifier") return prev.text;

    if (prev.type === "selector") {
      // Method call shape: previous selector wraps unconditional_assignable_selector.
      const inner = findChild(prev, "unconditional_assignable_selector");
      if (inner) {
        // Pick the LAST identifier inside the inner selector — that's the
        // method name (earlier identifiers, if any, are receiver fragments).
        let last: string | null = null;
        for (let i = 0; i < inner.childCount; i++) {
          const child = inner.child(i);
          if (child && child.type === "identifier") last = child.text;
        }
        return last;
      }
    }

    return null;
  }
```

Run — expect call-graph tests pass.

- [ ] **Step 3: Commit**

```bash
git add understand-anything-plugin/packages/core/src/plugins/extractors/dart-extractor.ts \
        understand-anything-plugin/packages/core/src/plugins/extractors/__tests__/dart-extractor.test.ts
git commit -m "feat(core): DartExtractor — call graph extraction"
```

---

## Task 13: Final verification + lint + push

**Files:** none — verification only.

- [ ] **Step 1: Run the full core test suite**

```bash
cd /Users/thejesh/Git/Understand-Anything
pnpm --filter @understand-anything/core test 2>&1 | tail -20
```

Expected: All existing tests pass AND the new Dart tests pass. Look for the summary line — should show counts like `Tests <existing + ~22> passed (<n> files)`. If any pre-existing test failed, investigate before continuing.

- [ ] **Step 2: Run the skill build (must not regress)**

```bash
pnpm --filter @understand-anything/skill build 2>&1 | tail -5
```

Expected: tsc clean.

- [ ] **Step 3: Run lint across the project**

```bash
pnpm lint 2>&1 | tail -10
```

Expected: clean (or only pre-existing warnings unrelated to our changes). Fix any errors introduced by our changes inline; do NOT commit lint warnings.

- [ ] **Step 4: Run the full test suite**

```bash
pnpm test 2>&1 | tail -10
```

Expected: full repo suite passes, no regressions.

- [ ] **Step 5: Manual smoke — verify integration with the real TreeSitterPlugin**

Write a one-off Node script at `/tmp/smoke-dart.mjs`:

```js
import { TreeSitterPlugin } from "@understand-anything/core";
import { dartConfig } from "@understand-anything/core/languages";

const plugin = new TreeSitterPlugin([dartConfig]);
await plugin.init();

const dart = `
import 'package:flutter/material.dart';

class Counter {
  int count = 0;
  void increment() => count++;
}

void main() {
  Counter().increment();
}
`;

const result = plugin.analyzeFile("example.dart", dart);
console.log(JSON.stringify(result, null, 2));
```

Run from the core package:

```bash
cd understand-anything-plugin/packages/core
cp /tmp/smoke-dart.mjs ./smoke-dart.mjs
node smoke-dart.mjs
rm smoke-dart.mjs
```

Expected output: a `StructuralAnalysis` JSON with non-empty `functions` (containing `main`, `increment`), `classes` (containing `Counter`), `imports` (containing `package:flutter/material.dart`), `exports` (containing `Counter`, `main`).

If the imports/exports are subtly different from what the unit tests assert (e.g., empty `specifiers`), that's fine — the integration test just confirms the plugin loads and produces non-empty results.

- [ ] **Step 6: Push the branch**

```bash
git push -u origin feat/dart-language-support
```

Expected: branch lands on remote. PR creation is a separate user step — do NOT open a PR autonomously.

---

## Coverage map (spec → tasks)

| Spec section | Task(s) |
|---|---|
| File-level changes — tree-sitter-dart-wasm package + workspace wiring | Task 1 |
| File-level changes — core dependency + dartConfig + index | Task 2 |
| File-level changes — DartExtractor + index | Task 3 |
| File-level changes — dart-extractor.test.ts | Tasks 4–12 |
| `dartConfig` shape | Task 2, Step 2 |
| WASM grammar source — vendored | Task 1 |
| DartExtractor — top-level AST nodes handled (functions) | Task 4 |
| DartExtractor — classes + constructors | Tasks 5, 6 |
| DartExtractor — mixins | Task 7 |
| DartExtractor — extensions (named + anonymous) | Task 8 |
| DartExtractor — enums | Task 9 |
| DartExtractor — imports (three forms) | Task 10 |
| Top-level `export` directive | Task 10 |
| Visibility rule (underscore prefix) | Task 11 |
| Class body walking convention (methods → functions[]) | Task 5, Step 2 |
| Call graph (bare + method calls) | Task 12 |
| Error handling (inherited from existing pipeline; no new modes) | covered implicitly; verified by Task 13 smoke |
| Verification commands | Task 13 |
| Edge cases NOT handled (records, patterns, `part of`) | not implemented, by design — spec rationale stands |

## Self-review notes (already applied)

- All step code blocks include the EXACT code to write; no "fill in similar code" cross-references.
- Method signatures used across tasks (`extractFunctionName`, `extractParams`, `extractReturnType`, `collectClassBody`, `constructorName`, `uriText`, `isExported`) are consistent everywhere they appear.
- The `extractCallGraph` implementation in Task 12 uses the same `extractFunctionName` helper introduced in Task 4 — no name drift.
- AST shapes for every walked node have been verified live against the freshly-built wasm; the plan is not speculating about grammar structure.
- Each task ends with a commit step so progress is incremental and the branch always builds.
- Task 11 may pass without code changes if Tasks 4–10 wired `isExported` correctly throughout; this is an intentional "coverage commit" tied to the spec's call-out that visibility is the one thing reviewers will trip on.
