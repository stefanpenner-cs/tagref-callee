# `$/` fails on annotated tags

`$/` in a reusable workflow `uses:` fails with `startup_failure` when the caller pins an **annotated tag**.
A lightweight tag and a branch work.

## Result

Same commit. Same files. Only the tag type changes.

```text
caller ──uses──> a-self.yaml@<ref> ──uses──> $/.github/workflows/b.yaml
```

| Ref type | `a-self.yaml` (uses `$/`) | `a-full.yaml` (full path) |
|---|---|---|
| Lightweight tag | [success](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832716859) | [success](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832739244) |
| Annotated tag | [**startup_failure**](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832724653) | [success](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832746109) |
| Branch (`main`) | [success](https://github.com/stefanpenner-cs/tagref-caller/actions/runs/39832732378) | n/a |

The failed run has 0 jobs and no error text.

## Why it matters

- Floating tags such as `v1` are annotated. Tools that run `git tag -fa` make annotated tags.
- Version tags such as `v1.3.358` are lightweight.
- Calling `workflow.yaml@v1.3.358` works. Calling `workflow.yaml@v1` fails. The commit is the same.
- Tests that call by branch do not catch this.

## Reproduce

1. Commit `.github/workflows/{a-self,a-full,b}.yaml` in this repo.
2. Create two tags on one commit:
   ```bash
   git tag lightweight
   git tag -a annotated -m annotated
   git push origin main lightweight annotated
   ```
3. In [tagref-caller](https://github.com/stefanpenner-cs/tagref-caller), run each workflow with `workflow_dispatch`.
4. Compare the results with the table.

## Known and unknown

- **Known:** the tag type alone changes the result. `$/` is the trigger.
- **Unknown:** why. The tag object SHA may be used where a commit SHA is needed. This is a guess.

## Workaround

Use a full `owner/repo/path@ref` in `uses:` for nested calls, or use lightweight floating tags.

## Files

| File | Purpose |
|---|---|
| `.github/workflows/b.yaml` | Leaf workflow. Prints a line. |
| `.github/workflows/a-self.yaml` | Calls `b.yaml` with `$/`. |
| `.github/workflows/a-full.yaml` | Control. Calls `b.yaml` by full path at `@main`. |
