---
title: Making a Silverstripe CMS core release
summary: Development guide for core contributors to build and publish a new release
iconBrand: github-alt
---

# Making a Silverstripe CMS core release

## Introduction

This guide is intended to be followed by core contributors, allowing them to take
the latest development branch of each of the core modules, and building a release.
The artifacts for this process are typically:

 * A downloadable tar / zip on silverstripe.org
 * A published announcement
 * A new composer installable stable tag for silverstripe/installer

While this document is not normally applicable to normal silverstripe contributors,
it is still useful to have it available in a public location so that these users
are aware of these processes.

## Module releases

Occasionally a fix to an individual module warrants a patch release outside of the standard quarterly release cycle. Rather than generating a full new recipe release, the following process should be followed to perform a patch release for an individual module:

1. Check out the module locally
1. Determine the new module version to be released (e.g. current version is 4.6.1, therefore new version will be 4.6.2)
1. Generate a changelog using our standard changelog format and the current release + branch name:
   ```
   > git log --oneline --pretty=format:"* %s (%an) - %h" --no-merges 4.6.1...4.6
   ```
1. Draft a new Release in GitHub, targeting the correct branch, specifying the version to be released, and pasting the changelog in the description field.
1. Publish the release, and ensure the new version becomes visible in Packagist.

Projects wanting to pick up this individual patch will need to alias it in their `composer.json` file if they're using a recipe:

```
    "requirements": {
        "silverstripe/recipe-cms": "4.6.1",
        "silverstripe/cms": "4.6.2 as 4.6.1"
    }
```

## First time setup

As a core contributor it is necessary to have installed the following set of tools:

### First time setup: Standard releases

* PHP 5.6+
* Python 2.7 / 3.5
* [cow release tool](https://github.com/silverstripe/cow#install). This should typically
  be installed in a global location via the below command. Please see the installation
  docs on the cow repo for more setup details.
  `composer global require silverstripe/cow ^2`
* [satis repository tool](https://github.com/composer/satis). This should be installed
  globally for minimum maintenance.
  `composer global require composer/satis ^1`
* [transifex client](http://docs.transifex.com/client/).
  `pip install transifex-client`
  If you're on OSX 10.10+, the standard Python installer is locked down.
  Use `brew install python; sudo easy_install pip` instead
* [AWS CLI tools](https://aws.amazon.com/cli/):
  `pip install awscli`
* The `tar` and `zip` commands
* A good `.env` setup in your localhost webroot.

Example `.env`:

```
# Environment
SS_TRUSTED_PROXY_IPS="*"
SS_ENVIRONMENT_TYPE="dev"

# DB Credentials
SS_DATABASE_CLASS="MySQLDatabase"
SS_DATABASE_SERVER="127.0.0.1"
SS_DATABASE_USERNAME="root"
SS_DATABASE_PASSWORD=""

# Each release will have its own DB
SS_DATABASE_CHOOSE_NAME=1

# So you can test releases
SS_DEFAULT_ADMIN_USERNAME="admin"
SS_DEFAULT_ADMIN_PASSWORD="password"

# Basic CLI request url default
SS_BASE_URL="http://localhost/"
```

You will also need to be assigned the following permissions. Contact one of the SilverStripe staff from
the [core committers](/project_governance/core_committers), who will assist with setting up your credentials.

* Write permissions on the [silverstripe](https://github.com/silverstripe) organisation.
* Admin permissions on [transifex](https://www.transifex.com/silverstripe/).
  Set up a [~/.transifexrc](https://docs.transifex.com/client/client-configuration) with your credentials.
* AWS write permissions on the `silverstripe-ssorg-releases` s3 bucket
  ([add credentials](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) via `aws configure`).
* Permission on [silverstripe release announcement](https://groups.google.com/forum/#!forum/silverstripe-announce).
* Moderator permissions in the [Slack channel](https://www.silverstripe.org/community/slack-signup/)
* Administrator account on [docs.silverstripe.org](https://docs.silverstripe.org) and
  [userhelp.silverstripe.org](https://userhelp.silverstripe.org).

### First time setup: Security releases

For doing security releases the following additional setup tasks are necessary:

* Write permissions on the [silverstripe-security](https://github.com/silverstripe-security)
  organisation.
* Permissions to write to the [security releases page](http://www.silverstripe.org/download/security-releases)
  and the [silverstripe.org CMS](http://www.silverstripe.org/admin).
* Permission on [security pre-announcement mailing list](https://groups.google.com/a/silverstripe.com/forum/#!forum/security-preannounce).

## Standard release process

See [Release Process](release-process) for details on the standard timeline for releases.
In summary, we produce a beta release, stabilise, produce a release candidate, perform
penetration testing, and then produce a stable release.

When creating a new release, the following process is broken down into two
main sets of commands:

### Stage 1: Release preparation:

If you are managing a release, it's best to first make sure that SilverStripe marketing
are aware of any impending release. This is so that they can ensure that a relevant blog
post will appear on [www.silverstripe.org/blog](http://www.silverstripe.org/blog/), and
cross-posted to other relevant channels such as social media.
Blog posts should be prepared for each major, minor and security releases.
Patch releases, alphas, betas and release candidates usually don't need blog posts,
unless they're introducing important changes (e.g. for a new major release). 
Sending an email to [marketing@silverstripe.com](mailto:marketing@silverstripe.com)
with an overview of the release and a rough release timeline.

Check all tickets assigned to that milestone are either closed or reassigned to another milestone.
Use the [list of all issues across modules](https://www.silverstripe.org/community/contributing-to-silverstripe/github-all-core-issues)
as a starting point, and add a `milestone:"your-milestone"` filter.

Merge up from other older [supported release branches](release-process#supported-versions) (e.g. merge `4.0`->`4.1`, `4.1`->`4.2`, `4.2`->`4`, `4`->`master`).
Some core modules use major version `1` for their CMS 4 release line - this can
be considered interchangeable with `4`.

This is the part of the release that prepares and tests everything locally, but
doe not make any upstream changes (so it's safe to run without worrying about
any mistakes migrating their way into the public sphere).

Invoked by running `cow release` in the format as below:

`cow release <version> [recipe] -vvv`

E.g.

`cow release 4.0.1 -vvv`

* `<version>` The recipe version that is to be released. E.g. `4.1.4` or `4.3.0-rc1`
* `<recipe>` Optional: the recipe that is being released (default: "silverstripe/installer")

This command has these options (note that `--repository` option is critical for security releases):

* `-vvv` to ensure all underlying commands are echoed
* `--directory <directory>` to specify the folder to create or look for this project in. If you don't specify this,
it will install to the path specified by `./release-<version>` in the current directory.
* `--repository <repository>` will allow a custom composer package url to be specified. E.g. `http://packages.cwp.govt.nz`
  See the above section "Setting up satis for hosting private security releases" on how to prepare a custom
  repository for a security release.
* `--branching <type>` will specify a branching strategy. This allows these options:
  * `auto` - Default option, will branch to the minor version (e.g. 1.1) unless doing a non-stable tag (e.g. rc1)
  * `major` - Branch all repos to the major version (e.g. 1) unless already on a more-specific minor version.
  * `minor` - Branch all repos to the minor semver branch (e.g. 1.1)
  * `none` - Release from the current branch and do no branching.
* `--skip-tests` to skip tests
* `--skip-i18n` to skip updating localisations

This can take between 5-15 minutes, and will invoke the following steps,
each of which can also be run in isolation (in case the process stalls
and needs to be manually advanced):

* `release:create` The release version will be created in the `release-<version>`
  folder directly underneath the folder this command was invoked in. Cow
  will look at the available versions and branch-aliases of `silverstripe/installer`
  to determine the best version to install from. E.g. installing 4.0.0 will
  know to install dev-master, and installing 3.3.0 will install from 3.x-dev.
  If installing pre-release versions for stabilisation, it will use the correct
  temporary release branch.
* `release:plan` The release planning will take place, this reads the various dependencies of the recipe being released
  and determines what new versions of those dependencies need to be tagged to create the final release. Note that
  the patch version numbers of each module may differ. This step requires the latest versions to be released are
  determined and added to the plan. The conclusion of the planning step is output to the screen and requires user
  confirmation.
* `release:branch` If release:create installed from a non-rc branch, it will
  create the new temporary release branch (via `--branch-auto`). You can also customise this branch
  with `--branch=<branchname>`, but it's best to use the standard.
* `release:translate` All upstream transifex strings will be pulled into the
  local master strings, and then the [i18nTextCollector](api:SilverStripe\i18n\TextCollection\i18nTextCollector)
  task will be invoked and will merge these strings together, before pushing all
  new master strings back up to transifex to make them available for translation.
  Changes to these files will also be automatically committed to git.
* `release:test` Will run all unit tests on this release. Make sure that you
  setup your `.env` correctly (as above) so that this will work.
* `release:changelog` Will compare the current branch head with `--from` parameter
  version in order to generate a changelog file. This will be placed into the
  `./framework/docs/en/04_Changelogs/` folder. If an existing file named after
  this version is already in that location, then the changes will be automatically
  regenerated beneath the automatically added line:
  `<!--- Changes below this line will be automatically regenerated -->`.
  It may be necessary to edit this file to add details of any upgrading notes
  or special considerations. If this is a security release, make sure that any
  links to the security registrar (http://www.silverstripe.org/download/security-releases)
  match the pages saved in draft.

#### Basing a new release on a previous one (tweak releases)

Commonly a stable release will need to mirror the contents of the release
candidate that preceded it, sometimes with a small set of additional commits.
However, running the standard `cow release` command will create a release that
includes all the latest commits on the branches it targets, which can include
unaudited code. A **tweak release** includes only the commits present in the
previous tagged release by default, and can optionally include additional
commits when necessary. To create one, use the `release:detach-tagged-base`
command:

1. `cow release:create <new-version>` to create the new release.
2. `cow release:plan <new-version>` to generate a plan for the new release.
3. `cow release:detach-tagged-base <new-version>` to shift all of the modules
  to the correct commit in the branch to match the contents of the last release.
  * **How?** This command finds the last common commit between the latest tag on
    the chosen branch and the tip of that branch, and then shifts the HEAD to
    that commit.
4. `cherry-pick` any extra commits that need to be included in the release onto
   the affected module(s).
5. Run usual release preparation commands (from `release:test` onwards).
6. Publish the release.

Any extra commits included in a tweak release should be applied to the release
branch as soon as possible (if they weren't cherry-picked from it). Avoid
merging the tagged release into the branch to achieve this, as this will include
the release commit, which may pin Composer dependencies to specific versions.

#### Updating Composer requirements in minor releases

We keep core modules in lockstep at the minor level - that is, we can release
patches (e.g. 4.5.x) for individual modules, but when we perform a minor release
(4.x.0), we ship that version of every core module. To this end, the Composer
dependencies of each module need to be manually adjusted when we perform a minor
release - for example, the `cms` module version `4.6.0` must include a minimum
requirement of `framework` `^4.6`. This ensures that language level requirements
(e.g. minimum PHP versions) can be safely centralised in the framework module
for surrounding core modules to inherit. In short, ensure you commit updates to
the Composer requirements of every core module after each minor branch is
created, and before you ship the release.

#### Testing the release

Once the release task has completed, it may be ideal to manually test the site out
by running it locally (e.g. `http://localhost/release-3.3.4`) to do some smoke-testing
and make sure that there are no obvious issues missed.

Since `cow` will only run the unit test suite, you'll need to check
the build status of Behat end-to-end tests manually on travis-ci.org.
Check the badges on the various modules available on [github.com/silverstripe](http://github.com/silverstripe).

It's also ideal to eyeball the Git changes generated by the release tool, making sure
that no translation strings were unintentionally lost, and that the changelog was generated correctly.

In particular, double check that all necessary information is included in the release notes,
including:

* Upgrading notes
* Security fixes included
* Major changes

Before publication, ensure that the release plan has been peer reviewed by another member of the core team.

Once this has been done, then the release is ready to be published live.

### Stage 2: Release publication

Once a release has been generated, has its translations updated, changelog generated,
and tested, the next step is to publish the release by tagging all modules
in the release plan.

`cow release:publish <version> [<recipe>] -vvv`

Example on how to publish the installer:

`cow release:publish 4.0.1 silverstripe/installer`

Options:

* `-vvv` to ensure all underlying commands are echoed
* `--directory <directory>` to specify the folder to look for the project created in the prior step. As with
  above, it will be guessed if omitted. You can run this command in the `./release-<version>` directory and
  omit this option.

Note: We are no longer creating or publishing archive downloads on silverstripe.org/download.

Once all of these commands have completed there are a couple of final tasks left that
aren't strictly able to be automated:

* It will be necessary to perform a post-release merge
  on open source. This normally will require you to merge the temporary release branch into the
  source branch (e.g. merge 3.2.4 into 3.2), or sometimes create new branches if
  releasing a new minor version, and bumping up the branch-alias in composer.json.
  E.g. branching 3.3 from 3, and aliasing 3 as 3.4.x-dev. You can then delete
  the temporary release branches. This will need to be done before updating the
  release documentation in stage 3.
* Merging up the changes in this release to newer branches, following the
  SemVer pattern (e.g. 3.2.4 > 3.2 > 3.3 > 3 > master). The more often this is
  done the easier it is, but this can sometimes be left for when you have
  more free time. Branches not receiving regular stable versions anymore (e.g.
  3.0 or 3.1) can be omitted.
* Set the github milestones to completed, and create placeholders for the next
  minor versions. It may be necessary to re-assign any issues assigned to the prior
  milestones to these new ones.
* Make sure that the [releases page](https://github.com/silverstripe/silverstripe-installer/releases)
  on github shows the new tag.

*Updating non-patch versions*

If releasing a new major or minor version it may be necessary to update various SilverStripe portals. Normally a new
minor version will require a new branch option to be made available on each site menu. These sites include:

* [docs.silverstripe.org](https://docs.silverstripe.org):
  * New branches (minor releases) require a code update. Changes are made to
    [github](https://github.com/silverstripe/doc.silverstripe.org) and deployed via
    [SilverStripe Platform](https://platform.silverstripe.com/naut/project/SS-Developer-Docs/environment/Production/)
  * The new version needs to be added to `app/_config/docs-repositories.yml`
  * Update the version for the "contributing" rewrite rule in `.htaccess` (`RewriteRule ^(.*)/(.*)/contributing/?(.*)?$ ...`)
  * Updates to markdown only can be made via the [build tasks](https://docs.silverstripe.org/dev/tasks).
    See below for more details.
* [userhelp.silverstripe.org](https://userhelp.silverstripe.org/en/3.2):
  * Updated similarly to docs.silverstripe.org: Code changes are made to
    [github](https://github.com/silverstripe/userhelp.silverstripe.org) and deployed via
    [SilverStripe Platform](https://platform.silverstripe.com/naut/project/SS-User-Docs/environment/Production/).
  * The content for this site is pulled from [silverstripe-userhelp-content](https://github.com/silverstripe/silverstripe-userhelp-content)
  * Updates to markdown made via the [build tasks](https://userhelp.silverstripe.org/dev/tasks).
    See below for more details.
* [demo.silverstripe.org](http://demo.silverstripe.org/): Update code on
  [github](https://github.com/silverstripe/demo.silverstripe.org/)
  and deployed via [SilverStripe Platform](https://platform.silverstripe.com/naut/project/ss3demo/environment/live).
* [api.silverstripe.org](https://api.silverstripe.org): Update on [github](https://github.com/silverstripe/api.silverstripe.org)
  and deployed via [SilverStripe Platform](https://platform.silverstripe.com/naut/project/api/environment/live). Currently
  the only way to rebuild the api docs is via SSH in and running the apigen task.

Further manual work on major or minor releases:

 * Check that `Deprecation::notification_version('4.0.0');` in framework/_config.php points to
the right major version. This should match the major version of the current release. E.g. all versions of 4.x
should be set to `4.0.0`.
 * Update the [userhelp.silverstripe.org](https://userhelp.silverstripe.org) version link in `LeftAndMain.help_links`

*Updating markdown files*

When updating markdown on sites such as userhelp.silverstripe.org or docs.silverstripe.org, the process is similar:

* Run `RefreshMarkdownTask` to pull down new markdown files.
* Then `RebuildLuceneDocsIndex` to update search indexes.

Running either of these tasks may time out when requested, but will continue to run in the background. Normally
only the search index rebuild takes a long period of time.

Note that markdown is automatically updated daily, and this should only be done if an immediate refresh is necessary.

### Stage 3: Let the world know

Once a release has been published there are a few places where user documentation
will need to be regularly updated.

* Make sure that the [download page](http://www.silverstripe.org/download) on
  silverstripe.org has the release available. If it's a stable, it will appear
  at the top of the page. If it's a pre-release, it will be available under the
  [development builds](http://www.silverstripe.org/download#download-releases)
  section. You can cache-bust this by adding `?release=<version>` to
  the url. If things aren't working properly (and you have admin permissions)
  you can run the [CoreReleaseUpdateTask](http://www.silverstripe.org/dev/tasks/CoreReleaseUpdateTask)
  to synchronise with packagist.
* Ensure that [docs.silverstripe.org](http://docs.silverstripe.org) has the
  updated documentation and the changelog link in your announcement works.
* Announce the release on the ["Releases" forum](https://forum.silverstripe.org/c/releases).
  Needs to happen on every minor release for previous releases, see [supported versions](release_process/#supported-versions)
* Announce any new EOLs for minor versions on the ["Releases" forum](https://forum.silverstripe.org/c/releases).
* Update the [roadmap](https://www.silverstripe.org/roadmap) with new dates for EOL versions ([CMS edit link](https://www.silverstripe.org/admin/pages/edit/EditForm/3103/field/TableComponentItems/item/670/edit))
* Update the [Slack](https://www.silverstripe.org/community/slack-signup/) topic to include the new release version.
* For major or minor releases: Work with SilverStripe marketing to get a blog post out.
  They might choose to announce the release on social media as well. 
* If the minor or major release includes security fixes, follow the publication instructions in the [Security Release Process](#security-release-process) section.
* If you released a minor, raise a PR to add a provisional changelog for the next minor release based on the
  [template](https://github.com/silverstripe/silverstripe-installer/blob/4/.cow/changelog.md.twig). This allows
  contributors to start adding release notes for changes in the next minor prior to its release.

## See also

* [Release Process](release_process)
* [Translation Process](translation_process)
* [Core committers](/project_governance/core_committers)

If at any time a release runs into an unsolveable problem contact the
core committers on the [discussion group](https://groups.google.com/forum/#!forum/silverstripe-committers)
to ask for support.
