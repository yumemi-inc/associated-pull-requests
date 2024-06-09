[![CI](https://github.com/yumemi-inc/associated-pull-requests/actions/workflows/ci.yml/badge.svg)](https://github.com/yumemi-inc/associated-pull-requests/actions/workflows/ci.yml)

# Associated Pull Requests

A GitHub Action that outputs a list of pull request numbers associated with commits between any references.

## Usage

See [action.yml](action.yml) for available action inputs and outputs.
Note that this action requires `contents: read` and `pull-requests: read` permissions.

### Supported workflow trigger events

Works on any event.
Basically it works as is, but if you want to customize it, refer to the [Specify comparison targets](#specify-comparison-targets) section.

### Basic

Use list of pull request numbers from `numbers` output.

```yaml
- uses: yumemi-inc/associated-pull-requests@v1
  id: associated-pr
- run: |
    for number in ${{ steps.associated-pr.outputs.numbers }}; do
      # do somethihg..
      echo "$number"
    done
```

By default, they are separated by spaces, but if you want to change the separator, specify it with `separator` input.

```yaml
- uses: yumemi-inc/associated-pull-requests@v1
  id: associated-pr
  with:
    separator: ','
```

If you want to output in JSON or Markdown instead of plain text like above, specify it with `format` input (default is `plain`).

```yaml
- uses: yumemi-inc/associated-pull-requests@v1
  id: associated-pr
  with:
    format: 'json' # or markdown
```

If you want to change the bullets in a list in Markdown format, specify `bullet` input (default is `- #`).

### Use other than merge commits

By default, list pull requests associated with merge commits.
In this case, pull requests merged with *[Squash and merge](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github#squashing-your-merge-commits)* or *[ Rebase and merge](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/about-merge-methods-on-github#rebasing-and-merging-your-commits)* cannot be detected.

To include pull requests merged with these in the results, specify `false` for `merge-commit-only` input.

```yaml
- uses: yumemi-inc/associated-pull-requests@v1
  id: associated-pr
  with:
    merge-commit-only: false
```

In this case, all pull requests can be detected, but the number of API calls will increase, so be careful when using it when the commit history is long.

### Specify comparison targets

Commits between `head-ref` input and `base-ref` input references are used to search for associated pull requests.
The default values ​​for these inputs for each workflow trigger event are the same as [yumemi-inc/path-filter](https://github.com/yumemi-inc/path-filter#specify-comparison-targets), so refer to it for details, but note that this Associated Pull Requests action always performs [three-dot](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-comparing-branches-in-pull-requests#three-dot-and-two-dot-git-diff-comparisons) comparison, does not support two-dot.

For example, to explicitly specify these inputs and list pull requests between `main` and `develop` branches:

```yaml
- uses: yumemi-inc/associated-pull-requests@v1
  id: associated-pr
  with:
    head-ref: 'develop'
    base-ref: 'main'
```

## Other examples

In a pull request, comment if there are any associated pull requests.

![image](doc/image.png)

```yaml
name: Associated PR Report

on:
  pull_request:
    types:
      - opened
      - edited
      - reopened
      - synchronize

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  report:
    name: PR report
    runs-on: ubuntu-latest
    permissions:
      contents: read # for yumemi-inc/associated-pull-requests
      pull-requests: write # for yumemi-inc/associated-pull-requests and yumemi-inc/comment-pull-request
    steps:
      - name: List associated pull request numbers
        uses: yumemi-inc/associated-pull-requests@v1
        id: associated-pr
        with:
          format: 'markdown'
      - name: Comment
        if: steps.associated-pr.outputs.numbers != null
        uses: yumemi-inc/comment-pull-request@v1
        with:
          comment: |
            ### pull requests merged into this pull request:
            ${{ steps.associated-pr.outputs.numbers }}
          previous-comment: 'hide'
```

## Limitation

- Due to the limitations of the [GitHub API](https://docs.github.com/en/rest/commits/commits?apiVersion=2022-11-28#list-pull-requests-associated-with-a-commit) used by this action, pull requests that reuse commits elsewhere will also be detected. Currently, this noise is reduced by limiting pull requests with merged status, and this behavior can be changed with `merged-status-only` input.
- Of course, if you merge or push without going through a pull request, there will be no associated pull request. If this is critical, use *branch protection rules* or *rulesets* to require merging via pull request.
