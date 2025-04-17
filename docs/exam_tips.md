
# Exams tips

### Useful links
- https://academy.akuity.io/courses/gitops-argocd-intro 
Argo CD explanation and final useful Lab
- https://paulyu.dev/article/capa-study-guide/#argo-workflows-36 Good structured study guide
- https://killercoda.com/argoproj/course/argo-workflows Argo Workflows exercises

Trained memory on the configuration files syntax using ChatGPT (giving it the Official doc links and asking for a multiple choice quiz).

### Official Doc
- Argo CD: https://argo-cd.readthedocs.io/en/stable/
- Argo Workflows: https://argo-workflows.readthedocs.io/en/latest/
- Argo Rollouts: https://argo-rollouts.readthedocs.io/en/stable/
- Argo Events: https://argoproj.github.io/argo-events/

---
---

## Argo CD
Argo CD supports several different ways in which Kubernetes manifests can be defined:
- Kustomize applications
- Helm charts
- A directory of YAML/JSON/Jsonnet manifests, including Jsonnet.
- Any custom config management tool configured as a config management plugin

---

Argo CD can override values in the values.yaml parameters using argocd app set command, in the form of `-p PARAM=VALUE`. For example:

```bash
argocd app set helm-guestbook -p service.type=LoadBalancer
```

---

If unspecified, an application belongs to the default project, which is created automatically and by default, permits deployments from any source repo, to any cluster, and all resource Kinds. For example:

```yaml
spec:
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```
⚠️ Keep in mind that `!*` is an invalid rule, since it doesn't make any sense to disallow everything.

---

When we want ArgoCD to create the namespace automatically then we can also add labels and annotations to the namespace through `managedNamespaceMetadata`. For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  namespace: test
spec:
  syncPolicy:
    managedNamespaceMetadata:
      labels: # The labels to set on the application namespace
        any: label
        you: like
      annotations: # The annotations to set on the application namespace
        the: same
        applies: for
        annotations: on-the-namespace
    syncOptions:
    - CreateNamespace=true
```

---
---

## Argo Workflows

What will be printed?
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
  - name: diamond
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```
```bash
A
B
C
D
```
---
What will happen?

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-recursive-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    # flip a coin
    - - name: flip-coin
        template: flip-coin
    # evaluate the result in parallel
    - - name: heads
        template: heads                 # call heads template if "heads"
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails                     # keep flipping coins if "tails"
        template: coinflip
        when: "{{steps.flip-coin.outputs.result}} == tails"
  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        result = "heads" if random.randint(0,1) == 0 else "tails"
        print(result)
  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]
```

Print tails recursively until it comes out heads.

This can potentially recurse many times if "tails" keeps coming up.
However, Argo does not support infinite recursion, so eventually, you'll hit a limit:
- Max steps (e.g. maxParallelism)
- Stack overflow due to too many nested templates
- Or resource quota limits
---
What will happen to the artifacts?
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-gc-
spec:
  entrypoint: main
  artifactGC:
    strategy: OnWorkflowDeletion  # default Strategy set here applies to all Artifacts by default
  templates:
    - name: main
      container:
        image: argoproj/argosay:v2
        command:
          - sh
          - -c
        args:
          - |
            echo "can throw this away" > /tmp/temporary-artifact.txt
            echo "keep this" > /tmp/keep-this.txt
      outputs:
        artifacts:
          - name: temporary-artifact
            path: /tmp/temporary-artifact.txt
            s3:
              key: temporary-artifact.txt
          - name: keep-this
            path: /tmp/keep-this.txt
            s3:
              key: keep-this.txt
            artifactGC:
              strategy: Never   # optional override for an Artifact
```
The `artifactGC` at artifact level will override the global one and will persist the artifact `keep-this`.

---
How to deploy every pod to different nodes:
```yaml
# This example demonstrates the use of retry back offs
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-backoff-
spec:
  entrypoint: retry-backoff
  templates:
  - name: retry-backoff
    retryStrategy:
      limit: 10
      retryPolicy: "Always"
      backoff:
        duration: "1"      # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
        factor: 2
        maxDuration: "1m"  # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
      affinity:
        nodeAntiAffinity: {}
    container:
      image: python:alpine3.6
      command: ["python", -c]
      # fail with a 66% probability
      args: ["import random; import sys; exit_code = random.choice([0, 1, 1]); sys.exit(exit_code)"]
```
With `retryStrategy.affinity.nodeAntiAffinity` parameter the pod will not be re-deployed on the same node.

---
How to make workflow steps run in parallel:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: hello-hello-hello

  # This spec contains two templates: hello-hello-hello and print-message
  templates:
  - name: hello-hello-hello
    # Instead of just running a container
    # This template has a sequence of steps
    steps:
    - - name: hello1            # hello1 is run before the following steps
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2a           # double dash => run after previous step
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "hello2a"
      - name: hello2b           # single dash => run in parallel with previous step
        template: print-message
        arguments:
          parameters:
          - name: message
            value: "hello2b"

  # This is the same template as from the previous example
  - name: print-message
    inputs:
      parameters:
      - name: message
    container:
      image: busybox
      command: [echo]
      args: ["{{inputs.parameters.message}}"]
```

If single dash `-` then the step is parallel with the previous one.

---

What is a containerSet?

A ContainerSet template is similar to a normal container template, but it allows you to run multiple containers within a single Pod.

---

