# Backport Github Action

## Getting started

Create a file `/.github/workflows/backport.yml` with the following content:

```yml
name: Automatic backport action

on:
  pull_request_target:
    types: ["labeled", "closed"]

jobs:
  label_checker:
    name: Check labels
    runs-on: ubuntu-latest
    outputs:
      do_backport: ${{ steps.result.outputs.do_backport }}
    steps:
      - id: prefix_check
        uses: agilepathway/label-checker@v1.6.55
        with:
          prefix_mode: true
          any_of: backport-to-
          allow_failure: true
          repo_token: ${{ secrets.GITHUB_TOKEN }}
      - id: label_filter
        uses: agilepathway/label-checker@v1.6.55
        with:
          none_of: [ 'backport', 'was-backported' ]
          allow_failure: true
          repo_token: ${{ secrets.GITHUB_TOKEN }}
      - id: result
        name: Set output
        shell: bash
        run: |
            {
                [ '${{ steps.prefix_check.outputs.label_check }}' = 'success' ] \
                && [ '${{ steps.label_filter.outputs.label_check }}' == 'success' ] \
                && echo 'do_backport=true' \
                || echo 'do_backport=false'
            } >> "$GITHUB_OUTPUT"
      - name: Print status
        shell: bash
        run: 'echo "Label detection status: ${{ steps.check.outputs.label_check }}"'

  backport:
    needs: [ label_checker ]
    name: Backport PR
    if: github.event.pull_request.merged == true && needs.label_checker.outputs.do_backport == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Backport Action
        uses: sorenlouv/backport-github-action@v9.5.1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          auto_backport_label_prefix: backport-to-

      - name: Info log
        if: ${{ success() }}
        run: cat ~/.backport/backport.info.log
        
      - name: Debug log
        if: ${{ failure() }}
        run: cat ~/.backport/backport.debug.log        
          
```
> Note: The action won't run if the PR contains the labels `backport` or `was-backported`. Also it uses the `pull-request-label-checker` action just to run if the PR contains the `backport-to-` label prefix.

Now, to backport a pull request, simply apply the label `backport-to-production`. This will automatically backport the PR to the branch called "production" when the PR is merged. 

## Configuration

For more fine grained customization, and for the ability to run the [Backport Tool](https://github.com/sorenlouv/backport) as a CLI tool locally, you should create a `.backportrc.json` file in the root directory:

```js
// .backportrc.json
{
  // example repo info
  "repoOwner": "torvalds",
  "repoName": "linux",

  // `targetBranch` option allows to automatically backport every PR to a specific branch without the need for labels
  "targetBranches": ["production"],

  // the branches available to backport to
  "targetBranchChoices": ["main", "production", "staging"],

  // In this case, adding the label "backport-to-production" will backport the PR to the "production" branch
  "branchLabelMapping": {
    "^backport-to-(.+)$": "$1"
  }
}
```


 See the [Backport Tool documentation](https://github.com/sorenlouv/backport/blob/main/docs/config-file-options.md) for all configuration options.

