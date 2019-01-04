# Releasing Habitat

This document contains step-by-step details for how to release Habitat. All components are released
from the master branch on a bi-weekly schedule occurring every other Monday.

# Create an issue to track progress from the template

1. Create an issue which will track the progress of the release with [this template](https://github.com/habitat-sh/habitat/issues/new?template=release-checklist.md).
1. Check off the items in the list as you go.
1. If you make any changes to the release automation or documentation to include in the next release, you can mark those PRs as resolving the issue. Otherwise, just close it when the release is done.

# If your Release is going to cause downtime

1. Put a scheduled maintenance window into PagerDuty so the on-call doesn't go off.
1. Pre-announce outage on Twitter & Slack #general channel. There is no hard rule for a length of time you need to do this ahead of a release outage.
1. When your release begins, post announcement in our statuspage.io with outage information.
1. You are responsible for updating status changes in statuspage.io as the downtime proceeds. You are not responsible for regular minutes or responding in any other venue.
1. When the downtime is over, announce the end of the outage via statuspage.io. It will automatically post an announcement to #general and twitter.

## Releasing Launcher

The [`core/hab-launcher` package](https://bldr.habitat.sh/#/pkgs/core/hab-launcher), which contains
the `hab-launch` binary has a separate release process. See
[its README](components/launcher/README.md) for details. The Buildkite pipeline does not build a `core/hab-launcher` package; that must currently be built out-of-band and uploaded to Builder. The Buildkite pipeline _will_ however take care of _promoting_ it for you. The pipeline will guide you as appropriate.

## Prepare Master Branch for Release

1. Call for a "Freeze" on all merges to master and set the topic in #hab-team to indicate that it
is in effect

1. Clone the Habitat repository if you do not already have it

    ```
    $ git clone git@github.com:habitat-sh/habitat.git ~/habitat
    ```

1. Ensure you are on the master branch and have the latest of `~/habitat`:

    ```
    $ cd ~/habitat
    $ git checkout master
    $ git pull origin master
    ```

1. Create a new release branch in the Habitat repo. You can call this branch whatever you wish:

    ```
    $ cd ~/habitat
    $ git checkout -b <branch>
    ```

1. Remove the `-dev` suffix from the version number found in the `VERSION` file. *Note*: there must not be a space after the `-i`.

    ```
    $ sed -i'' -e 's/-dev//' VERSION
    ```
1. If necessary, fix up any issues with `CHANGELOG.md`, such as PRs that were missing the `X-` label and didn't get put in the correct category.

1. Commit `VERSION` changes and push your branch.
1. Issue a new PR with the `Expeditor: Exclude from Changelog` label and await approval (in the form of a [dank gif](http://imgur.com/X0sNq)) from two maintainers.
1. Pull master once again once the PR is merged into master.
1. Create & push a Git tag:

    ```
    $ make tag-release
    ```

If there are problems discovered during validation, or you need to modify the tag to include
additional commits, see [Addressing issues with a Release](#addressing-issues-with-a-release).

Once the release tag is pushed, Buildkite and AppVeyor builds will be triggered on the release tag. AppVeyor builds are currently very prone to timing out,
so set a 1-hour timer to go and check on them. If they do time out, you just have to restart them and hope. You may also want to set up [email notifications](https://ci.appveyor.com/notifications).

You can view/adminster Buildkite builds [here](https://buildkite.com/chef/habitat-sh-habitat-master-release).

The release tag builds will upload all release binaries to a channel named `rc-[VERSION]` and the `hab` cli will be uploaded but _not_ published to the `stable` Bintray repository. These builds can take about 45 minutes to fully complete. Keep an eye on them so we can validate the binaries when they finish.

## Create a Release Notes Blog Post PR

While you wait for the release automation to run, start working on the announcement.

If you haven't posted to the blog before, see [this article](https://github.com/habitat-sh/habitat/tree/master/www#how-to-add-a-blog-post) since there may be some setup required. Also, don't forget to build and test your post locally before merging it.

Create a "Release Notes" blog post file:
```
$ touch www/source/blog/$(date +%F)-$(sed -e 's/-dev//' ./VERSION)-release.html.md
```

Add the following content template to the file:

```
---
title: Habitat <VERSION> Released
date: <yyyy-mm-dd>
author: Tasha Drew
tags: release notes
category: product
classes: body-article
---

Habitat <VERSION> Release Notes

We are happy to announce the release of Habitat <VERSION>. We have a number of new features as well
as bug fixes, so please read on for all the details. If you just want the binaries, head on over to
[Install Habitat](https://www.habitat.sh/docs/using-habitat/#install-habitat).

Thanks again for using Habitat!

Highlights:
* [BREAKING CHANGE] SOME BREAKING CHANGE
```

For the subsequent body, start with the [autogenerated CHANGELOG](https://github.com/habitat-sh/habitat/blob/master/CHANGELOG.md) between `<!-- latest_release unreleased -->` and `<!-- latest_release -->`. Look through the merged pull requests and see if any of them is marked with the `X-change` label. If they are, make sure to highlight them as appropriate according to the seriousness of the change.

If you happen to be aware of features or bugs and feel there are user impacting words that need to be expressed, this is your opportunity to express them.

Create a PR and ask all team members (especially those with changes in the release) to review the google doc and

1. Add in any details to the changelog for any changes they made
2. Add a sentence or two explaining how the change was tested/validated

Don't actually merge the PR until the release is complete.

## Build the Windows Docker Studio image

Until this is integrated into Builfkite, it needs to be perfoemed manually on a Windows Docker host after Buildkite has uploaded all of the release candidate binaries:

```
$env:BINTRAY_USER="YOUR_USER_NAME"
$env:BINTRAY_KEY="YOUR_API_KEY"
$env:HAB_BLDR_CHANNEL="RC_CHANNEL"
hab pkg exec core/hab-bintray-publish publish-studio
```

This will build the Docker Studio image for Windows and push it to our bintray registry.

## Validate the Release

For each platform ([darwin](https://bintray.com/habitat/stable/hab-x86_64-darwin), [linux](https://bintray.com/habitat/stable/hab-x86_64-linux), [linux-kernel2](https://bintray.com/habitat/stable/hab-x86_64-linux-kernel2), [windows](https://bintray.com/habitat/stable/hab-x86_64-windows)), download the latest stable cli version from [Bintray](https://bintray.com/habitat/stable) (you will need to be signed into Bintray and a member of the "Habitat" organization). These can be downloaded from the version files page but are unpublished so that our download page does not yet include them. There may be special behavior related to this release that you will want to validate but at the very least, do the following basic tests.

You need to set `HAB_INTERNAL_BLDR_CHANNEL` and `CI_OVERRIDE_CHANNEL` to the name of the release channel (you _may_ also need to set `HAB_STUDIO_SECRET_HAB_INTERNAL_BLDR_CHANNEL` and `HAB_STUDIO_SECRET_CI_OVERRIDE_CHANNEL` for non-Docker-based studio). If a new Launcher is in the release channel, you should be fine; however, since that should be rare, you may have some additional work.

NOTE: If you are running `sudo hab studio enter` with all the required environmental variables set, but it's still telling you that it cannot find the package in stable, try `sudo -E hab studio enter`.

In a previous release, we were able to validate things on Linux by re-using a chroot studio and installing a Launcher out-of-band. You can probably create a new studio, enter it with `HAB_STUDIO_SUP=false`, manually install the latest stable Launcher (if a new one isn't part of the current release), exit the studio, then re-enter with `HAB_STUDIO_SUP` unset (but with all the override variables mentioned above set). This should reuse the Launcher you just installed, but pull in additional artifacts as needed from your release channel.

See https://github.com/habitat-sh/habitat/issues/4656 for further context and ideas.

Then you can actually exercise the software as follows:

1. It pulls down the correct studio image
1. That studio's `hab` is at the correct version (`hab --version`)
1. A `sup-log` shows a running supervisor (if `sup-log` does not show a supervisor running, run `hab install core/hab-sup --channel release_channel` then `hab sup run`)
1. Verify that the supervisor is the correct version (`hab sup --version`)

When testing the linux studio, you will need to `export CI_OVERRIDE_CHANNEL` to the rc channel of the release. So if you are releasing 0.75.2, the channel would be `rc-0.75.2`.

### Validating x86_64-linux-kernel2

For this PackageTarget it is important that you perform validation on a Linux system running a 2.6 series kernel. CentOS 6 is recommended because it ships with a kernel of the appropriate age,  but any distro with a Kernel between 2.6.32 and 3.0.0 can be used. Included in the `support/validation/x86_64-linux-kernel2` directory in this repository is a Vagrantfile that will create a CentOS-6 VM to perform the validation. You can also run a VM in EC2.

### Addressing issues with a Release

If you find issues when validating the release binaries that must be fixed before promoting the release, you will need to fix those issues and then have Buildkite and AppVeyor rerun the deployment. After you merge the necessary PRs to fix the release issues:

```
     $ make re-tag-release
```

# Post-Release Tasks
The Buildkite release is fairly-well automated at this point, but once it is complete, there are still a few remaining manual tasks to perform. In time, these will be automated as well.

## Update Builder Bootstrap Bundle

Once the Buildkite linux deployment has completed, we generate a release bundle of all Habitat and Builder components which are uploaded to an S3 bucket which we read from when we bootstrap new nodes. This bundle is useful if you are bootstrapping in an environment which doesn't have access to Builder or there simply isn't a Builder instance in existence (ah, those were the days).

NOTE: Do this step from either a Linux VM or in a studio.

1. Configure your AWS credentials in your environment
1. Execute the script that currently lives in the [builder](https://github.com/habitat-sh/builder) repository:

    ```
    $ cd /path/to/builder-repo
    $ sudo AWS_ACCESS_KEY_ID=<...> AWS_SECRET_ACCESS_KEY=<...> terraform/scripts/create_bootstrap_bundle.sh <HABITAT_VERSION>'
    ```

## Update Homebrew Tap

This should be automatically handled by Buildkite. You can find manual instructions in [a previous version of this file](https://github.com/habitat-sh/habitat/blob/267d31f03a00dfa3b1b8e0ba00c20efa4913a7a8/RELEASE.md).

Validate the update by running `brew update hab` on Mac OS X and checking the version is correct.

## Push Chocolatey package

Until Buildkite integrates the Chocolatey package creation and upload, we need to run `support/ci/choco_push.ps1` from a Windows machine that has the `choco` cli installed.

The `choco` cli can be installed via:
```
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

Now run:
```
.\support\ci\choco_push.ps1 -Version [VERSION] -Release [RELEASE] -ApiKey [CHOCO_API_KEY] -Checksum [BINTRAY_PUBLISHED_CHECKSUM]
```

`CHOCO_API_KEY` can be retrieved from 1password. `BINTRAY_PUBLISHED_CHECKSUM` should be the checksum in `windows.zip.sha256sum` file uploaded to bintray.


## Publish Release

Create release in [GitHub](https://github.com/habitat-sh/habitat/releases)

On the GitHub releases page, there should already be a tag for the release (pushed up previously).
Draft a new Release, specify the tag, and title it with the same (eg, 0.18.0). Then hit Publish Release.

Merge the release notes blog post PR now. The live blog will be automatically updated, but check
to make sure it is successful and looks correct.

## Update the Acceptance environment with the new hab-backline

After the new hab is released and there is a new hab-backline in stable, the acceptance environment will also need to be updated. In order to do this, (from a Linux machine):

```
./update-hab-backline.sh stable $(< VERSION)
```

If your auth token isn't specified in your environment, you can add `-z <AUTH_TOKEN>`
(or any other arguments to pass to the `hab pkg upload` command) to the
`update-hab-backline.sh` script after the channel and version arguments.

Make sure the commands from the trace output look correct when the script executes:
1. The version is the version being released (no `-dev` suffix)
1. The install is from the `stable` channel
1. The upload is to the `stable` channel

## Update the Docs

Assuming you've got a locally installed version of the `hab` CLI you just released, you can update the CLI documentation in a separate PR. To do that run the following commands on OS X (other platforms may work as well):

```
cd www
make cli_docs
make template_reference
```

Verify the diff looks reasonable and matches the newly released version, then submit your PR.

# Drink. It. In.

## Bump Version

1. Update the version number found in the `VERSION` file to the next target release and append the `-dev` suffix to that number
1. Issue a PR and merge it yourself

> Example: If the release version was `0.9.0` then the contents of `VERSION` might read `0.10.0-dev` if your next target is `0.10.0`.

## Update the Acceptance environment with the new hab-backline

Now there's yet another new version of hab-backline, in unstable. So off to the races again for acceptance. This time you'll install the hab backline from _unstable_ channel.  The other steps are the same:

```
./update-hab-backline.sh unstable $(< VERSION)
```

Make sure the commands from the trace output look correct when the script executes:
1. The version is the new dev version after the one we just released; there should be a `-dev` suffix
1. The install is from the `unstable` channel
1. The upload is to the `stable` channel

## Promote the Builder Worker

Now that the release is stable, it is time to promote the workers.

In the Builder Web UI, go to the builds page for [`habitat/builder-worker`](https://bldr.habitat.sh/#/pkgs/habitat/builder-worker/latest) and check that the latest version is a successful recent build.

Promote the release. Wait for a few minutes, then perform a test build. Check the build log for the test build to confirm that the version of the Habitat client being used is the desired version.

# Release Notification

1. Create new posts in [Habitat Announcements](https://discourse.chef.io/c/habitat) on the Chef discourse as well as [Announcements](https://forums.habitat.sh/c/announcements) in the Habitat forums.
1. Link forum posts to the github release
1. Link github release to forum post
1. Tweet a link to the announcement @habitatsh (credentials in [1password](https://team-habitat.1password.com))
1. Make sure the blog post PR with the release announcements is merged and published.
1. Announce that the "Freeze" on merges to master is lifted in both the Chef internal slack team and in the Habitat slack team.

# Update Cargo.lock

1. In the [habitat](https://github.com/habitat-sh/habitat) repo, run `cargo update`, `cargo check --all --tests`.
1. If there are warnings or errors that are simple, fix them. Otherwise, lock the appropriate versions in `Cargo.toml` files that lets the build succeed and file an issue to resolve the failure and relax the version lock.
1. Open a PR for the `Cargo.lock` updates and any accompanying fixes which are necessary.
1. Repeat with the [core](https://github.com/habitat-sh/core) and [builder](https://github.com/habitat-sh/builder) repos (omit the `habitat-launcher` build).
