# Administrators Guide

This guide covers AWS account setup, IAM configuration, and GitHub environment configuration for repository administrators. For day-to-day CI/CD and merge workflows, see the [Maintainers Guide](./MAINTAINERS_GUIDE.md).

## CodeBuild Integration

The [Build (CodeBuild)](./../.github/workflows/build-codebuild.yml) workflow runs the full project build (`mise run build`) on AWS CodeBuild. It is triggered manually via `workflow_dispatch` and gated by a protected GitHub environment that requires non-self-approval.

The workflow runs on a GitHub-hosted runner, authenticates to AWS via OIDC, and invokes CodeBuild's `StartBuild` API using the [aws-codebuild-run-build](https://github.com/aws-actions/aws-codebuild-run-build) action. CloudWatch logs are streamed back to the GitHub Actions console. No webhooks are involved.

### Architecture

```text
workflow_dispatch
        │
        ▼
GitHub Environment gate ("codebuild")
  - requires non-self-approval
        │
        ▼
GitHub-hosted runner (ubuntu-latest)
        │
        ├─► configure-aws-credentials (OIDC)
        │     role-to-assume: AWS_CODEBUILD_ROLE_ARN
        │
        └─► aws-codebuild-run-build (StartBuild API)
              project: CODEBUILD_PROJECT_NAME
              buildspec: inline (mise install → mise run build)
              logs: streamed via CloudWatch
```

### Prerequisites

Complete these steps in order. Steps 1-3 are AWS-side, steps 4-5 are GitHub-side.

#### 1. Create the IAM OIDC identity provider

If your account already has the GitHub Actions OIDC provider (`token.actions.githubusercontent.com`), skip to step 2.

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

> The thumbprint is a formality — AWS validates via the OIDC discovery document. GitHub rotates certificates independently.

#### 2. Create the CodeBuild project

> **Prerequisite:** GitHub credentials must already be connected to CodeBuild in this account. If not, import them first:
>
> ```bash
> aws codebuild import-source-credentials \
>   --server-type GITHUB \
>   --auth-type PERSONAL_ACCESS_TOKEN \
>   --token <GITHUB_PAT>
> ```

Deploy the CloudFormation stack below. It creates:

- A CodeBuild project with GitHub source and no buildspec (the workflow passes an inline override)
- A service role with least-privilege permissions for CloudWatch Logs and S3 artifacts
- An S3 bucket for build artifacts (SARIF reports from security scanners)
- A CloudWatch log group with 90-day retention (matches GitHub Actions log retention)

**CloudFormation template:**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >-
  CodeBuild project for the Build (CodeBuild) GitHub Actions workflow.
  The workflow passes an inline buildspec override, so no buildspec file
  is configured on the project.

Parameters:
  ProjectName:
    Type: String
    Default: agent-plugins-build
    Description: >-
      CodeBuild project name. Must match the CODEBUILD_PROJECT_NAME
      GitHub Actions variable.

  GitHubRepoUrl:
    Type: String
    Default: https://github.com/awslabs/agent-plugins
    Description: HTTPS URL of the GitHub repository.

  ComputeType:
    Type: String
    Default: BUILD_GENERAL1_LARGE
    AllowedValues:
      - BUILD_GENERAL1_SMALL
      - BUILD_GENERAL1_MEDIUM
      - BUILD_GENERAL1_LARGE
    Description: Compute resources for the build environment.

  Image:
    Type: String
    Default: aws/codebuild/amazonlinux-x86_64-standard:5.0
    Description: Docker image for the build environment.

  LogRetentionDays:
    Type: Number
    Default: 90
    AllowedValues: [1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365]
    Description: Number of days to retain CloudWatch build logs.

Resources:
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${ProjectName}-artifacts-${AWS::AccountId}'
      LifecycleConfiguration:
        Rules:
          - Id: ExpireArtifacts
            Status: Enabled
            ExpirationInDays: !Ref LogRetentionDays
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/codebuild/${ProjectName}'
      RetentionInDays: !Ref LogRetentionDays

  ServiceRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${ProjectName}-service-role'
      Description: !Sub 'Service role for CodeBuild project ${ProjectName}.'
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: CloudWatchLogs
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowLogs
                Effect: Allow
                Action:
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Sub >-
                  arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/${ProjectName}:*
        - PolicyName: S3Artifacts
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowArtifactUpload
                Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:GetBucketAcl
                  - s3:GetBucketLocation
                Resource:
                  - !GetAtt ArtifactBucket.Arn
                  - !Sub '${ArtifactBucket.Arn}/*'
              - Sid: AllowKMSForArtifacts
                Effect: Allow
                Action:
                  - kms:Decrypt
                  - kms:GenerateDataKey
                Resource: '*'
                Condition:
                  StringEquals:
                    kms:ViaService: !Sub 's3.${AWS::Region}.amazonaws.com'

  Project:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Ref ProjectName
      Description: >-
        Runs the agent-plugins build. Invoked by the Build (CodeBuild)
        GitHub Actions workflow via aws-codebuild-run-build.
      ServiceRole: !GetAtt ServiceRole.Arn
      Source:
        Type: GITHUB
        Location: !Ref GitHubRepoUrl
        BuildSpec: '{}'
      Artifacts:
        Type: S3
        Location: !Ref ArtifactBucket
        NamespaceType: BUILD_ID
        Packaging: ZIP
        EncryptionDisabled: false
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: !Ref ComputeType
        Image: !Ref Image
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED
          GroupName: !Ref LogGroup
      TimeoutInMinutes: 60
      QueuedTimeoutInMinutes: 30

Outputs:
  ProjectName:
    Description: >-
      Store this as the CODEBUILD_PROJECT_NAME variable in GitHub Actions.
    Value: !Ref Project

  ProjectArn:
    Description: ARN of the CodeBuild project.
    Value: !GetAtt Project.Arn

  ArtifactBucketName:
    Description: >-
      S3 bucket for build artifacts (SARIF reports). Pass this as the
      ArtifactBucketName parameter when deploying the IAM role stack in step 3.
    Value: !Ref ArtifactBucket

  ServiceRoleArn:
    Description: ARN of the CodeBuild service role.
    Value: !GetAtt ServiceRole.Arn
```

**Deploy with defaults:**

```bash
aws cloudformation deploy \
  --template-file administrators-guide-codebuild-project.yaml \
  --stack-name agent-plugins-codebuild \
  --capabilities CAPABILITY_NAMED_IAM
```

After deployment, note the `ProjectName` and `ArtifactBucketName` outputs — you will need them in steps 3 and 5.

#### 3. Create the IAM role and policy

Deploy the CloudFormation stack below. It creates a single IAM role with:

- An OIDC trust policy scoped to `repo:awslabs/agent-plugins:environment:codebuild`
- Least-privilege permissions: `codebuild:StartBuild`, `codebuild:BatchGetBuilds`, `logs:GetLogEvents`, and `s3:GetObject` on the artifact bucket from step 2

**CloudFormation template:**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >-
  IAM role for the Build (CodeBuild) GitHub Actions workflow.
  Assumes OIDC federation from the "codebuild" GitHub environment.

Parameters:
  GitHubOrg:
    Type: String
    Default: awslabs
    Description: GitHub organization that owns the repository.

  GitHubRepo:
    Type: String
    Default: agent-plugins
    Description: GitHub repository name.

  GitHubEnvironment:
    Type: String
    Default: codebuild
    Description: >-
      GitHub environment name. The OIDC subject claim is scoped to this
      environment so only approved jobs can assume the role.

  CodeBuildProjectName:
    Type: String
    Default: agent-plugins-build
    Description: Name of the CodeBuild project (from step 2 ProjectName output).

  ArtifactBucketName:
    Type: String
    Default: ''
    Description: >-
      Name of the S3 bucket for build artifacts (from step 2
      ArtifactBucketName output). If provided, the role gets s3:GetObject
      to download SARIF reports.

  CreateOIDCProvider:
    Type: String
    Default: 'true'
    AllowedValues: ['true', 'false']
    Description: >-
      Set to false if the GitHub Actions OIDC provider already exists in
      this account. When false, provide OIDCProviderArn.

  OIDCProviderArn:
    Type: String
    Default: ''
    Description: >-
      ARN of an existing GitHub Actions OIDC provider. Required when
      CreateOIDCProvider is false.

Conditions:
  ShouldCreateOIDCProvider: !Equals [!Ref CreateOIDCProvider, 'true']
  HasExistingOIDCProvider: !Not [!Equals [!Ref OIDCProviderArn, '']]
  HasArtifactBucket: !Not [!Equals [!Ref ArtifactBucketName, '']]

Resources:
  GitHubOIDCProvider:
    Type: AWS::IAM::OIDCProvider
    Condition: ShouldCreateOIDCProvider
    Properties:
      Url: https://token.actions.githubusercontent.com
      ClientIdList:
        - sts.amazonaws.com
      ThumbprintList:
        - 6938fd4d98bab03faadb97b34396831e3780aea1

  GitHubActionsCodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: !Sub '${GitHubRepo}-codebuild-gha'
      Description: !Sub >-
        Assumed by GitHub Actions (OIDC) to run CodeBuild builds for
        ${GitHubOrg}/${GitHubRepo} from the ${GitHubEnvironment} environment.
      MaxSessionDuration: 3600
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Federated: !If
                - ShouldCreateOIDCProvider
                - !Ref GitHubOIDCProvider
                - !If
                  - HasExistingOIDCProvider
                  - !Ref OIDCProviderArn
                  - !Sub >-
                    arn:${AWS::Partition}:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com
            Action: sts:AssumeRoleWithWebIdentity
            Condition:
              StringEquals:
                token.actions.githubusercontent.com:aud: sts.amazonaws.com
                token.actions.githubusercontent.com:sub: !Sub >-
                  repo:${GitHubOrg}/${GitHubRepo}:environment:${GitHubEnvironment}
      Policies:
        - PolicyName: CodeBuildAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Sid: AllowStartAndMonitorBuild
                Effect: Allow
                Action:
                  - codebuild:StartBuild
                  - codebuild:BatchGetBuilds
                Resource: !Sub >-
                  arn:${AWS::Partition}:codebuild:${AWS::Region}:${AWS::AccountId}:project/${CodeBuildProjectName}
              - Sid: AllowReadBuildLogs
                Effect: Allow
                Action:
                  - logs:GetLogEvents
                Resource: !Sub >-
                  arn:${AWS::Partition}:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/codebuild/${CodeBuildProjectName}:*
              - !If
                - HasArtifactBucket
                - Sid: AllowDownloadArtifacts
                  Effect: Allow
                  Action:
                    - s3:GetObject
                    - s3:ListBucket
                  Resource:
                    - !Sub 'arn:${AWS::Partition}:s3:::${ArtifactBucketName}'
                    - !Sub 'arn:${AWS::Partition}:s3:::${ArtifactBucketName}/*'
                - !Ref AWS::NoValue
              - !If
                - HasArtifactBucket
                - Sid: AllowDecryptArtifacts
                  Effect: Allow
                  Action:
                    - kms:Decrypt
                    - kms:GenerateDataKey
                  Resource: '*'
                  Condition:
                    StringEquals:
                      kms:ViaService: !Sub 's3.${AWS::Region}.amazonaws.com'
                - !Ref AWS::NoValue

Outputs:
  RoleArn:
    Description: >-
      Store this as the AWS_CODEBUILD_ROLE_ARN secret in the GitHub
      "codebuild" environment.
    Value: !GetAtt GitHubActionsCodeBuildRole.Arn

  OIDCProviderArn:
    Condition: ShouldCreateOIDCProvider
    Description: ARN of the created OIDC provider.
    Value: !Ref GitHubOIDCProvider
```

**Deploy with the artifact bucket from step 2:**

```bash
aws cloudformation deploy \
  --template-file administrators-guide-codebuild-role.yaml \
  --stack-name agent-plugins-gha-codebuild \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ArtifactBucketName=agent-plugins-build-artifacts-123456789012
```

**Deploy with an existing OIDC provider:**

```bash
aws cloudformation deploy \
  --template-file administrators-guide-codebuild-role.yaml \
  --stack-name agent-plugins-gha-codebuild \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    CreateOIDCProvider=false \
    OIDCProviderArn=arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com \
    ArtifactBucketName=agent-plugins-build-artifacts-123456789012
```

After deployment, note the `RoleArn` output — you will need it in step 5.

#### 4. Create the GitHub environment

**Using the GitHub CLI:**

```bash
# Look up the agent-plugins-admins team ID
TEAM_ID=$(gh api orgs/awslabs/teams/agent-plugins-admins --jq '.id')

# Create the environment with the admins team as required reviewers
gh api -X PUT repos/awslabs/agent-plugins/environments/codebuild \
  --input - <<EOF
{
  "prevent_self_review": true,
  "reviewers": [
    {"type": "Team", "id": ${TEAM_ID}}
  ]
}
EOF
```

**Using the GitHub UI:**

1. Go to **Settings > Environments > New environment**
2. Name: `codebuild`
3. Configure protection rules:
   - **Required reviewers**: add the `@awslabs/agent-plugins-admins` team
   - **Prevent self-review**: enable this checkbox

The non-self-approval requirement ensures that the person who triggers the workflow cannot approve their own run.

#### 5. Configure GitHub secrets and variables

In **Settings > Environments > codebuild**:

| Type   | Name                     | Value                                                       |
| ------ | ------------------------ | ----------------------------------------------------------- |
| Secret | `AWS_CODEBUILD_ROLE_ARN` | The `RoleArn` output from the CloudFormation stack (step 3) |

In **Settings > Secrets and variables > Actions > Variables**:

| Type     | Name                     | Value                                                            |
| -------- | ------------------------ | ---------------------------------------------------------------- |
| Variable | `AWS_REGION`             | AWS region where the CodeBuild project lives (e.g., `us-east-1`) |
| Variable | `CODEBUILD_PROJECT_NAME` | The `ProjectName` output from the CloudFormation stack (step 2)  |

`AWS_CODEBUILD_ROLE_ARN` is scoped to the `codebuild` environment so it is only available to jobs that have passed the approval gate. `AWS_REGION` and `CODEBUILD_PROJECT_NAME` are not sensitive and are stored as repo-level variables.

### Running the Workflow

1. Go to **Actions > Build (CodeBuild) > Run workflow**
2. Select the branch to build
3. A reviewer (not yourself) must approve the environment deployment
4. The build runs on CodeBuild and streams logs back to the Actions console

### Troubleshooting

| Symptom                                                   | Cause                        | Fix                                                                                                            |
| --------------------------------------------------------- | ---------------------------- | -------------------------------------------------------------------------------------------------------------- |
| "Not authorized to perform sts:AssumeRoleWithWebIdentity" | OIDC subject claim mismatch  | Verify the IAM trust policy `sub` condition matches `repo:awslabs/agent-plugins:environment:codebuild` exactly |
| "Could not find project"                                  | Wrong project name or region | Check `CODEBUILD_PROJECT_NAME` and `AWS_REGION` variables                                                      |
| Build hangs waiting for approval                          | No reviewer configured       | Add a required reviewer in the `codebuild` environment settings                                                |
| "Access denied" on StartBuild                             | IAM policy too restrictive   | Ensure the role has `codebuild:StartBuild` and `codebuild:BatchGetBuilds` on the correct project ARN           |
