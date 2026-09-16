# pnpm dedupe requires two passes to reach a fixed point

This repository is a minimal reproduction for a `pnpm dedupe` instability in pnpm 11.x.
It uses only public packages from the npm registry and pins pnpm 11.25.0 through the
`packageManager` field.

The starting dependency graph is:

```text
playground-pnpm-dedupe-instability-bug
├── chalk@4.1.0
└── boxen@5.1.2
    └── chalk@^4.1.0
```

After the root dependency is bumped to `chalk@^4.1.2`, one dedupe pass updates the root
dependency but leaves `boxen` on `chalk@4.1.0`. A second dedupe pass removes the duplicate.

The expected invariant is that `pnpm dedupe` reaches a fixed point in one run, so
`pnpm dedupe && pnpm dedupe --check` should succeed.

To test another pnpm version, temporarily update the `packageManager` field before running the
steps below.

## Reproduction

1. Install the committed baseline:

   ```bash
   pnpm install --frozen-lockfile
   ```

2. Bump the direct dependency to a newer range without updating the lockfile first:

   ```bash
   pnpm pkg set 'dependencies.chalk=^4.1.2'
   ```

3. Run one dedupe pass:

   ```bash
   pnpm dedupe
   ```

4. Check whether the lockfile is fully deduplicated:

   ```bash
   pnpm dedupe --check
   ```

   This exits with code 1 and reports:

   ```text
   [ERR_PNPM_DEDUPE_CHECK_ISSUES] Dedupe --check found changes to the lockfile

   Packages
   boxen@5.1.2
   └── chalk 4.1.0 → 4.1.2

   - chalk@4.1.0

   Run pnpm dedupe to apply the changes above.
   ```

5. Run dedupe a second time:

   ```bash
   pnpm dedupe
   ```

6. Check again:

   ```bash
   pnpm dedupe --check
   ```

   The second check succeeds. The same package manifests required two consecutive dedupe
   passes for the lockfile to reach a fixed point.

The important starting condition is that `chalk@4.1.2` is not present in the committed
lockfile. On the first pass, the old lockfile version remains strongly preferred for `boxen`'s
range while the bumped direct dependency introduces the new version. On the second pass, both
versions are already represented in the lockfile's preferred versions, and pnpm selects the
higher compatible version.

In pnpm 11, this behavior comes from the lockfile preferred-version seed used during dedupe:
existing lockfile versions receive `EXISTING_VERSION_SELECTOR_WEIGHT`, while the newly resolved
direct version initially has only the much smaller direct-dependency preference. After the
first pass writes the new version to the lockfile, the next pass treats both versions as
existing and can converge on the higher compatible version.

## Reset

To return to the committed baseline:

```bash
git restore package.json pnpm-lock.yaml
pnpm install --frozen-lockfile
```

## Affected Versions

| pnpm versions | Result |
|---|---:|
| 8.0.0–8.15.9 | ✅ No repro |
| 9.0.0–9.15.9 | ✅ No repro |
| 10.0.0–10.34.5 | ✅ No repro |
| 11.0.0–11.27.0 | ❌ Repro |
| 12.0.0–12.4.2 | ✅ No repro |
