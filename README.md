# ee-aws-kubernetes

AWX execution environment for Kubernetes and Amazon EKS automation.

## Purpose

This image is used by AWX job templates that need to talk directly to the Kubernetes API or EKS control plane.

It is separate from `ee-aws-ssm`. The SSM execution environment is for private EC2 host management. This image is for Kubernetes, Helm, and EKS operations.

## Included Runtime

- `ansible-core`
- `ansible-runner`
- AWS CLI v2
- `kubectl`
- Helm
- `amazon.aws`
- `community.aws`
- `kubernetes.core`
- `boto3`
- `botocore`
- Python Kubernetes client

## Repository Contract

```text
execution-environment.yml   # ansible-builder definition
requirements.yml            # Ansible collections
requirements.txt            # Python dependencies
bindep.txt                  # intentionally empty to avoid public OS mirror dependency
.gitlab-ci.yml              # consumes shared EE pipeline
```

## Pipeline

This repo uses the central shared pipeline:

```yaml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: ansible/ee.yml
```

Pipeline behavior:

```text
validate source
  -> build image with ansible-builder
  -> smoke test collections, executables, and Python imports
  -> push GitLab Container Registry image
  -> manually register AWX Execution Environment on protected semantic tags
```

## AWX Registration

The shared pipeline registers this EE as:

```text
ee-aws-kubernetes
```

Expected AWX settings:

```text
Image: registry.midhtech.local:5050/infra_team/automation/ee-aws-kubernetes:v1.0.0
Pull: Always
Registry credential: gitlab-container-registry
```

Required protected CI/CD variables for the manual registration job:

```text
AWX_USERNAME
AWX_PASSWORD
```

The GitLab registry credential must already exist in AWX:

```text
Name: gitlab-container-registry
Credential Type: Container Registry
Authentication URL: registry.midhtech.local:5050
Username: <GitLab deploy token username>
Password: <GitLab deploy token with read_registry>
```

## Runtime Model

EKS jobs should authenticate through AWS IAM and generate kubeconfig at job runtime:

```bash
aws eks update-kubeconfig \
  --region us-east-1 \
  --name dev-midh-eks \
  --role-arn arn:aws:iam::<account-id>:role/dev-midh-eks-platform-admin-role
```

This avoids storing static kubeconfig content in AWX. The AWX job template should attach:

- central base AWS credential
- AWS assume-role profile credential for the EKS platform admin role

## Consumers

Primary consumer:

```text
infra_team/automation/awx-eks-operations
```

That repo owns EKS validation and controlled cluster operations playbooks.
