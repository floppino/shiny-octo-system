# Argo CD
Notes for the Certified Argo Project Associate (CAPA).

## Continuous Delivery
Continuous Delivery (CD) is an organization's __technological__ and cultural __transformation__.

- __Codifying__ the specialized knowledge from both devs and ops, freeing them from the toil of releases to focus on delivering value.
- __Visibility__ is improved by codifying.
- __Speeding up__ the delivery of value to the customer by automating the release of code changes.
- __Scalability__ of CD is well implemented through improvements to the process that will persist well into the future.

### Continuous Integration
Continuous Integrations (CI) is an essential prerequisite for CD.
CI is regularly merging code into a centralized branch.

### Continuous Deployment
- __Continuous Delivery__ automates deploying a build artifact into an environment. This practice does not account for the promotion between environments and the production release requires for a human to approve it manually.
- __Continuous Deployment__ is the practice of automating the entire deployment lifecycle for an application without any human intervention.

Getting to Continuous Deployment requires the complete incorporation of CI and CD (Delivery).

## GitOps

Four OpenGitOps principles:

1. __Declarative__: Desired state must be expressed declaratively.
2. __Versioned and immutable__: Desired state is stored in a way that supports versioning, immutability, and a complete version history.
3. __Pulled automatically__: Software agents automatically pull the desired state declarations from the source.
4. __Continuously reconciled__: Software agents continuously observe actual system state and attempt to apply the desired state.

In a sentence: __GitOps__ is declaratively storing the desired state with immutable versions and using a software agent to reconcile the live state with the desired state.

### Pull vs. Push model

Push model is 'imperative', here its disadvantages:
- Credentials with write access and direct access to the cluster's API are needed.
- Reproducing the cluster's state would require rerunning the whole workflow.
- Changes to the workflow's steps require detailed consideration and meticulous testing.

Pull model is 'declarative', here its advantages:
- __Repeatability__: Deployments are repeatability, and environments are easy to recreate.
- __Collaboration__: Changes are made through Pull Requests.
- __Visibility__: Commit history shows how the desired state changed and why.
- __Consistency__: Changes to the collective desired state of the cluster must first be integrated into the environment configuration repository.
- __Security__: The workflow does not have credentials with write access or direct access to the cluster. Changes are applied using a service operating withing your infrastructure (or cluster).

### GitOps and CD
GitOps is a framework for managing the state of environments.
CD focuses on the automation of the changes.

## Argo CD

__Argo CD__ is an extension of Kubernetes. It is part of the Argo Project, a suit of open-source tools for managing cluster, running workflows and executing complex deployment strategies.

### Argo CD and GitOps

Argo CD acts as the software agent that automatically pulls the desired state from the specified source and continuously reconciles it with the live state of the cluster.

### Argo CD and CD

Argo CD is responsible for deploying the application and its configuration to Kubernetes.

The Argo CD API/CLI can be implemented into workflows to trigger the deployment.

### Argo CD and Kubernetes
Argo CD and its configuration are represented by Custom Resource Definitions (CRDs) in Kubernetes.

After the initial bootstrapping Argo CD can then manage itself the same way it would any other Kubernetes resource.

Argo CD supports a variety of architectures including multi-cluster environments.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    syncOptions:
    - CreateNamespace: true
    automated:
      selfHeal: true
```

## The Argo CD application resource

Application CRD is the main tool Argo CD uses to declaratively define the deployment process for manifests into Kubernetes. It declares where to source the manifests, how to render them, when to deploy the resources, when to reconcile the live state with the desired state and much more.

### Config management tools
Argo CD has built-in support for common config management tools (Helm, Kustomize...) and supports plain yaml. 

You can also integrate any tool  into Argo CD using a **config management plugin (CMP)**.

If supported tool detection happens automatically based on the files found in the source repository path.

For Git repository the targetRevision can be the branch or tag or a pinned commit. For Helm chart repository the targetRevision parameter will be the chart version.

The *destination* parameter indicates the Kubernetes cluster and will have either a name or server URL, and the namespace in it.

### Application health

Five health statuses for resources:
- **Healthy**: resource running as expected
- **Progressing**: resource is being created or updated
- **Degraded**: resource has been created but it has errors (ex. pod with wrong image)
- **Missing**: resource still not created (ex. added application but synch is manual)
- **Suspended**: resource is waiting for an external event (ex. manual pause of a deployment via kubectl)
