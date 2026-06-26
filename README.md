# uv transitive OSV remediation — repro

Minimal reproduction for [renovatebot/renovate PR — surface `uv.lock` transitive deps for `osvVulnerabilityAlerts`](https://github.com/renovatebot/renovate/pull/PR_NUMBER).

## Scenario

- `pyproject.toml` declares a single, non-vulnerable direct dependency: `requests`.
- `requests` pulls in `idna` transitively. `idna` is **not** a declared dependency — it exists only in `uv.lock`.
- `uv.lock` deliberately carries an outdated transitive `idna` 3.6 — a version with a known advisory (`GHSA-jjg7-2v4v-x38h` / CVE-2024-3651, fixed in 3.7). It was produced with `uv lock --upgrade-package idna==3.6`; nothing pins it in `pyproject.toml`, so Renovate is free to remediate it.

Before this PR, Renovate's uv manager discards packages that live only in `uv.lock`, so `osvVulnerabilityAlerts` never sees `idna` and the vulnerability goes unreported.

## Reproduce

Run the PR branch against this directory using the [local platform](https://github.com/renovatebot/renovate/tree/main/lib/modules/platform/local) (dry run, no changes written):

```bash
# from a checkout of the PR branch
cd /path/to/this/repo
LOG_LEVEL=debug node /path/to/renovate/lib/renovate.ts --platform=local
```

`renovate.json` enables the feature:

```json
{ "osvVulnerabilityAlerts": true, "enabledManagers": ["pep621"] }
```

## Result

Renovate surfaces `idna` as a disabled `uv.lock` dependency, matches it against OSV, re-enables it, and proposes a lockfile update to the fixed version — while the non-vulnerable transitive deps stay disabled and silent.

```
DEBUG: Vulnerability GHSA-jjg7-2v4v-x38h affects idna 3.6
DEBUG: Setting allowed version >= 3.7 to fix vulnerability GHSA-jjg7-2v4v-x38h in idna 3.6
DEBUG: 1 flattened updates found: idna
DEBUG: ... branch=renovate/pypi-idna-vulnerability
```

The extracted `idna` dependency and its proposed update:

```jsonc
{
  "depName": "idna",
  "depType": "uv.lock",       // surfaced from the lockfile by this PR
  "lockedVersion": "3.6",
  "enabled": false,            // disabled by default — no routine updates
  "updates": [
    {
      "newVersion": "3.15",    // re-enabled only because of a fixable advisory
      "updateType": "minor",
      "isLockfileUpdate": true,
      "branchName": "renovate/pypi-idna-vulnerability"
    }
  ]
}
```

`certifi` and `charset-normalizer` (also transitive, but not vulnerable) appear as `enabled: false` with `updates: []` and `skipReason: "disabled"` — confirming the path is security-only and produces no routine-update noise.
