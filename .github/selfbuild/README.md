# AirSync+ self-build

`.github/workflows/airsync-plus-build.yml` watches
[sameerasw/airsync-mac](https://github.com/sameerasw/airsync-mac) and builds every
new commit on its `main` branch and every new upstream release. It uses upstream's
own **AirSync Self Compiled** scheme, which compiles with `SELF_COMPILED` and so
turns on AirSync+. Each build is published as a release in a **private** repository.

The project's LICENSE (Additional Terms) and EULA allow self-compiled builds for
private use only, so the workflow refuses to publish to a public repository. This
fork is public, so its own Releases page is not used.

## How it works

1. **detect** (Linux, ~15 s, every 20 min): reads upstream's branch head and latest
   release, then skips any commit that already has a release in `RELEASE_REPO`.
   If upstream tags a commit that was already built, that release is promoted to
   *Latest* without rebuilding.
2. **build** (macOS): checks out the upstream commit, points Sparkle at the empty
   feed in this folder (otherwise the app offers to "update" to the official build,
   which has AirSync+ off), builds, signs ad hoc and packages `AirSync.dmg`.
3. **publish**: creates release `v<version>-<sha7>` in `RELEASE_REPO`. Builds of
   upstream releases are marked *Latest*; branch-head builds are pre-releases, and
   only the newest 15 are kept.

If a commit fails to compile, it is not retried on the next poll. To retry it, run
the workflow manually with **force** ticked.

## Setup

1. **Create the private release repo**, e.g. `Tekkiech/airsync-builds`, and tick
   *Add a README* (a release needs at least one commit to attach its tag to).
2. **Create a fine-grained personal access token**
   (Settings → Developer settings → Fine-grained tokens), limited to that repo only,
   with **Contents: Read and write**.
3. In **this fork** go to Settings → Secrets and variables → Actions:
   - Secret `RELEASE_TOKEN`: the token from step 2.
   - Variable `RELEASE_REPO`: `Tekkiech/airsync-builds`.
4. **Enable Actions on the fork.** GitHub disables workflows on new forks, so open
   the Actions tab and click *I understand my workflows, go ahead and enable them*.
5. Merge this workflow into the fork's default branch. Scheduled runs and the
   *Run workflow* button only work from there.
6. Run **AirSync+ self-build** once from the Actions tab to check everything.

Optional: disable upstream's own `Swift Build and Debug Release` and
`Telegram Notify` workflows in the Actions tab (⋯ → *Disable workflow*). They need
upstream's signing and Telegram secrets, so in a fork they just fail.

### Optional settings

| Name | Kind | Default | Purpose |
|---|---|---|---|
| `NTFY_URL` | secret | — | ntfy topic URL (`https://ntfy.sh/<random-topic>` or self-hosted) for build and failure push notifications |
| `NTFY_TOKEN` | secret | — | Access token, if the ntfy topic is protected |
| `SYNC_FORK` | variable | off | `true` also fast-forwards/merges this fork from upstream on each poll |
| `UPSTREAM_REPO` | variable | `sameerasw/airsync-mac` | Repository to build |
| `UPSTREAM_BRANCH` | variable | `main` | Branch to follow |
| `MACOS_RUNNER` | variable | `macos-26` | Runner label. The project needs the Xcode 26 SDK |
| `XCODE_VERSION` | variable | `latest-stable` | Passed to `maxim-lobanov/setup-xcode` |
| `KEEP_PRERELEASES` | variable | `15` | How many branch-head builds to keep |
| `SPARKLE_FEED_URL` | variable | this folder's `appcast.xml` | Sparkle feed baked into the app |

## Installing a build

Download `AirSync.dmg` from the private repo's releases and drag the app to
Applications. The build is ad-hoc signed, not notarized, so clear the quarantine
flag once:

```sh
xattr -dr com.apple.quarantine /Applications/AirSync.app
```

From a terminal, `gh release download -R Tekkiech/airsync-builds -p AirSync.dmg`
fetches the latest one.

Known limitation: without an Apple Developer team, the widget's app-group
entitlement can't be provisioned, so the desktop widget may not load. The app
itself is unaffected.
