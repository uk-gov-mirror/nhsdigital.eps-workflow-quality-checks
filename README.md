# THIS PROJECT HAS BEEN ARCHIVED AND FUNCTIONALITY MOVED TO https://github.com/NHSDigital/eps-common-workflows

# eps-workflow-quality-checks


A workflow to run the quality checks for EPS repositories. The main element of this lives in the [`quality-checks.yml`](./.github/workflows/quality-checks.yml) configuration file. The steps executed by this workflow are as follows:

- **Install Project Dependencies**
- **Generate and Check SBOMs**: Creates Software Bill of Materials (SBOMs) to track dependencies for security and compliance. Uses [THIS](https://github.com/NHSDigital/eps-action-sbom) action.
- **Run Linting**
- **Run Unit Tests**
- **Scan git history for secrets**: Scans for secret-like patterns, using https://github.com/NHSDigital/software-engineering-quality-framework/blob/main/tools/nhsd-git-secrets/git-secrets
- **SonarCloud Scan**: Performs code analysis using SonarCloud to detect quality issues and vulnerabilities.
- **Validate CloudFormation Templates** (*Conditional*): If CloudFormation, AWS SAM templates or CDK are present, runs `cfn-lint` (SAM and cloudformation only) and `cfn-guard` to validate templates against AWS best practices and security rules.
- **Validate Terraform Plans** Terraform plans can also be scanned by `cfn-guard` by uploading plans as artefacts in the calling workflow. All Terraform plans must end _terraform_plan and be in json format.
- **CDK Synth** (*Conditional*): Runs `make cdk-synth` if packages/cdk folder exists
- **Check Licenses**: Runs `make check-licenses`.
- **Check Python Licenses** (*Conditional*): If the project uses Poetry, scans Python dependencies for incompatible licenses.

The secret scanning also has a dockerfile, which can be run against a repo in order to scan it manually (or as part of pre-commit hooks). This can be done like so:
```bash
docker build -f https://raw.githubusercontent.com/NHSDigital/eps-workflow-quality-checks/refs/tags/v3.0.0/dockerfiles/nhsd-git-secrets.dockerfile -t git-secrets .
docker run -v /path/to/repo:/src git-secrets --scan-history .
```
For usage of the script, see the [source repo](https://github.com/NHSDigital/software-engineering-quality-framework/blob/main/tools/nhsd-git-secrets/git-secrets). Generally, you will either need `--scan -r .` or `--scan-history .`. The arguments default to `--scan -r .`, i.e. scanning the current state of the code.

In order to enable the pre-commit hook for secret scanning (to prevent developers from committing secrets in the first place), add the following to the `.devcontainer/devcontainer.json` file:
```json
{
    "remoteEnv": { "LOCAL_WORKSPACE_FOLDER": "${localWorkspaceFolder}" },
    "postAttachCommand": "docker build -f https://raw.githubusercontent.com/NHSDigital/eps-workflow-quality-checks/refs/tags/v4.0.2/dockerfiles/nhsd-git-secrets.dockerfile -t git-secrets . && pre-commit install --install-hooks -f",
    "features": {
      "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {
        "version": "latest",
        "moby": "true",
        "installDockerBuildx": "true"
      }
    }
}
```

And the this pre-commit hook to the `.pre-commit-config.yaml` file:
```yaml
repos:
- repo: local
  hooks:
    - id: git-secrets
      name: Git Secrets
      description: git-secrets scans commits, commit messages, and --no-ff merges to prevent adding secrets into your git repositories.
      entry: bash
      args:
        - -c
        - 'docker run -v "$LOCAL_WORKSPACE_FOLDER:/src" git-secrets --pre_commit_hook'
      language: system
```

# Usage

## Inputs

None

## Required Makefile targets

In order to run, these `make` commands must be present. They may be mocked, if they are not relevant to the project.

- `install`
- `lint`
- `test`
- `check-licenses`
- `cdk-synth` - only needed if packages/cdk folder exists

## Environment variables

### `SONAR_TOKEN`

Required for the SonarCloud Scan step, which analyzes your code for quality and security issues using SonarCloud.

# Example Workflow Call

To use this workflow in your repository, call it from another workflow file:

```yaml
name: Quality Checks

on:
  push:
    branches:
      - main
      - develop

jobs:
  quality_checks:
    uses: NHSDigital/eps-workflow-quality-checks/.github/workflows/quality-checks.yml@4.0.2
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```
