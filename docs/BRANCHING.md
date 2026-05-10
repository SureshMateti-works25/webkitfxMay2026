# Branch workflow

| Branch | Role |
|--------|------|
| `master` | Released / stable documentation snapshots |
| `develop` | Integration line for doc updates before release |
| `feature/*` | Short-lived work; open PRs into `develop` |

Default integration target for changes is **`develop`**. Merge `develop` into `master` when you cut a doc release.
