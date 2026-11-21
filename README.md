# CloudFormation Nested Stack Template : API Gateway Rest API Resource

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-api-gateway-resource/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/5215343ac7af12cd93778f7eb984b4a8/raw/cfn-nested-aws-api-gateway-resource.json)](https://gist.github.com/subhamay-bhattacharyya-cfn/5215343ac7af12cd93778f7eb984b4a8)

---

A reusable CloudFormation nested stack template that creates a path resource on an Amazon API Gateway REST API. It is designed to be invoked from a parent stack with `AWS::CloudFormation::Stack`.

## Overview

The template in `cloudformation/template.yaml` creates:

- An `AWS::ApiGateway::Resource` with the path segment set by `PathPart` (default `orders`), under the parent resource given by `ParentId`, on the REST API given by `RestApiId`
- When `CiSuffix` is set (a CI build pipeline), it also creates an `AWS::ApiGateway::RestApi` with a regional endpoint, and the path resource is created under that API's root

Methods, child resources, and deployments are not created by this template yet.

## Repository Layout

```text
cloudformation/
├── template.yaml          # Nested stack template: REST API and path resource
├── parameters.json        # Sample parameter values
└── stack-config.json      # Stack name, template file, and parameter file used by CI

.github/workflows/
├── ci.yaml                # Builds and deploys the template on changes to cloudformation/
├── release.yaml           # Semantic release on push to main
└── ...

scripts/plugins/           # Semantic-release configuration and plugins
package.json               # Dependencies: semantic-release, commitizen
```

## Parameters

| Parameter | Type | Default | Required | Description |
| --- | --- | --- | --- | --- |
| `ProjectName` | String | `cfn-nested` | No | Project name, 1–20 chars, lowercase letters, numbers, and hyphens |
| `PathPart` | String | `orders` | No | Path segment created under the parent resource, 1–256 chars: letters, numbers, `.`, `_`, `-` |
| `RestApiId` | String | `""` | Yes, when `CiSuffix` is empty | ID of an existing REST API. Ignored by CI builds |
| `ParentId` | String | `""` | Yes, when `CiSuffix` is empty | ID of the parent resource (usually the REST API root resource ID). Ignored by CI builds |
| `Environment` | String | `devl` | No | `devl`, `test`, or `prod` |
| `CiSuffix` | String | `""` | No | CI suffix. A non-empty value marks a CI build: it creates the REST API and appends the suffix to its name |

A CI build is any run with a non-empty `CiSuffix`. It creates the REST API and ignores `RestApiId` and `ParentId`. A non-CI run creates no REST API, so it needs `RestApiId` and `ParentId`. A stack rule fails the deployment if either is empty when `CiSuffix` is empty.

## Naming Convention

A CI build names the REST API from the parameters:

```bash
{ProjectName}-api-{AccountId}-{Environment}-{Region}-{CiSuffix}
```

Example: `cfn-nested-api-123456789012-devl-us-east-1-12345`

## Outputs

| Output | Description |
| --- | --- |
| `RestApiId` | ID of the REST API: created by a CI build, otherwise the `RestApiId` parameter |
| `RestApiRootResourceId` | Parent resource ID the path resource was created under: the root resource for CI builds, otherwise the `ParentId` parameter |
| `RestApiResourceId` | ID of the path resource created by `PathPart` |

## Usage

### 1. Upload the Template to S3

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/template.yaml
```

### 2. Reference from a Parent Stack

```yaml
ApiNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/template.yaml
    Parameters:
      ProjectName: my-project
      PathPart: orders
      RestApiId: abc123def4
      ParentId: xyz789root
      Environment: devl
      CiSuffix: ""

Outputs:
  RestApiId:
    Value: !GetAtt ApiNestedStack.Outputs.RestApiId
  RestApiRootResourceId:
    Value: !GetAtt ApiNestedStack.Outputs.RestApiRootResourceId
```

### 3. Deploy Directly with the AWS CLI

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name cfn-nested-aws-api-gateway-resource-stack \
  --parameter-overrides ProjectName=my-project Environment=devl \
  --region us-east-1
```

For a non-CI run, also pass `RestApiId=<api-id>` and `ParentId=<parent-resource-id>`. For a CI build, pass `CiSuffix=<suffix>`.

## Validation

```bash
aws cloudformation validate-template --template-body file://cloudformation/template.yaml
```

## Best Practices Implemented

- Tags the CI-created REST API with `ProjectName` and `Environment`
- Creates the REST API only in CI builds. Other runs attach the path resource to an existing API

## Contributing

Use conventional commits (`feat:`, `fix:`, `chore:`). See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
