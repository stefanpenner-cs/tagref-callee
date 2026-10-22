# Annotated tag + `$/` reusable workflow ref gives `startup_failure`

Same commit. Same code. One ref works. One ref fails.

```yaml
uses: stefanpenner-cs/tagref-callee/.github/workflows/a-self.yaml@light-a  # ✅ works
uses: stefanpenner-cs/tagref-callee/.github/workflows/a-self.yaml@ann-a    # ❌ startup_failure
```

Both tags point to commit `9c4154b`. `a-self.yaml` calls `b.yaml` with `$/`.

## The two tag types

```text
light-a ───────────────────────────> commit 9c4154b
ann-a ──> tag object ──────────────> commit 9c4154b
```

- A **lightweight tag** names a commit.
- An **annotated tag** names a tag object. The tag object points to the commit.

## Results

```text
caller
  └─ uses a-self.yaml@<ref>
       └─ uses $/.github/workflows/b.yaml
```

n = number of runs. All runs are `workflow_dispatch` in a public repo.

| Caller ref | Tag type | Nested `uses:` | n | Result |
|---|---|---|---|---|
| `ann-a` | annotated | `$/` | 3 | ❌ startup_failure |
| `ann-b` | annotated | `$/` | 1 | ❌ startup_failure |
| `lightweight-named-ann` | annotated (lightweight-looking name) | `$/` | 1 | ❌ startup_failure |
| `light-a` | lightweight | `$/` | 3 | ✅ success |
| `light-b` | lightweight | `$/` | 1 | ✅ success |
| `annotated-named-lw` | lightweight (annotated-looking name) | `$/` | 1 | ✅ success |
| `ann-a` | annotated | full path to `@ann-a` (`a-full-ann`) | 1 | ✅ success |
| `light-a` | lightweight | full path to `@light-a` (`a-full-light`) | 1 | ✅ success |
| `ann-a` | annotated | full path to `@light-a` (`a-full-light`) | 1 | ✅ success |

- Tag type decides the result. The tag name does not.
- The full-path controls use the same tag type and pass. So `$/` is the trigger.
- A failed run has 0 jobs and no error text.

Every case uses commit `9c4154b`. Run IDs are in
[tagref-caller](https://github.com/stefanpenner-cs/tagref-caller/actions).

## Why it matters

- The incident: `v1` is an annotated tag. `v1.3.358` is a lightweight tag. Both name one commit.
- `@v1.3.358` worked. `@v1` failed.
- Tests that call by branch do not catch this.

## Reproduce

1. Open [tagref-caller](https://github.com/stefanpenner-cs/tagref-caller/actions).
2. Run the `m*` workflows with `workflow_dispatch`.
3. Check each result:

```bash
gh run list -R stefanpenner-cs/tagref-caller --json workflowName,conclusion
```

Rebuild the tags from scratch:

```bash
git tag light-a
git tag -a ann-a -m ann-a
git push origin main light-a ann-a
```

## Known and unknown

- **Known:** on this commit, annotated tag + `$/` fails. Lightweight tag + `$/` passes.
- **Unknown:** the cause. GitHub shows no error text.
- **Unknown:** public vs internal repos, `push` and PR events, deeper chains, force-moved tags.
- **Not tested:** the claim in the revert PR about how `$/` resolves.

## Workaround

Not proven. A full `owner/repo/path@ref` in nested `uses:` passed in the controls above.

## Files

| File | Purpose |
|---|---|
| `.github/workflows/b.yaml` | Leaf workflow. Prints a line. |
| `.github/workflows/a-self.yaml` | Calls `b.yaml` with `$/`. |
| `.github/workflows/a-full*.yaml` | Controls. Call `b.yaml` by full path. |
