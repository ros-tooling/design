# Design: GitHub Action Release Process

## Context

We vend a variety of GitHub actions from the ros-tooling organization.
Til now we have not defined a consolidated process for their management.
This means that there is some confusion about release tags, indecision around the final structure of their repositories, and difficulties around the production of build artifacts.

## Goal

Lay out a cohesive plan for repository structure, reliable and automatic generation of build artifacts, and a release tagging plan.

## Design

### Repository structure

Design constraint: Publish ROS CI Actions to the GitHub Marketplace for discoverability.

There was a consideration of combining several of the ROS CI actions into a single repository, similar to https://github.com/octokit/graphql-action, as requested in https://github.com/ros-tooling/roadmap/issues/5

However, in order to release our Actions into the GitHub Marketplace, we cannot do this.
According to the [GitHub Marketplace Documentation](https://docs.github.com/en/free-pro-team@latest/actions/creating-actions/publishing-actions-in-github-marketplace) - "Each repository must contain a single action."
Therefore, we will maintain a separate repository for each action that we wish to publish to the marketplace.

The primary goal of combining repositories was to reduce code duplication - this will be accomplished by creating a Node package containing shared code, in an additional repository, to be consumed by the released Actions. This package can easily released using GitHub package registries.

### Build artifact generation

The GitHub actions are built with Typescript - which requires compilation into Javascript.
This compilation is expected to be done manually by contributors, and checked in alongside pull requests.
This makes parallel development difficult - when the development branch is updated, one can't just click "Update Branch" as this does not rebuild the Javascript output - it must be done manually.
Additionally, as dependencies are updated via Dependabot - the build output is not updated, so when a developer does make a change, many unrelated-looking changes can end up in their `dist/index.js`

All of this just goes to show that we need an automated solution:

* Development will occur on the default branch
* The build output `dist/index.js` will _not_ be checked into this development branch
* CI for PRs and on schedule will build the sources before running the tests
* A separate branch, `release`, will be created. Every time a new commit is pushed to `master`, a GitHub Action will be triggered that builds `master` and pushes the resulting build artifact to `release`

In this way, parallel development will be easier by allowing simple branch updates, Dependabot updates will actually update the build artifacts, and developers never have to remember to check in build artifacts, they can focus on the changes at hand.

### Release tagging

GitHub Actions do not provide any semantic versioning as is afforded by e.g. setuptools or Node.js package specifications.
Therefore, when consuming our GitHub Actions, users must update even patch-level bugfixes in their codebase to receive these updates.
This can be managed by Dependabot configuration, but that is not something that all users have enabled, or necessarily want to use.

Other popular Actions manage this situation by providing "tracking tags" that follow specific minor or major versions - such as https://github.com/actions/checkout/tree/v2.
This is probably the most-used GitHub Action, and it recommends `uses: actions/checkout@v2` - which does not actually happen to be one of the releases on the GitHub marketplace! The versions released to the marketplace follow semantic versioning, the latest is `v2.3.4` - the `v2` tag is updated to track the latest `v2.x.x` whenever a release is cut.

Our solution will do the same:
* Tag releases on the `release` branch proposed in "Build artifact generation"
* Tag specific semantic versions such as `0.2.5` and release these to the GitHub Marketplace
* Track all pre-1.0 minor versions as `v0.x`, updating this every time a new patch release is created. This allows users to receive bugfixes without doing anything at all
* Once reaching version 1.0, keep major version tags `v1`, `v2`, etc. that are updated with all minor and patch updates within that major version

### Documentation

Once the exact specific process are implemented, a `DEVELOPING.md` will be written, and copied to each Action managed by this process. This file can be managed by file synchronization as laid out in the GitHub Repository Management design.
