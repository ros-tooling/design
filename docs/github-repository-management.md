# Design: GitHub Repository Management

## Context

There are now a significant number of repositories under the `ros-tooling` organization.
To support our best practice development, they are configured in two ways:
* In-repository files (dependabot configuration, CODEOWNERS, pull request templates, etc.)
* Repository configuration (branch protection rules, allowed merge methods, team access roles, etc.)

Much of this configuration is the same across all managed repositories - we decide once the practice that we want to follow, and implement this on all repostories.
When we add a new repository, change/evolve a decision, or decide on a new common setting, applying everything can involve many PRs or clicking on settings pages (or writing a custom script to update settings).
This doesn't scale well and takes up precious time that could be spent on more meaningful software development.

## Goal

Provide a single source of truth for configuration of both types (files and settings) that is automatically applied to a list of managed repositories.

### Solution constraints

* Apply updates to managed repositories as Pull Requests
  * This will allow for CI testing to be run on any changes, ensuring they do not break anything
  * Automatically approve and merge based on passing CI, so that human intervention is not needed
* File-based configuration
  * Keep common configuration files in a single repository (Template Repository)
  * Allow specifying files to delete from the managed repositories, in addition to the positive case of files to synchronize from the Template Repository
  * Allow overrides on a per-repo basis to customize their resulting configuration
  * Allow template files that are rendered to the managed repository, replacing variables specific to the repository
* Settings configuration
  * Managed as code, to allow easy inspection and version control
  * Specified in the same place as file configuration
  * Any unspecified settings are not touched, so that repositories can have customization where necessary
  * Allow specifying overrides/extensions of the base set for repositories

Nice to have but not mandatory
* All infrastructure as GitHub Actions, so that we do not need to enable GitHub Apps or any external infrastructure
* Settings configuration can be applied from the Template Repository without copying it over to the managed repository
  * This would avoid having to have a two-step process of "PR to update the settings spec, then have a local action apply it"

## Implementation Notes

There are preexisting projects for some of this functionality (see Appendix)
During the course of implementation evaluate them, with the following order of preference:
* if one gets close to meeting the need, consider contributing any remaining features back to the upstream version.
* If we need to fork a project as a jumping off point, that is second choice
* Lastly, if nothing comes close enough to meeting our constraints, then we may need to start from scratch on a new action - this will be of clear use to other communities within the ROS ecosystem, and is likely generically useful to other software projects

Application of settings will be performed by having the Template Repository configured as follows
* Action workflow, triggered on push to the default branch - that runs the file and settings synchronizers
* Has a GitHub API Token, set as a Secret, that allows it to perform the settings updates and open PRs

Due to the fact that we may decide to use a preexisting project, the following are not set in stone, but are provided as guideposts to clarify the goals.


### The Management Configuration

A YAML list of repositories to manage - to be stored in the template repository.
"Overlay files" are YAML configuration using the same format as the corresponding Settings Config or Files Config.
Values in here override the values from a base config - so that new specifics can be added or existing values overridden, for specific repositories.

Should look something like this:

```
# A "File Configuration" to be applied to all repos
base_file_config: base_files.yml
# A "Settings Configuration" to be applied to all repos
base_settings_config: base_settings.yml
repositories:
- repo: ros-tooling/action-ros-ci
- repo: ros-tooling/libstatistics_collector
  overlays:
    file_config: ros2_package_files.yml
- repo: ros-tooling/setup-ros
  overlays:
    settings_config: setup_ros_settings.yml
```

### The File Configuration

A YAML file specifying what files are to be synchronized - to be stored in the template repository.
Should look something like this:

```
# add or update these files
include_files:
- .github/dependabot.yml
- .github/workflows/autoapprove_and_merge_dependabot.yml
- .github/workflows/autoapprove_and_merge_repository_management.yml
- .github/CODEOWNERS

# these are rendered - so in this case the managed repo would receive ".github/workflows/slightly_different_per_repo.yml" with variables filled in
template_files:
- .github/workflows/slightly_different_per_repo.yml.j2

# ensure these files are not present on the target repo - if processes change, makes sure no dangling files are left
exclude_files:
- .github/workflows/auto_dependabot.yml
```

### The Settings Configuration

A YAML file specifying what GitHub Repository settings are to be synchronized - to be stored in the template repository.
Should look something like this:

```
allow_issues: true
allow_projects: true
allow_wiki: false
squash_merge: true
merge_commit: false
access:
  teams:
    approvers: maintain
    members: triage
    owners: admin
    reviewers: write
branch_protection:
  master:
    require_reviews: true
    required_reviewers: 1
    require_status_checks:
      - DCO
```


## Appendix: Preexisting Projects


### Notable File Synchronization existing projects:
* https://github.com/varunsridharan/action-github-workflow-sync
    * 10star, allows PR
* https://github.com/kbrashears5/github-action-file-sync
    * 6 stars, allows PR
* https://github.com/adrianjost/files-sync-action
    * 2 stars


### Notable Settings Synchronization existing projects
* https://github.com/marketplace/actions/repo-settings-sync
* https://github.com/apps/settings
* https://github.com/probot/probot
* https://github.com/marketplace/actions/github-action-to-run-probot-settings
* https://github.com/probot/settings
