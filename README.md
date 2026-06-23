# release-automation

Shared release-automation workflows for the
[eventb-rossi](https://github.com/eventb-rossi) organization.

## `notify-downstream`

A [reusable workflow](.github/workflows/notify-downstream.yml) that fans out a
`repository_dispatch` event (`event_type: upstream-release`) to every packaging
repo — [`apt`](https://github.com/eventb-rossi/apt),
[`gentoo-overlay`](https://github.com/eventb-rossi/gentoo-overlay),
[`homebrew-tap`](https://github.com/eventb-rossi/homebrew-tap),
[`scoop-eventb`](https://github.com/eventb-rossi/scoop-eventb) and
[`eventb-copr`](https://github.com/eventb-rossi/eventb-copr) — when an upstream
tool publishes a release.

Each packaging repo listens for that event and runs its version-check
immediately, instead of waiting for its daily cron. The cron stays as a safety
net and to cover the third-party upstreams (ProB, Rodin, TLC4B, …) that can't
notify us.

### Producer side

Call the workflow from the release workflow, after the publish/release step:

```yaml
jobs:
  notify-downstream:
    needs: [<publish-or-release job id>]
    uses: eventb-rossi/release-automation/.github/workflows/notify-downstream.yml@main
    with:
      tool: rossi                                     # the released tool
      version: ${{ github.event.release.tag_name }}   # or ${{ github.ref_name }}
    secrets:
      dispatch-token: ${{ secrets.VERSION_BUMP_TOKEN }}
```

`dispatch-token` must be a PAT with `Contents: write` on the packaging repos
(`POST /repos/{owner}/{repo}/dispatches` requires it). The shared org secret
`VERSION_BUMP_TOKEN` already qualifies and must be visible to the tool repos.

### Consumer side

Add the trigger to the existing scheduled workflow; nothing else changes:

```yaml
on:
  schedule:
    - cron: '...'             # kept as safety net + third-party coverage
  workflow_dispatch:
  repository_dispatch:
    types: [upstream-release]
```

