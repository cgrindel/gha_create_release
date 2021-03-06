# Create Release GitHub Action for Bazel Workspaces Using bzlrelease

This repository contains a GitHub Action that creates a release for Bazel workspaces which use
[cgrindel/bazel-starlib/bzlrelease](https://github.com/cgrindel/bazel-starlib/tree/main/bzlrelease).

The action will do the following:

1. Creates a release tag with the latest commit on your main branch.
2. Generates release notes by calling `//release:generate_release_notes` using the workspace snippet
   generated by `//release:generate_workspace_snippet`.
3. Creates a release using the generated release notes and the release tag.
4. Updates the `README.md` file with the release's workspace snippet.
5. Creates a PR with the `README.md` update and configures it to auto-merge.

_(This assumes that the release targets were defined under `//release`.)_

## Table of Contents

* [Quickstart](#quickstart)
  * [Implement bzlrelease in your Bazel workspace](#implement-bzlrelease-in-your-bazel-workspace)
  * [Set Up a GitHub App to Generate Tokens](#set-up-a-github-app-to-generate-tokens)
  * [Implement a GitHub Actions Workflow for Creating a Release](#implement-a-github-actions-workflow-for-creating-a-release)
  * [Create a release](#create-a-release)


## Quickstart

The following provides a brief introduction to setting up a release process using `bzlrelease` and
the action in this repository.

### Implement bzlrelease in your Bazel workspace

See [bzlrelease](https://github.com/cgrindel/bazel-starlib/tree/main/bzlrelease) for more details.

For this quickstart, we will assume that you implemented your release targets in a Bazel package
located at `//release`. The `BUILD.bazel` file should look something like the following:

```python
load(
    "@cgrindel_bazel_starlib//bzlrelease:defs.bzl",
    "create_release",
    "generate_release_notes",
    "generate_workspace_snippet",
    "update_readme",
)

# MARK: - Release

generate_workspace_snippet(
    name = "generate_workspace_snippet",
    template = "workspace_snippet.tmpl",
)

generate_release_notes(
    name = "generate_release_notes",
    generate_workspace_snippet = ":generate_workspace_snippet",
)

update_readme(
    name = "update_readme",
    generate_workspace_snippet = ":generate_workspace_snippet",
)

create_release(
    name = "create",
    workflow_name = "Create Release",
)

```


### Set Up a GitHub App to Generate Tokens

The default Github token that is provided to the workflow does not have permission to create
releases and PRs. This [article from
peter-evans/create-pull-request](https://github.com/peter-evans/create-pull-request/blob/master/docs/concepts-guidelines.md#authenticating-with-github-app-generated-tokens)
explains how to create a GitHub application to generate tokens with the appropriate permissions.
When you have created the token generator application, you need to create two secrets in your
repository. The first is named `APP_ID`. It's value is the `App ID: XXXX` value (e.g., `XXXX`) found
on the token generator application's About page. The second secret is named `APP_PRIVATE_KEY`. It's
value is the private key that you generated while creating the token generator application.


### Implement a GitHub Actions Workflow for Creating a Release

In your repository, create a file at `.github/workflows/create_release.yml` with the following
contents:

```yaml
name: Create Release

on:
  workflow_dispatch:
    inputs:
      release_tag:
        required: true
        type: string
      reset_tag:
        type: boolean
        default: false
      base_branch:
        description: The branch being merged to.
        type: string
        default: main

jobs:
  create_release:
    runs-on: ubuntu-latest
    env:
      CC: clang

    steps:

    # Check out your code
    - uses: actions/checkout@v2

    # Generate a token that has permssions to create a release and create PRs.
    - uses: tibdex/github-app-token@v1
      id: generate_token
      with:
        app_id: ${{ secrets.APP_ID }}
        private_key: ${{ secrets.APP_PRIVATE_KEY }}

    # Configure the git user for the repository
    - uses: cgrindel/gha_configure_git_user@v1

    # Create the release
    - uses: cgrindel/gha_create_release@v1
      with:
        release_tag: ${{  github.event.inputs.release_tag }}
        reset_tag: ${{  github.event.inputs.reset_tag }}
        base_branch: ${{  github.event.inputs.base_branch }}
        github_token: ${{ steps.generate_token.outputs.token }}
```

Commit this file and merge to your main branch.


### Create a release

After merging the `create_release.yml` workflow to your main branch, you can create a release by
running the following inside your repository:

```sh
# Create a release with the tag v1.2.3
$ bazel run //release:create -- v1.2.3
```

This utility will launch the `create_release.yml` workflow in GitHub Actions.
