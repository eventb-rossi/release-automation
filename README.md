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
net and to cover the third-party upstreams (ProB, Atelier B, TLC4B, …) that can't
notify us.

Producers, and the `tool` each one sends:

| Repository | Workflow | `tool` |
| --- | --- | --- |
| [`rossi`](https://github.com/eventb-rossi/rossi) | `release.yml` | `rossi` |
| [`Rodin-Bundle`](https://github.com/eventb-rossi/Rodin-Bundle) | `release.yml` | `rodin` |
| [`rodin-headless`](https://github.com/eventb-rossi/rodin-headless) | `release.yml` | `rodin-headless` |
| [`eventb-animate`](https://github.com/eventb-rossi/eventb-animate) | `release.yml` | `eventb-animate` |
| [`eventb-checker`](https://github.com/eventb-rossi/eventb-checker) | `release.yml` | `eventb-checker` |
| [`eventb-to-txt`](https://github.com/eventb-rossi/eventb-to-txt) | `deploy.yml` | `eventb-to-txt` |

The packaging repos ship Rodin as the `rodin` package built from Rodin-Bundle,
not vanilla Rodin, which is why its `tool` is `rodin`. Rodin-Bundle also notifies
for `-rcN` candidates, because `rodin-rc` follows the newest release or candidate.

### The website is a special case

[`eventb-rossi.org`](https://github.com/eventb-rossi/eventb-rossi.org) also
listens for `upstream-release` — it redeploys to refresh the version it bakes in
at build time. But it shows the **Rossi** version and nothing else, so it is
notified by a separate `notify-site` job guarded with `if: inputs.tool == 'rossi'`,
not by the packaging matrix. An `eventb-animate` or `rodin-headless` release must
not rebuild the site. The site re-checks `client_payload.tool` on its own side as
well, since `repository_dispatch` accepts a payload from any `Contents: write`
token; the guard here just keeps no-op runs out of its Actions tab.

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

`dispatch-token` must be a PAT with `Contents: write` on the downstream repos —
the packaging repos and `eventb-rossi.org` (`POST /repos/{owner}/{repo}/dispatches`
requires it). The shared org secret `VERSION_BUMP_TOKEN` already qualifies and must
be visible to the tool repos.

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

A consumer that cares about only one upstream should say so in the job, rather
than trusting the sender — `repository_dispatch` is postable by anything holding a
`Contents: write` token. Keep the `event_name` check or manual runs get skipped too:

```yaml
    if: ${{ github.event_name != 'repository_dispatch' || github.event.client_payload.tool == 'rossi' }}
```

