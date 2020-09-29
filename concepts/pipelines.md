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

Next, we create cluster role and role-binding for this pipeline and pipeline-run

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

You can monitor the execution of your `PipelineRun` in realtime as follows:

`tkn pipelinerun logs pipelinerun-with-params -f -n tekton-pipelines`

To view detailed information about your `PipelineRun`, use the following command:

`tkn pipelinerun describe pipelinerun-with-params -n tekton-pipelines`

## Pipeline with resources

In the real world you'll never really be lucky enough to create add and multiply pipelines, so let's look at the more realistic example. Here, we will clone source for a git repository and run custom tasks on it:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: cat-branch-readme
  namespace: tekton-pipelines
spec:
  params:
  - name: repo-url
    type: string
    description: The git repository URL to clone from.
  - name: branch-name
    type: string
    description: The git branch to clone.
  workspaces:
  - name: shared-data
    description: |
      This workspace will receive the cloned git repo and be passed
      to the next Task for the repo's README.md file to be read.
  tasks:
    - name: fetch-repo
      taskRef:
        name: git-clone
      workspaces:
      - name: output
        workspace: shared-data
      params:
        - name: url
          value: $(params.repo-url)
        - name: revision
          value: $(params.branch-name)
    - name: cat-readme
      runAfter: ["fetch-repo"] # Wait until the clone is done before reading the readme.
      workspaces:
        - name: source
          workspace: shared-data
      taskSpec:
        workspaces:
        - name: source
        steps:
          - image: zshusers/zsh:4.3.15
            script: |
              #!/usr/bin/env zsh
              cat $(workspaces.source.path)/readme.md
  ```

This example takes a git repository and a branch name and prints the __`readme.md`__ file from that branch. This is an example Pipeline demonstrating the following:

- Using the `git-clone` catalog Task to clone a branch
- Passing a cloned repo to subsequent Tasks using a Workspace
- Ordering `Tasks` in a `Pipeline` using __`runAfter`__ so that `git-clone` completes before we try to read from the Workspace
- Using a `volumeClaimTemplate` Volume as a Workspace
- Avoiding hard-coded paths by using a Workspace's path variable instead

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: git-clone-checking-out-a-branch
  namespace: tekton-pipelines
spec:
  podTemplate:
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: "tekton.dev/pipelineRun"
              operator: In
              values:
              - git-clone-checking-out-a-branch
          topologyKey: kubernetes.io/hostname
  pipelineRef:
    name: cat-branch-readme
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Mi
  params:
  - name: repo-url
    value: https://github.com/KnowledgeHut-AWS/katacoda-labs.git
  - name: branch-name
    value: master
```

We could also use a `persistantVolumeClaim` instead of `volumeClaimTemplate`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cat-branch-readme-storage
spec:
  resources:
    requests:
      storage: 10Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

followed by replacing this section in `PipelineRun`

```yaml
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Mi
```

with

```yaml
workspaces:
- name: myworkspace
  persistentVolumeClaim:
    claimName: cat-branch-readme-storage
  subPath: some-subdir
```

Note that `subPath` isn't mandatory here.

## Runing Parallel Tasks in a Pipeline

Running Parallel tasks can come in really handy in Tekton Pipelines, suppose you're running __Gatling__ or __JMeter__ tests on a spring boot microservice for performance evaluation and also have to run __Spring Cloud Contract__ tests for it's __Consumer Driven Contracts Approach__ implementation, since they're idependent of each other you don't have to run these tests in sequential order; running Parallel Tasks comes in very handy in managing such independent or long running tasks in a pipeline.

A `PipelineRun` will pass a message parameter to the `Pipeline` in this example. The `STARTER` task will write the message to a file in the workspace. The `UPPER` and `LOWER` tasks will execute in parallel and process the message written by the `STARTER`, and transform it to upper case and lower case. The `REPORTER` task is will use the Task Result from the UPPER task and print it - it is intended to mimic a Task that sends data to an external service and shows a `Task` that doesn't use a workspace. The `VALIDATOR` task will validate the result from `UPPER` and `LOWER`.

We will use the `runAfter` property in a `Pipeline` to configure that a task depend on another task. Output can be shared both via Task Result (e.g. like REPORTER task) or via files in a workspace.

```bash
             -- (upper) -- (reporter)
           /                         \
  (starter)                           (validator)
           \                         /
             -- (lower) ------------
```

We first create the starter task

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: persist-param
  namespace: tekton-pipelines
spec:
  params:
    - name: message
      type: string
  results:
    - name: message
      description: A result message
  steps:
    - name: write
      image: ubuntu
      script: echo $(params.message) | tee $(workspaces.task-ws.path)/message $(results.message.path)
  workspaces:
    - name: task-ws
```

and the other tasks,

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: to-upper
  namespace: tekton-pipelines
spec:
  description: |
    This task read and process a file from the workspace and write the result
    both to a file in the workspace and as a Task Result.
  params:
    - name: input-path
      type: string
  results:
    - name: message
      description: Input message in upper case
  steps:
    - name: to-upper
      image: ubuntu
      script: cat $(workspaces.w.path)/$(params.input-path) | tr '[:lower:]' '[:upper:]' | tee $(workspaces.w.path)/upper $(results.message.path)
  workspaces:
    - name: w
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: to-lower
  namespace: tekton-pipelines
spec:
  description: |
    This task read and process a file from the workspace and write the result
    both to a file in the workspace and as a Task Result
  params:
    - name: input-path
      type: string
  results:
    - name: message
      description: Input message in lower case
  steps:
    - name: to-lower
      image: ubuntu
      script: cat $(workspaces.w.path)/$(params.input-path) | tr '[:upper:]' '[:lower:]' | tee $(workspaces.w.path)/lower $(results.message.path)
  workspaces:
    - name: w
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: result-reporter
  namespace: tekton-pipelines
spec:
  description: |
    This task is supposed to mimic a service that post data from the Pipeline,
    e.g. to an remote HTTP service or a Slack notification.
  params:
    - name: result-to-report
      type: string
  steps:
    - name: report-result
      image: ubuntu
      script: echo $(params.result-to-report)
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: validator
  namespace: tekton-pipelines
spec:
  steps:
    - name: validate-upper
      image: ubuntu
      script: cat $(workspaces.files.path)/upper | grep HELLO\ TEKTON
    - name: validate-lower
      image: ubuntu
      script: cat $(workspaces.files.path)/lower | grep hello\ tekton
  workspaces:
    - name: files
```

Followed by the pipeline

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: parallel-pipeline
  namespace: tekton-pipelines
spec:
  params:
    - name: message
      type: string

  workspaces:
    - name: ws

  tasks:
    - name: starter          # Tasks that does not declare a runAfter property
      taskRef:               # will start execution immediately
        name: persist-param
      params:
        - name: message
          value: $(params.message)
      workspaces:
        - name: task-ws
          workspace: ws
          subPath: init

    - name: upper
      runAfter:               # Note the use of runAfter here to declare that this task
        - starter             # depends on a previous task
      taskRef:
        name: to-upper
      params:
        - name: input-path
          value: init/message
      workspaces:
        - name: w
          workspace: ws

    - name: lower
      runAfter:
        - starter
      taskRef:
        name: to-lower
      params:
        - name: input-path
          value: init/message
      workspaces:
        - name: w
          workspace: ws

    - name: reporter          # This task does not use workspace and may be scheduled to
      runAfter:               # any Node in the cluster.
        - upper
      taskRef:
        name: result-reporter
      params:
        - name: result-to-report
          value: $(tasks.upper.results.message)  # A result from a previous task is used as param

    - name: validator         # This task validate the output from upper and lower Task
      runAfter:               # It does not strictly depend on the reporter Task
        - reporter            # But you may want to skip this task if the reporter Task fail
        - lower
      taskRef:
        name: validator
      workspaces:
        - name: files
          workspace: ws
```

and finally the `PipelineRun`

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: parallel-pipelinerun-
  namespace: tekton-pipelines
spec:
  params:
    - name: message
      value: Hello Tekton
  pipelineRef:
    name: parallel-pipeline
  workspaces:
    - name: ws
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Mi
```

When you run `tkn pipelinerun describe parallel-pipelinerun-b7vx8 -n tekton-pipelines` a few times you'll notice an output similar to

```bash
Name:           parallel-pipelinerun-b7vx8
Namespace:      tekton-pipelines
Pipeline Ref:   parallel-pipeline

🌡️  Status

STARTED          DURATION   STATUS
34 seconds ago   ---        Running

📦 Resources

 No resources

⚓ Params

 NAME        VALUE
 ∙ message   Hello Tekton

🗂  Taskruns

 NAME                                         TASK NAME   STARTED          DURATION     STATUS
 ∙ parallel-pipelinerun-b7vx8-lower-7vlrz     lower       7 seconds ago    ---          Running
 ∙ parallel-pipelinerun-b7vx8-upper-d644d     upper       7 seconds ago    ---          Running
 ∙ parallel-pipelinerun-b7vx8-starter-58q2f   starter     34 seconds ago   27 seconds   Succeeded
```

Thats the lower and upper tasks runiing in parallel.
