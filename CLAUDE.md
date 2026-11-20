# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for creating a path resource on an Amazon API Gateway REST API. The template is designed to be referenced by parent CloudFormation stacks.

**Key characteristics:**

- Nested CloudFormation template (referenced via `TemplateURL`)
- CI builds (non-empty `CiSuffix`) create the REST API (regional endpoint, hardcoded)
- Non-CI runs attach the path resource to an existing REST API via `RestApiId` and `ParentId`
- Parameterized REST API naming with account ID, environment, and region
- Automated semantic versioning and releases
- AWS OIDC authentication for CI/CD deployments

## Project Structure

```text
cloudformation/
├── template.yaml                  # Nested template: REST API and path resource
├── parameters.json                # Sample parameter values
└── stack-config.json              # Stack name, template file, parameter file (used by CI)

.github/workflows/
├── ci.yaml                        # Builds and deploys the template on cloudformation/ changes
├── release.yaml                   # Semantic release on push to main
├── create-branch.yaml             # Auto-create feature branches from issues
└── ...

.env/
└── environments.yaml              # Environment-to-GitHub-environment mapping and regions

scripts/plugins/
├── release.config.js              # Semantic-release configuration
└── (other release plugins)        # Custom commit analysis, notes generation

.devcontainer/
└── devcontainer.json              # Dev container setup (Node.js 20)

package.json                       # Dependencies: semantic-release, commitizen
README.md                          # Template documentation and usage examples
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Trigger semantic release (usually automatic on main)

```bash
npm run release
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` type. Only `feat` and `fix` trigger releases.

## Key Architecture Concepts

### Nested Stack Pattern

This repo provides a **nested stack template** that is referenced from parent/root CloudFormation stacks via `TemplateURL`. The template is self-contained.

- **Parent stack** calls: `AWS::CloudFormation::Stack` with `TemplateURL` pointing to S3
- **Nested template** exposes values via the `Outputs` section
- Parent retrieves outputs via `!GetAtt NestedStack.Outputs.OutputKey`

Note: the outputs in `cloudformation/template.yaml` do not use `Export`. Parents must read them through `!GetAtt`.

### REST API Naming Convention

A CI build (non-empty `CiSuffix`) names the REST API from parameters:

```bash
{ProjectName}-api-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `cfn-nested-api-123456789012-devl-us-east-1-12345`

Non-CI runs create no REST API and use the `RestApiId` parameter instead.

This ensures:

- Uniqueness across AWS accounts and regions
- Environment isolation
- Consistent naming for infrastructure automation

## Key Files to Understand

### `cloudformation/template.yaml`

**Purpose:** Creates an API Gateway REST API and a path resource under its root.

**Key inputs:**

- `ProjectName` (default `cfn-nested`): Project prefix
- `PathPart` (default `orders`): Path segment created under the parent resource
- `RestApiId` (default `""`): Existing REST API ID. Required when `CiSuffix` is empty
- `ParentId` (default `""`): Parent resource ID, usually the REST API root. Required when `CiSuffix` is empty
- `Environment` (default `devl`): `devl`, `test`, or `prod`
- `CiSuffix` (default `""`): Non-empty marks a CI build (creates the REST API and appends the suffix to its name)

**Rules:** `RequireExistingApiWhenNotCi` fails the stack if `RestApiId` or `ParentId` is empty when `CiSuffix` is empty.

**Key resources:**

- `RestApi` (`AWS::ApiGateway::RestApi`): Condition `IsCiBuild`. Tagged with `ProjectName` and `Environment`
- `RestApiResource` (`AWS::ApiGateway::Resource`): Always created. `RestApiId` and `ParentId` are `!If [IsCiBuild, ...]`: the created API and its root for CI builds, otherwise the parameters. Named by `PathPart`

**Key outputs:**

- `RestApiId`: Created or provided REST API ID
- `RestApiRootResourceId`: Parent resource ID the path resource was created under
- `RestApiResourceId`: ID of the path resource created by `PathPart`

**Not yet created:** methods, child resources, deployments, and stages.

### `cloudformation/parameters.json` and `cloudformation/stack-config.json`

- `stack-config.json` names the stack, template file, and parameter file that CI uses.
- `parameters.json` holds sample values. Its keys do not all match the template's parameters, and it is not in the `[{"ParameterKey": ..., "ParameterValue": ...}]` format that `--parameters file://` expects. Use `--parameter-overrides` with `aws cloudformation deploy` instead.

### `.github/workflows/ci.yaml`

**Triggered on:**

- Manual `workflow_dispatch` (anytime)
- Pull requests targeting `main` that change `cloudformation/**`
- Pushes to `feature/**` and `bug/**` branches that change `cloudformation/**` or the workflow itself

**Phases:**

1. **Load config:** Reads `cloudformation/stack-config.json`
2. **CI build:** Calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with environment `ci`
3. **Changelog and release:** On `main` only

**Environment setup:**

- Uses AWS OIDC for keyless authentication (`id-token: write`)
- Requires GitHub environment `ci` with OIDC trust configured
- `.env/environments.yaml` maps `ci` and `devl` to the `AWS-CFN-TEMPLATES` GitHub environment and lists `us-east-1` as the region

### `.github/workflows/release.yaml`

**Triggered:** On push to main

**Process:**

1. Analyze commits (conventional format: `feat:`, `fix:`, `BREAKING CHANGE:`)
2. Generate release notes
3. Update CHANGELOG.md
4. Create GitHub release and tag
5. Commit version bump

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` → MAJOR bump (0.1.0 → 1.0.0)
- Other commits → no release

## Testing & Validation

**Manual template validation:**

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

**Manual stack deployment:**

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-aws-api-gateway-resource-stack \
  --parameter-overrides ProjectName=my-project Environment=devl \
    RestApiId=abc123def4 ParentId=xyz789root \
  --region us-east-1
```

For a CI build, pass `CiSuffix=<suffix>` instead of `RestApiId` and `ParentId`.

## AWS Credentials & Environment Variables

**GitHub environment `ci` requires:**

- An OIDC role that the GitHub OIDC provider trusts for this repository
- Region `us-east-1` (see `.env/environments.yaml`)

## Conventional Commits & Release Flow

This repo enforces conventional commits to drive semantic versioning:

```bash
npx cz commit
```

Commit types:

- `feat: add support for X` → triggers MINOR release
- `fix: correct behavior of Y` → triggers PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits to `main` trigger releases. Feature branches use this format but releases happen on merge to main.

## When Modifying the Template

1. **Edit `cloudformation/template.yaml`**
2. **Update `cloudformation/parameters.json`** if parameters are added or renamed
3. **Test locally** with `aws cloudformation validate-template`
4. **Create a PR** with a conventional commit message (e.g., `feat: add stage resource parameter`)
5. **CI validates and deploys** the template automatically
6. **Merge to main** → release workflow creates version tag and GitHub release

## Dev Container

Pre-configured with:

- Node.js 20
- GitHub Copilot extension

Use via VS Code: `code --remote-container-url <repo-url>`

## Current Branch

Main branch is the release branch. Feature work branches from here and merges back via PR. Branch naming follows: `{type}/CFN-{issue-number}-{slug}` (e.g., `feature/CFN-42-add-encryption`).
