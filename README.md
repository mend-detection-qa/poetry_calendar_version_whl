# poetry_calendar_version_whl

> **Regression test repo for [SCA-5540](https://mend-io.atlassian.net/browse/SCA-5540)**

## What this tests

`AbstractPythonDependencyResolver.versionPatternWhl` previously required at least one dot in the version segment of a wheel filename. Packages that ship with a **calendar version** (`YYYYMMDD`, e.g. `pdfminer.six==20251230`) or a **single-integer version** (e.g. `pkg==1`) failed the match, causing:

1. `addDependencyInfoData()` to extract an empty artifact name and empty version.
2. `PoetryDependencyResolver.processDependencyInfos()` to discard the resulting `DependencyInfo` because `normalizeDepKey("", ...)` returns `null`.
3. The resolver to fall through to `getDefaultDepInfo()` which emits a stripped record — **no `sha1`/`systemPath`/`filename`/`dependencyFile`**.

This repo contains a Poetry project that directly depends on `pdfplumber`, which **transitively** pulls in `pdfminer.six==20251230` — the canonical repro case.

## Regression signal

The `autotest_config.json` enables these checks:

| Check | What it catches |
|---|---|
| `transitive_dependencies_names_list` contains `pdfminer-six` | Package must be present in the resolved tree |
| `check_sha1_not_empty` | `sha1` field must be non-empty (was `""` before the fix) |
| `check_sha1_format` | `sha1` must be a valid 40-char lowercase hex string |
| `check_sha1_consistency` | `dep.sha1` must match `dep.checksums.SHA1` |
| `check_update_request_sca_results` | No ERROR-level SCA results — resolver must not throw on the whl lookup |

If any of these checks fail, `pdfminer.six` is being resolved as a stripped record — the regression has re-appeared.

## Package structure

```
pyproject.toml        — direct dep: pdfplumber ^0.11
poetry.lock           — pinned: pdfminer.six 20251230, pdfplumber 0.11.4, pillow 11.2.1
authotest_config.json — validation config with SHA1 presence checks enabled
expected-tree.json    — expected dep tree structure
.whitesource          — Mend scan configuration
```

## Jira

[SCA-5540 — Unified agent fails to find relevant whl files in .venv cache for edge case](https://mend-io.atlassian.net/browse/SCA-5540)
