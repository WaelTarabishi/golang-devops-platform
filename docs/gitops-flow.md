# GitOps Flow

## What Makes This Repository GitOps

This project follows GitOps because Git is the source of truth for the application's desired runtime state.

The key deployment input is not a manual `kubectl` command and not a direct GitHub Actions deployment step. Instead, the desired state is defined in version-controlled files:

- the Helm chart in `helm/go-web-app-chart`
- the image tag inside `helm/go-web-app-chart/values.yaml`
- the Argo CD application definition in `argocd/application.yaml`

When those files change in Git, Argo CD uses them to reconcile the EKS cluster.

## Source of Truth

In a GitOps model, the repository describes what should be running. In this project, Git contains:

- the application source
- the container build definition
- the Helm deployment configuration
- the version of the container image to run

That means a reviewer can inspect Git history and understand exactly what version was intended for deployment and when that desired state changed.

## What GitHub Actions Does

GitHub Actions is used here for continuous integration and artifact publication. It:

- builds the Go application
- runs tests
- builds the Docker image
- pushes the image to Docker Hub
- updates the Helm chart image tag in `values.yaml`
- commits that updated desired state back to Git

This is an important boundary: GitHub Actions prepares the release and updates Git, but it does not directly deploy to Amazon EKS.

## What GitHub Actions Does Not Do

GitHub Actions does not:

- call `kubectl apply`
- run a direct Helm install or upgrade against the EKS cluster
- make the cluster the deployment source of truth

That distinction is what keeps the workflow aligned with GitOps instead of a traditional push-based CD pipeline.

## What Argo CD Does

Argo CD is responsible for deployment. It watches the repository, detects changes to the Helm chart path, renders the chart, and synchronizes the resulting manifests into Amazon EKS.

Its job is continuous reconciliation:

- compare desired state in Git
- compare live state in EKS
- bring the cluster back to the desired state when drift or updates are detected

In this repository, that means a new image tag committed by GitHub Actions becomes deployable only after Argo CD observes the Git change and syncs it.

## Continuous Reconciliation in Practice

The operational loop looks like this:

1. A developer pushes code.
2. GitHub Actions validates the code and publishes a new Docker image.
3. GitHub Actions updates the Helm image tag and commits the change.
4. Argo CD detects that commit.
5. Argo CD renders the Helm chart and syncs the application to EKS.
6. Kubernetes performs a rolling update to the new image version.

Because Argo CD keeps reconciling over time, the cluster is not updated only once. It is continuously checked against the repository. If drift appears, Argo CD can correct it to match Git again.

## Why This Matters

This model gives the platform several advantages:

- clear audit history in Git
- better separation between CI and deployment control
- repeatable and declarative cluster updates
- easier rollback through Git history
- reduced risk from manual cluster changes

For engineers, this demonstrates a sound operating model. For recruiters and reviewers, it shows that the project is more than a containerized app: it is a complete GitOps delivery workflow built around versioned infrastructure and controlled deployment automation.
