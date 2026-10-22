# `$/` fails on annotated tags

Same commit. Same code. One call works. One call fails.

```yaml
uses: stefanpenner-cs/tagref-callee/.github/workflows/a-self.yaml@lightweight   # ✅ works
uses: stefanpenner-cs/tagref-callee/.github/workflows/a-self.yaml@annotated     # ❌ startup_failure
```

Both tags point to the same commit. `a-self.yaml` calls `b.yaml` with `$/`.

## The two tags

```text
lightweight ───────────────────────────> commit d25c374
annotated ──> tag object ──────────────> commit d25c374
```

- A **lightweight tag** is a name for a commit.
- An **annotated tag** is a name for a tag object. The tag object points to the commit.

## What fails

```text
caller
  └─ uses a-self.yaml@<ref>
       └─ uses $/.github/workflows/b.yaml
```

| `<ref>` | Result | Run |
|---|---|---|
| `lightweight` | ✅ success | [39832716859](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832716859) |
| `main` (branch) | ✅ success | [39832732378](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832732378) |
| `annotated` | ❌ **startup_failure**, 0 jobs, no error text | [39832724653](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832724653) |

## Control: remove `$/`

`a-full.yaml` calls `b.yaml` with a full path. Nothing else changes.

| `<ref>` | Result | Run |
|---|---|---|
| `lightweight` | ✅ success | [39832739244](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832739244) |
| `annotated` | ✅ success | [39832746109](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832746109) |

The annotated tag works without `$/`. So `$/` is the trigger.

## Why it matters

- Floating tags such as `v1` are annotated (`git tag -fa`).
- Version tags such as `v1.3.358` are lightweight.
- `workflow.yaml@v1.3.358` works. `workflow.yaml@v1` fails. The commit is the same.
- Tests that call by branch do not catch this.

## Reproduce

1. Open [tagref-caller](https://github.com/stefanpenner-cs/tagref-caller/actions).
2. Run `light-self`, `annotated-self` and `branch-self` with `workflow_dispatch`.
3. Compare. Only `annotated-self` fails.

To rebuild from scratch:

```bash
git tag lightweight
git tag -a annotated -m annotated
git push origin main lightweight annotated
```

## Known and unknown

- **Known:** the tag type alone changes the result. `$/` is the trigger.
- **Unknown:** why. The tag object SHA may be used where a commit SHA is needed. This is a guess.

## Workaround

Use lightweight floating tags, or use a full `owner/repo/path@ref` in nested `uses:`.

## Files

| File | Purpose |
|---|---|
| `.github/workflows/b.yaml` | Leaf workflow. Prints a line. |
| `.github/workflows/a-self.yaml` | Calls `b.yaml` with `$/`. |
| `.github/workflows/a-full.yaml` | Control. Calls `b.yaml` by full path at `@main`. |
