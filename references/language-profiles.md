# Language Profiles

MCD-Flow is tech-stack agnostic via a per-project language profile stored at `.mcd/config.json`. Phase 1 (`mcd-plan`) detects the language, writes this file, and every later phase reads it to know how to emit code, stubs, imports, tests, and shell commands.

## Config schema

```json
{
  "language": "typescript",
  "profile": {
    "file_ext": ".ts",
    "test_ext": ".test.ts",
    "src_dir": "src",
    "test_dir": "tests",
    "types_dir": "types",
    "stub_body": "throw new Error('not implemented')",
    "comment_prefix": "//",
    "validation_lib": "zod",
    "test_framework": "vitest",
    "commands": {
      "lint": "npx eslint {file} --fix",
      "typecheck": "npx tsc --noEmit",
      "test": "npx vitest run --reporter=verbose {test_file}"
    }
  }
}
```

`{file}` / `{test_file}` are placeholders the orchestrator substitutes at dispatch time.

## Built-in profiles

### typescript (default when `package.json` is present)

```json
{
  "language": "typescript",
  "profile": {
    "file_ext": ".ts",
    "test_ext": ".test.ts",
    "src_dir": "src",
    "test_dir": "tests",
    "types_dir": "types",
    "stub_body": "throw new Error('not implemented')",
    "comment_prefix": "//",
    "validation_lib": "zod",
    "test_framework": "vitest",
    "commands": {
      "lint": "npx eslint {file} --fix",
      "typecheck": "npx tsc --noEmit",
      "test": "npx vitest run --reporter=verbose {test_file}"
    }
  }
}
```

### python (when `pyproject.toml` or `requirements.txt` is present)

```json
{
  "language": "python",
  "profile": {
    "file_ext": ".py",
    "test_ext": "_test.py",
    "src_dir": "src",
    "test_dir": "tests",
    "types_dir": "src/types",
    "stub_body": "raise NotImplementedError",
    "comment_prefix": "#",
    "validation_lib": "pydantic",
    "test_framework": "pytest",
    "commands": {
      "lint": "ruff check --fix {file}",
      "typecheck": "mypy --strict {file}",
      "test": "pytest -v {test_file}"
    }
  }
}
```

### go (when `go.mod` is present)

```json
{
  "language": "go",
  "profile": {
    "file_ext": ".go",
    "test_ext": "_test.go",
    "src_dir": "internal",
    "test_dir": "internal",
    "types_dir": "internal/types",
    "stub_body": "panic(\"not implemented\")",
    "comment_prefix": "//",
    "validation_lib": "go-playground/validator",
    "test_framework": "go test",
    "commands": {
      "lint": "gofmt -w {file} && go vet {file}",
      "typecheck": "go build ./...",
      "test": "go test -v -run . {test_file}"
    }
  }
}
```

## Detection rules (Phase 1)

Scan the project root for these markers, in order — first match wins:

| Marker file | Language |
|-------------|----------|
| `package.json` | typescript |
| `pyproject.toml` or `requirements.txt` | python |
| `go.mod` | go |
| (none) | **ask user** |

If a marker exists but the user wants a different profile (e.g. Python project using unittest not pytest), Phase 1 gate lets them override.

## Adding a new profile

Copy one of the built-ins, change the fields, save to `.mcd/config.json` before Phase 1 completes. No code changes needed — the phase commands read this config dynamically.

## Consumer phases

- **mcd-plan (Phase 1)** — detects language, writes config, generates `types_dir/*<file_ext>` files using `validation_lib` idioms
- **mcd-skeleton (Phase 2)** — uses `src_dir`, `file_ext`, `stub_body`, `comment_prefix` to write stubs
- **mcd-audit (Phase 3)** — uses `test_framework` idioms and `test_dir`, `test_ext` for oracle tests
- **mcd-fill (Phase 4)** — no config reads; works from the skeleton (language-neutral)
- **mcd-verify (Phase 5)** — uses `commands.{lint,typecheck,test}` verbatim
