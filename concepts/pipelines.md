# Working with Pipelines

Let's create a simple Pipeline with multuple tasks and a Pipeline run to execute it

## Creating tasks

Our Pipeline multiplies and adds the parameters passed to it. We'll define a simple multiply and add task to achieve this

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: sum-params
  namespace: tekton-pipelines
  annotations:
    description: |
      A simple task that sums the two provided integers
spec:
  params:
  - name: a
    type: string
    default: "1"
    description: The first integer
  - name: b
    type: string
    default: "1"
    description: The second integer
  steps:
  - name: sum
    image: bash:latest
    script: |
      #!/usr/bin/env bash
      echo -n $(( "$(inputs.params.a)" + "$(inputs.params.b)" ))
```

and

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: multiply-params
  namespace: tekton-pipelines
  annotations:
    description: |
      A simple task that multiplies the two provided integers
spec:
  params:
  - name: a
    type: string
    default: "1"
    description: The first integer
  - name: b
    type: string
    default: "1"
    description: The second integer
  steps:
  - name: product
    image: bash:latest
    script: |
      #!/usr/bin/env bash
      echo -n $(( "$(inputs.params.a)" * "$(inputs.params.b)" ))
```

Now let's put the tasks together in a Pipeline

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-with-params
  namespace: tekton-pipelines
spec:
  params:
  - name: pl-param-x
    type: string
    default: "1"
  - name: pl-param-y
    type: string
    default: "1"
  tasks:
  - name: sum-params
    taskRef:
      name: sum-params
    params:
    - name: a
      value: "$(params.pl-param-x)"
    - name: b
      value: "$(params.pl-param-y)"
  - name: multiply-params
    taskRef:
      name: multiply-params
    params:
    - name: a
      value: "$(params.pl-param-x)"
    - name: b
      value: "$(params.pl-param-y)"
```

`kubectl apply -f pipeline-with-params -n tekton-pipelines`

Next we create cluster role and role-binding for this pipeline and pipeline-run

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: tekton-pipelines
  name: math-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "deployments.apps", "replicasets", "pods"]
  verbs: ["*"]
```

and 

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: math-service
  namespace: tekton-pipelines
```

finally,

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: math-role-binding
  namespace: tekton-pipelines
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tutorial-role
subjects:
  - kind: ServiceAccount
    name: math-service
    namespace: tekton-pipelines
```

With all this in place we can create a pipeline run to execute this pipeline and view resutls:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: pipelinerun-with-params
  namespace: tekton-pipelines
spec:
  params:
  - name: pl-param-x
    value: "100"
  - name: pl-param-y
    value: "500"
  pipelineRef:
    name: pipeline-with-params
```

`kubectl apply -f pipelinerun-with-params.yml`

You can monitor the execution of your PipelineRun in realtime as follows:

`tkn pipelinerun logs pipelinerun-with-params -f -n tekton-pipelines`

To view detailed information about your PipelineRun, use the following command:

`tkn pipelinerun describe pipelinerun-with-params -n tekton-pipelines`

## Pipeline with resources

In the real world you'll never really be lucky enough to create add and multiply pipelines, so let's look at the more realistc example. Here, we will clone source for a git repository and run a maven build.

