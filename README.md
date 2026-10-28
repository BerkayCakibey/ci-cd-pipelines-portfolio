# GitHub Actions CI/CD Pipelines – Portfolio

This repository contains **template-based CI/CD pipelines** designed with reusability, modularity, and security in mind. All workflows are built using GitHub Actions and follow AWS best practices for deploying Dockerized applications via ECR and ECS.

These templates reflect the architecture I used in previous roles, where they cut deployment time substantially in the projects they came from.

---

## 📁 Folder Structure

- `monorepo/` – Multi-service architecture with change detection
- `multi-env/` – Branch-based (dev/prod) deployments; the Bun and Next.js variants add build caching
- `single-repo/` – Minimal pipelines for isolated services

Copy the workflow files into `.github/workflows/` of the target repository. The monorepo callers reference `./.github/workflows/Monorepo-CICD-ECS-Deployment.yaml`, so the reusable workflow must keep that path and name.

---

## 🔧 Key Features

- ✅ **Reusable Templates**: Each workflow is designed as a portable template, copy-paste ready for different projects.
- 🔒 **OIDC-Based AWS Access**: Workflows assume an IAM role through GitHub's OIDC provider. No AWS access keys are stored in GitHub. Non-secret settings (ECR repository, cluster, service, container name) are GitHub environment variables, not secrets. Every workflow declares least-privilege `permissions:`.
- 🧱 **Modular Build Steps**: Built primarily using `uses:` syntax from GitHub’s official and community actions, instead of manual shell commands.
- 🔁 **ECS Deployments**: Updates ECS task definitions and waits for service stability; availability depends on service capacity, health checks, and deployment settings.
- 🏷️ **Unique Image Tags**: Docker images are tagged `<commit SHA>-<run ID>-<run attempt>` instead of `latest`, so every build is traceable to its commit and workflow run and a push never overwrites an earlier image.
- ⏳ **Serialized Deployments**: A `concurrency` group per repository and environment (per service in the monorepo) queues overlapping deployment runs instead of running them in parallel; in-progress runs are never cancelled.

---

## 🚀 Included Pipelines

### 1. Single Repo – ECS Deployment

- Detects changes on the `main` branch
- Builds Docker image from project root
- Pushes image to Amazon ECR
- Updates ECS task definition
- Deploys the new task definition to the ECS service and waits for service stability

📄 [`single-repo/Single-Repo-CICD-ECS-Deployment.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/single-repo/Single-Repo-CICD-ECS-Deployment.yaml)

---

### 2. Multi-Environment – Dev/Prod ECS Deployments

- Deploys based on the target branch (`main` → `production` environment, `dev` → `dev` environment)
- Separate ECR repositories, task definitions, and ECS services per environment, read from the environment's variables
- The Bun and Next.js variants use Docker build caching for faster deployment
- Ideal for frontend/backend workflows where build size matters

📄 [`multi-env/Multi-Environment-CICD-ECS-Deployment.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/multi-env/Multi-Environment-CICD-ECS-Deployment.yaml)  
📄 [`multi-env/Multi-Environment-CICD-Next.js-ECS-Deployment.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/multi-env/Multi-Environment-CICD-Next.js-ECS-Deployment.yaml)  
📄 [`multi-env/Multi-Environment-CICD-Bun-ECS-Deployment.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/multi-env/Multi-Environment-CICD-Bun-ECS-Deployment.yaml)

---

### 3. Monorepo – Change Detection & Reusable Workflow

- Detects which specific folder/service was changed (`services/<name>/**`)
- Triggers deployment only for affected service(s)
- Built using centralized reusable workflows
- Two alternative strategies available:
  - **Static Job-Based**: Each service has its own deployment job with conditional logic
  - **Matrix-Based**: Builds the matrix from the change detection output (`fromJSON`), so only changed services get a deploy job
- Easily extensible for large microservice setups

📄 [`monorepo/Monorepo-CICD-Change-Detection.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/monorepo/Monorepo-CICD-Change-Detection.yaml)  
📄 [`monorepo/Monorepo-CICD-ECS-Deployment.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/monorepo/Monorepo-CICD-ECS-Deployment.yaml)  
📄 [`monorepo/Monorepo-CICD-Matrix-Change-Detection.yaml`](https://github.com/BerkayCakibey/ci-cd-pipelines-portfolio/blob/main/monorepo/Monorepo-CICD-Matrix-Change-Detection.yaml)

---

## 🔑 AWS Setup (OIDC)

The workflows authenticate to AWS with GitHub's OIDC provider via `aws-actions/configure-aws-credentials`. No long-lived AWS access keys are needed.

### 1. OIDC identity provider

Create one IAM OIDC identity provider per AWS account:

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 2. IAM role per environment

Create one role per GitHub environment (`production`, `dev`). Trust policy for the `production` role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:BerkayCakibey/<repo>:environment:production"
        }
      }
    }
  ]
}
```

Replace `<ACCOUNT_ID>` and `<repo>`. For the `dev` role use `repo:BerkayCakibey/<repo>:environment:dev`.

The role's permission policy needs:

- ECR: `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`, `ecr:PutImage`, `ecr:BatchGetImage`
- ECS: `ecs:DescribeTaskDefinition`, `ecs:RegisterTaskDefinition`, `ecs:DescribeServices`, `ecs:UpdateService`
- IAM: `iam:PassRole` for the task execution role and task role referenced in the task definition

### 3. GitHub environments and variables

Create the environments `production` and `dev` (`single-repo/` and `monorepo/` use only `production`). Set these as environment variables (Settings > Environments > Variables), not secrets:

| Variable | Used by | Example |
| --- | --- | --- |
| `AWS_ROLE_ARN` | all | `arn:aws:iam::123456789012:role/github-deploy-production` |
| `AWS_REGION` | all | `eu-central-1` |
| `ECR_REPOSITORY` | single-repo, multi-env | `example-repository` |
| `TASK_DEFINITION_FAMILY` | single-repo, multi-env | `example-task-def` |
| `ECS_CLUSTER_NAME` | single-repo, multi-env | `example-cluster` |
| `ECS_SERVICE_NAME` | single-repo, multi-env | `example-service` |
| `CONTAINER_NAME` | single-repo, multi-env | `example-container` |

The monorepo reusable workflow derives the ECR repository from its `service_name` input and looks up the task definition family, service and container names in the `SERVICE_MAP` table at the top of the workflow; add one row there per new service. The AWS account ID is not needed: the registry host comes from the `amazon-ecr-login` output.

### 4. Workflow permissions

Every workflow sets `permissions:` with `id-token: write` (required for OIDC) and `contents: read`. In the monorepo callers the deploy jobs grant these to the reusable workflow explicitly, because a called workflow cannot elevate the permissions it receives from the caller.

---

## 📌 Notes

- 🚫 **Sensitive Data Masked**: All names, repository paths, and container IDs have been altered to prevent any data leakage.
- ♻️ **Easily Reusable**: You can reuse any pipeline by replacing the service name, ECR repo, ECS settings, and GitHub environment variables.
- ⚙️ **Built for Production**: All pipelines have been tested and used in real deployment scenarios with active ECS clusters and services.
- 🔁 **Rollback**: Each deploy registers a new task definition revision and pushes a uniquely tagged image (`<commit SHA>-<run ID>-<run attempt>`). Rollback requires retained images and task-definition revisions. These templates do not configure automatic rollback.
- **Action versions**: pinned to the current major releases, checked against each action's latest release on 27 Oct 2026: `actions/checkout@v7`, `actions/cache@v6`, `actions/setup-node@v7`, `oven-sh/setup-bun@v2`, `aws-actions/configure-aws-credentials@v6`, `aws-actions/amazon-ecr-login@v2`, `aws-actions/amazon-ecs-render-task-definition@v1`, `aws-actions/amazon-ecs-deploy-task-definition@v2`, `docker/setup-buildx-action@v4`, `docker/build-push-action@v7`, `dorny/paths-filter@v4`.

## License

MIT, see [LICENSE](LICENSE).

---

Berkay Cakibey · Open to B2B/Contract Cloud & DevOps opportunities. Let's connect on https://www.linkedin.com/in/berkay-cakibey/, visit [berkaycakibey.com](https://berkaycakibey.com) or reach out via hello@berkaycakibey.com.

---

Thanks for visiting the CI/CD portfolio!  
Feel free to fork, copy, or reach out for improvements or collaboration.
