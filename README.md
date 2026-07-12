# BIPL Language Server (SLE Assignment 3, Option LSP.2)

A Language Server Protocol implementation for **BIPL** — the Basic
Imperative Programming Language from the [softlang/yas](https://github.com/softlang/yas/tree/master/languages/BIPL)
repository of the Software Languages Book — with a VS Code client.

The architecture extends the FSML LSP demo from the course's LSP lecture
(`languages/FSML/Python/LSP` in YAS): a Python server built on
[pygls](https://github.com/openlawlibrary/pygls) talking to a thin
TypeScript VS Code client over stdio.

## The language

BIPL is a small imperative language (a fragment of C): integer and boolean
values, assignment, `if`/`else`, `while`, blocks, and
arithmetic/comparison/logical expressions. The parser follows the concrete
syntax in `languages/BIPL/cs.egl` of YAS (including its right-recursive
binary operators), and the typing discipline follows
`Language/BIPL/TypeChecker.hs`.

Two properties make BIPL's static semantics interesting for an LSP:

- **No declarations, no boolean literals.** A variable's type must be
  *inferred* — from its first assignment, or, for never-assigned variables,
  from how it is used (`i <= x` makes `x` an int; `if (b)` makes `b` a bool).
- **Program inputs.** BIPL programs read inputs from the initial store, so
  official YAS samples (e.g. `factorialV1.bipl`, "Assume x to be positive")
  deliberately read variables they never assign. A naive "use before
  assignment is an error" rule would flag the official samples as broken.
  The server therefore distinguishes:
  - `input-variable` (Information): never assigned anywhere — treated as a
    program input;
  - `maybe-uninitialized` (Warning): assigned somewhere, but not *definitely*
    assigned on every path reaching the read (branch-sensitive definite
    assignment: `if` requires both branches, a `while` body may run zero
    times).

## LSP features

- **Diagnostics**: lexical and syntax errors (with recovery), type errors
  (non-boolean conditions, operand type mismatches, conflicting assignments),
  and the two data-flow diagnostics above
- **Hover**: a variable's inferred type, input status, and assignment/read
  counts; keyword documentation
- **Go to definition**: a variable's first assignment
- **Find references** and **rename** (with identifier validity check)
- **Completion**: keywords and known variables with their types
- **Document symbols**: outline of variables with inferred types
- **Quick fix**: initialize a maybe-uninitialized variable at program start

## Build and test the server

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -e '.[test]'
pytest
```

The test suite includes running the analyzer over all official YAS BIPL
samples (`examples/yas-samples/`), which must be free of errors.

## Try the analyzer without an editor

```bash
python - <<'PY'
from pathlib import Path
from bipl_lsp.analysis import analyze

result = analyze(Path('examples/bad.bipl').read_text())
for d in result.diagnostics:
    print(f'{d.severity.name}: {d.message} '
          f'at {d.range.start.line + 1}:{d.range.start.character + 1}')
PY
```

## VS Code demo client

```bash
cd vscode
npm install
npm run compile
```

Open the `vscode` folder in VS Code and press `F5` to launch an Extension
Development Host. The extension assumes `bipl-lsp` is on `PATH`, so start
VS Code from a shell with the virtual environment activated:

```bash
. .venv/bin/activate
code vscode
```

In the Extension Development Host:

1. Open `examples/good.bipl` — note the blue *program input* hints on `x`
   and `y`, and hover them to see the types inferred from use.
2. Open `examples/bad.bipl` — inspect the type errors and warnings.
3. Hover a variable; Ctrl/Cmd-click it to jump to its first assignment.
4. Right-click → Find All References, or F2 to rename.
5. Trigger completion (Ctrl-Space) to see keywords and typed variables.
6. Use the quick fix (lightbulb) on a *maybe-uninitialized* warning.
