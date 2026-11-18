# launchset action

Changesets-style releases for lock-step monorepos: one version drives many targets (Docker, Android, iOS, web). The action decides what to release, writes version and build numbers into your files and records what shipped as git tags. It does not build anything: your own workflow steps do that.

Documentation: <https://demonsters.github.io/launchset/>

This repository only holds the compiled action (`action.yml` and `dist/index.js`). It is published automatically with every launchset release. Don't send changes here.

## Usage

The easiest way to get the workflows is `npx @demonsters/launchset init`, which writes them for your repo. They call the action like this:

```yaml
jobs:
  launchset:
    runs-on: ubuntu-latest
    outputs:
      mode: ${{ steps.launchset.outputs.mode }}
      matrix: ${{ steps.launchset.outputs.matrix }}
      plan: ${{ steps.launchset.outputs.plan }}
    steps:
      - id: app-token
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ secrets.LAUNCHSET_APP_ID }}
          private-key: ${{ secrets.LAUNCHSET_APP_PRIVATE_KEY }}
      - uses: actions/checkout@v5
        with:
          token: ${{ steps.app-token.outputs.token }}
          fetch-depth: 0
      - id: launchset
        uses: demonsters/launchset-action@v1
        with:
          command: release
          token: ${{ steps.app-token.outputs.token }}
```

Later jobs pass the `plan` output back in, so nothing is computed twice:

```yaml
- uses: demonsters/launchset-action@v1
  with:
    command: stamp
    target: ${{ matrix.target }}
    plan: ${{ needs.launchset.outputs.plan }}
```

Commands: `check`, `release`, `stamp`, `verify`, `mark`, `finalize`, `preview-plan`, `preview-comment` and `add`. Every command writes a summary to the job summary.

Releases need a GitHub App token rather than `GITHUB_TOKEN`, because events caused by `GITHUB_TOKEN` don't trigger other workflows. The `check` command only reads and comments, so the default `GITHUB_TOKEN` is enough there. See the documentation for how to create the App.

The action runs on the `node24` runtime and needs `git` on the runner (every GitHub-hosted runner has it). Checkouts need `fetch-depth: 0` for `check`, `release`, `finalize` and the preview commands.

## Inputs

<!-- inputs:start -->

| Input | Description | Default |
| --- | --- | --- |
| `command` | What to run. One of: check (PR check and sticky comment), release (decide the release mode: publish, ship, release-pr or none), stamp (write version and build number into a target's files), verify (store, registry or npm check before uploading), mark (record that a target shipped), finalize (create the release tag and GitHub Release when every auto target has shipped), preview-plan (plan a preview or prerelease build), preview-comment (sticky PR comment and prerelease GitHub Release), add (write a changeset from a workflow). | required |
| `token` | GitHub token. For check, the default (github.token) is enough: it only reads and comments. For release, mark, finalize, preview-plan and preview-comment pass a GitHub App installation token (actions/create-github-app-token), because events caused by GITHUB_TOKEN do not trigger other workflows. | `${{ github.token }}` |
| `target` | Target name. Required for stamp, verify and mark. With release: the manual target to ship (shipping it for an already released version). With preview-plan: an optional comma-separated list of targets that overrides the configured preview targets (used by workflow_dispatch). |  |
| `bump` | With add: the bump type of the new changeset (major, minor, patch or none). |  |
| `message` | With add: the changeset summary (the changelog entry). |  |
| `version` | With release and target: the already released version to ship (default: the latest release on the branch the workflow was started from). |  |
| `plan` | The plan JSON from the release or preview-plan job (its plan output). stamp, verify, mark, finalize and preview-comment read it and never recompute the version or build number. |  |
| `working-directory` | Directory of the repository root to run in (where launchset.config.* lives). | `.` |

<!-- inputs:end -->

## Outputs

Which outputs a command sets is listed in the description. Outputs that don't apply are empty or not set.

<!-- outputs:start -->

| Output | Description |
| --- | --- |
| `mode` | Set by release. publish (build and ship the release), ship (build one manual target), release-pr (the release PR was created or updated) or none (nothing to do). |
| `matrix` | Set by release and preview-plan. JSON array of target names to build, for fromJSON() in a build matrix. Manual targets are only in it in ship mode. |
| `plan` | Set by release and preview-plan. The plan as JSON (version, build number, channel, release sha and per-target data). Pass it to the later commands as the plan input. |
| `version` | Set by release, preview-plan, stamp and verify. The version of this run: the release version, the preview version or the prerelease version. |
| `build-number` | Set by release, preview-plan and stamp. The build number of this run. Empty with the eas strategy. |
| `channel` | Set by release, preview-plan and stamp. release, prerelease or preview. Use it to pick the store track, TestFlight group or npm tag. |
| `release-sha` | Set by release and preview-plan. The commit every build job must check out (the release commit, or the commit that was built for a preview). |
| `already-published` | Set by verify. "true" when this exact version and build number is already in the store, registry or npm. The workflow skips the build and upload steps and goes straight to mark. |
| `docker-tags` | Set by stamp for docker targets. Full image references (image:tag), one per line. Pass it to docker/build-push-action as tags. |
| `docker-labels` | Set by stamp for docker targets. OCI labels (org.opencontainers.image.version, .revision, .created), one key=value per line. Pass it to docker/build-push-action as labels. |
| `docker-refs-json` | Set by stamp for docker targets. The same references as docker-tags, as a JSON array of strings, for steps that need a structured list (for example scanners or deploy jobs). |
| `dist-tag` | Set by release, preview-plan and stamp. Suggested npm dist-tag: latest for base-branch releases, release-&lt;major&gt;.&lt;minor&gt; on maintenance branches, &lt;preid&gt; for prereleases and pr-&lt;n&gt; or &lt;branchSlug&gt; for previews. |
| `pr-number` | Set by check, and by preview-plan on pull_request runs. The number of the pull request, or the release PR number in release-pr mode. |
| `marketing-version` | Set by stamp. The version written into the target's marketing-version fields. It is the plain X.Y.Z base version for iOS targets (Apple does not accept prerelease suffixes) and the same as version for every other target. |
| `tag` | Set by release and preview-plan. The git tag of this version: the release tag in publish and ship mode, or the reserved prerelease tag on prerelease branches. Empty for other previews. |
| `prerelease` | Set by release, preview-plan and stamp. "true" when the channel is prerelease, otherwise "false". A convenient condition for steps that publish only prereleases. |
| `path` | Set by add. The path of the new changeset file, relative to the repository root (for example .launchset/brave-lions-sing.md). Use it to commit the file in a workflow. |
| `skipped` | Set by preview-plan. A short code (for example no-label or fork) when no preview is planned for this run; empty otherwise. When it is set, matrix is [] and plan is empty. |
| `skip-reason` | Set by preview-plan. The sentence that says why the preview was skipped. |
| `comment-url` | Set by preview-comment on pull request previews. The URL of the sticky comment. |
| `release-url` | Set by preview-comment on prerelease branches. The URL of the GitHub pre-release. |

<!-- outputs:end -->

## License

All rights reserved, demonsters. See [LICENSE](LICENSE).
