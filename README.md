# usc-gha

:warning: Deprecated, use https://github.com/ingka-group-digital/usc-gha

A Github action for using the upload service client, `usc`.

## Setup

- Add an action below to your workflow.
- Change the code to fit your project and paths.
- Add the required secrets, the AWS keys, to your Github project.
- Push it, then watch the Action tab on Github.
- Debug whatever problems you get.

### Action

```yaml
# Deploy contents of build directory to dev (using OIDC — no long-lived AWS credentials needed)
# Requires: permissions: id-token: write at the job level
# and an AWS IAM role configured to trust your GitHub repository via OIDC.
# To get your OIDC role set up, reach out in #exp-platform-framework-pub on Slack.
- name: Deploy to Dev
  if: github.ref == 'refs/heads/master'
  uses: mammutmw/usc-gha@<full SHA>
  with:
    oidc_role: "<my-usc-username>-oidc-role" # or pass a full ARN: arn:aws:iam::179942336946:role/<my-usc-username>-oidc-role
    cmd: "upload"
    src: "build"
    target: "my-target"
    info_git: "https://github.com/my-org/my-repo"
    info_slack: "#project-slack-channel"
    info_email: "project@email.com"
    info_team: "team-name"
    info_product: "product-name"
```

```yaml
# Deploy contents of build directory to dev (using static AWS credentials)
- name: Deploy to Dev
  if: github.ref == 'refs/heads/master'
  uses: mammutmw/usc-gha@<full SHA>
  with:
    aws_access_key: ${{secrets.AWS_ACCESS_KEY_ID}}
    aws_secret_access_key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
    cmd: "upload"
    src: "build"
    target: "my-target"
    info_git: "https://github.com/my-org/my-repo"
    info_slack: "#project-slack-channel"
    info_email: "project@email.com"
    info_team: "team-name"
    info_product: "product-name"
```

```yaml
# Delete files older than 1 week
- name: Delete old files
  uses: mammutmw/usc-gha@<full SHA>
  with:
    aws_access_key: ${{secrets.AWS_ACCESS_KEY_ID}}
    aws_secret_access_key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
    cmd: "delete"
    target: "my-target"
    older: "1 week ago" # https://github.com/tj/go-naturaldate/blob/master/naturaldate_test.go
    info_git: "https://github.com/my-org/my-repo"
    info_slack: "#project-slack-channel"
    info_email: "project@email.com"
    info_team: "team-name"
    info_product: "product-name"
```

```yaml
# Get detailed list of files older than 1 week limited to 100 files
- name: List files
  uses: mammutmw/usc-gha@<full SHA>
  with:
    aws_access_key: ${{secrets.AWS_ACCESS_KEY_ID}}
    aws_secret_access_key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
    cmd: "list"
    target: "my-target"
    older: "1 week ago" # https://github.com/tj/go-naturaldate/blob/master/naturaldate_test.go
    recursive: "true"
    extra_options: "--long --limit 100"
    info_git: "https://github.com/my-org/my-repo"
    info_slack: "#project-slack-channel"
    info_email: "project@email.com"
    info_team: "team-name"
    info_product: "product-name"
```


### Example workflow

Here's a full example, [usc-github-action-example](https://github.com/ingka-group-digital/usc-github-action-example).

```yaml
# .github/workflows/main.yml
name: Node build and deploy with USC
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e # v6.4.0
        with:
          node-version: 24
      - name: Dump GitHub context, for debugging
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"

      # Your stuff
      - run: npm install

      # Build the site in the build directory.
      # build/
      #     - se/sv/your-project
      #     - ca/en/your-project
      #     - ca/fr/your-project
      - run: npm run build
        env:
          SITE_FOLDER: aj/xx,aj/yy
          IS_RELEASE: ${{github.ref == 'refs/heads/release'}}

      - name: List the files in your build, for debugging
        run: find build -print

        # Deploy contents of build directory to CTE (master) or PROD (release)
      - name: Deploy
        if: github.ref == 'refs/heads/master' || github.ref == 'refs/heads/release'
        uses: mammutmw/usc-gha@<full SHA>
        with:
          aws_access_key: ${{secrets.AWS_ACCESS_KEY_ID}}
          aws_secret_access_key: ${{secrets.AWS_SECRET_ACCESS_KEY}}
          cmd: "upload"
          src: "build"
          target: ${{github.ref == 'refs/heads/release' && 'my-prod-target' || 'my-target'}}
          info_git: "https://github.com/my-org/my-repo"
          info_slack: "#project-slack-channel"
          info_email: "project@email.com"
          info_team: "team-name"
          info_product: "product-name"
```

### Parameters

| Name                  | Description                                                                                                       | Default  |
| --------------------- | ----------------------------------------------------------------------------------------------------------------- | -------- |
| oidc_role             | IAM role to assume via OIDC. Short name (e.g. `my-service`) expanded to full ARN, or full ARN. Mutually exclusive with `aws_access_key`/`aws_secret_access_key`. Job must set `permissions: id-token: write`. | optional |
| aws_access_key        | The AWS_ACCESS_KEY_ID. Mutually exclusive with `oidc_role`.                                                       | optional |
| aws_secret_access_key | The AWS_SECRET_ACCESS_KEY. Mutually exclusive with `oidc_role`.                                                   | optional |
| cmd                   | 'The command to run'                                                                                              | 'upload' |
| debug                 | 'Debug output'                                                                                                    | false    |
| src                   | 'root directory of files'                                                                                         | required for update and optional for delete |
| ignore_empty          | 'ignore errors cause by empty file list'                                                                          | false    |
| dry                   | 'dry run, only output files to be uploaded'                                                                       | false    |
| files                 | 'Comma-separated list of files to wait upload'                                                                    | optional |
| target                | 'The target site and (optionally) directory'                                                                      | required |
| timeout               | 'timeout in seconds'                                                                                              | 60       |
| verbose               | 'verbose output'                                                                                                  | true     |
| wait                  | 'wait until files are uploaded'                                                                                   | false    |
| recursive             | 'Recursively include sub directories'                                                                                | false    |
| info_git              | 'Git repository of this project'                                                                                  | optional |
| info_slack            | 'Slack channel of this project'                                                                                   | optional |
| info_email            | 'Email address of this project or person responsible'                                                             | optional |
| info_team             | 'Team name of this project'                                                                                       | optional |
| info_product          | 'Product name of this project'                                                                                    | optional |
| newer                 | Files must be newer than this date, format: https://github.com/tj/go-naturaldate/blob/master/naturaldate_test.go' | optional |
| older                 | Files must be older than this date, format: https://github.com/tj/go-naturaldate/blob/master/naturaldate_test.go' | optional |
| includes              | Files must match this regexp                                                                                      | optional |
| excludes              | Files must NOT match this regexp                                                                                  | optional |
| extra_options         | Extra options not supported by all commands                | optional |

## Releasing a new version

Refer to the release steps in https://github.com/ingka-group-digital/usc-gha.