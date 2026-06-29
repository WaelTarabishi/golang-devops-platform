# Architecture Overview

## Summary

This repository demonstrates a GitOps-based CI/CD workflow for a Go web application running on Amazon EKS. The delivery model separates responsibilities cleanly:

- GitHub Actions handles build, test, containerization, and repository updates.
- Docker Hub stores versioned application images.
- Helm defines the Kubernetes deployment configuration.
- Argo CD watches Git and performs cluster synchronization.
- Amazon EKS runs the application as a Kubernetes workload.

This pattern is useful for both learning and production-style portfolio work because it shows traceability from source code to live infrastructure.

## End-to-End Workflow

The workflow in this project is:

1. A developer pushes code changes to the `main` branch.
2. GitHub Actions starts the CI pipeline from `.github/workflows/ci.yaml`.
3. The pipeline builds the Go application and runs tests.
4. If CI succeeds, the pipeline builds a Docker image.
5. The image is pushed to Docker Hub with a unique tag.
6. GitHub Actions updates `helm/go-web-app-chart/values.yaml` with the new image tag.
7. The updated Helm chart is committed back to the Git repository.
8. Argo CD detects the Git change because it watches the Helm chart path in this repository.
9. Argo CD renders the Helm chart into Kubernetes manifests.
10. Argo CD syncs the rendered manifests to the Amazon EKS cluster.
11. Kubernetes performs a rolling update so the new application version replaces the previous version gradually.

## Component Responsibilities

### 1. Developer and GitHub Repository

The repository holds the application source code, Dockerfile, Helm chart, workflow definition, and Argo CD manifest. This makes the project easy to review because the application and the deployment configuration live together in version control.

### 2. GitHub Actions

GitHub Actions provides the automation for continuous integration:

- checks out the repository
- builds the Go application
- runs the test suite
- builds the Docker image
- pushes the image to Docker Hub
- updates the Helm chart image tag in Git

Its role stops at updating the desired state in the repository. It does not talk directly to Amazon EKS.

### 3. Docker Hub

Docker Hub acts as the container image registry. Each successful pipeline run publishes an image that can later be deployed by Kubernetes through the Helm chart.

This keeps application artifacts versioned and separate from the cluster itself.

### 4. Helm

Helm packages the Kubernetes configuration for the Go application. In this repository, the chart lives under `helm/go-web-app-chart`.

The image tag in `values.yaml` is the key handoff point between CI and GitOps:

- CI updates the image tag
- Git stores that change
- Argo CD notices the chart change
- EKS receives the updated workload definition

### 5. Argo CD

Argo CD is the deployment controller. It watches the Git repository for changes to the Helm chart path and continuously compares:

- desired state in Git
- live state in the EKS cluster

When the repository changes, Argo CD renders the Helm chart and synchronizes the cluster to match.

### 6. Amazon EKS

Amazon EKS runs the Kubernetes control plane and worker nodes that host the application. Once Argo CD syncs a new Helm release, Kubernetes creates or updates the deployment and service resources required to run the Go web application.

## Operational Outcome

The practical result is a controlled release pipeline:

- application changes are validated in CI
- deployable artifacts are published to a registry
- the desired deployment state is recorded in Git
- Argo CD applies that state to EKS
- Kubernetes performs the rollout

That gives teams auditability, rollback visibility, and a clearer operational model than a pipeline that deploys straight from CI into the cluster.
