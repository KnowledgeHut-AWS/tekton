# Pipeline Resources 

To define a configuration file for a PipelineResource, you can specify the following fields:

Required:

- `apiVersion` - Specifies the API version, for example tekton.dev/v1alpha1
- `kind` - Specify the PipelineResource resource object.
- `metadata` - Specifies data to uniquely identify the PipelineResource object, for example a name.
- `spec` - Specifies the configuration information for your PipelineResource resource object.
- `type` - Specifies the type of the PipelineResource

## Simple Resource Example

Let's create a resource for our Tekton Pipeline. Create a file with following content:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: git-resource-tag-
  namespace: tekton-pipelines
spec:
  taskSpec:
    resources:
      inputs:
      - name: skaffold
        type: git
    steps:
    - image: ubuntu
      script: cat skaffold/README.md
  resources:
    inputs:
    - name: skaffold
      resourceSpec:
        type: git
        params:
          - name: revision
            value: v0.32.0
          - name: url
            value: https://github.com/GoogleContainerTools/skaffold
```

Here we have a `TaskRun` which defines a git `PipelineResource`, create the TaskRun

`kubectl create -f whatever-you-named-it.yaml`

To see the results run

`tkn taskrun logs --last -f`

Let's understand what happneded here, ofoucrse we defined a resource undr a taksrun but let's understand the things under the hood:

1. We defined the resource `input` to be precise as we needed the __git repositroy__ code to do some work
2. We're passing params to the resource definiton to define details of the git repository we want to use

That makes this example, an example of __Input resources__, like source code (git) or artifacts, are dumped at path `/workspace/task_resource_name` within a mounted volume and are available to all steps of your `Task`. The path that the resources are mounted at can be overridden with the `targetPath` field. `Steps` can use the path __variable substitution__ key to refer to the local path to the mounted resource. 

Resources themselves can be referd as params as well as predefined variables such as path using the variable substitution syntax below where `<name>` is the resource's name and `<key>` is one of the resource's params:

__In Task Spec__

For an input resource in a `Task` spec: `$(resources.inputs.<name>.<key>)`

Or for an output resource: `$(outputs.resources.<name>.<key>)`

__In Condition Spec__

Input resources can be accessed by: `$(resources.<name>.<key>)`

__Accessing local path to resource__

The path key is predefined and refers to the local path to a resource on the mounted volume `$(resources.inputs.<name>.path)`

__Controlling where resources are mounted__

The optional field `targetPath` can be used to initialize a resource in a specific directory. If `targetPath` is set, the resource will be initialized under `/workspace/targetPath`. If `targetPath` is not specified, the resource will be initialized under `/workspace`. The following example demonstrates how git input repository could be initialized in `$GOPATH` to run tests:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-with-input
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
        targetPath: go/src/github.com/tektoncd/pipeline
  steps:
    - name: unit-tests
      image: golang
      command: ["go"]
      args:
        - "test"
        - "./..."
      workingDir: "/workspace/go/src/github.com/tektoncd/pipeline"
      env:
        - name: GOPATH
          value: /workspace/go
```

You can also override resource properties; for example when specifying input and output `PipelineResources`, you can optionally specify paths for each resource. paths will be used by `TaskRun` as the resource's new source paths i.e., copy the resource from a specified list of paths. `TaskRun` expects the folder and contents to be already present in specified paths. The paths feature could be used to provide extra files or altered version of existing resources before the execution of steps.

The output resource includes the name and reference to the pipeline resource and optionally paths. Paths will be used by TaskRun as the resource's new destination paths i.e., copy the resource entirely to specified paths. `TaskRun` will be responsible for the creation of required directories and content transition. The paths feature could be used to inspect the results of `TaskRun` after the execution of steps.

>The paths feature for input and output resources is heavily used to pass the same version of resources across tasks in context of `PipelineRun`.

In the following example, `Task` and `TaskRun` are defined with an input resource, output resource and step, which builds a war artifact. After the execution of `TaskRun`(volume-taskrun), custom volume will have the entire resource java-git-resource (including the war artifact) copied to the destination path /custom/workspace/.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: volume-task
  namespace: default
spec:
  resources:
    inputs:
      - name: workspace
        type: git
    outputs:
      - name: workspace
  steps:
    - name: build-war
      image: objectuser/run-java-jar #https://hub.docker.com/r/objectuser/run-java-jar/
      command: jar
      args: ["-cvf", "projectname.war", "*"]
      volumeMounts:
        - name: custom-volume
          mountPath: /custom
```

and

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: volume-taskrun
  namespace: default
spec:
  taskRef:
    name: volume-task
  resources:
    inputs:
      - name: workspace
        resourceRef:
          name: java-git-resource
    outputs:
      - name: workspace
        paths:
          - /custom/workspace/
        resourceRef:
          name: java-git-resource
  podTemplate:
    volumes:
      - name: custom-volume
        emptyDir: {}
```

__Resource Status__

When resources are bound inside a `TaskRun`, they can include extra information in the `TaskRun` `Status.ResourcesResult` field. This information can be useful for auditing the exact resources used by a `TaskRun` later. Currently the Image and Git resources use this mechanism.

For an example of what this output looks like:

```yaml
resourcesResult:
- key: digest
  value: sha256:a08412a4164b85ae521b0c00cf328e3aab30ba94a526821367534b81e51cb1cb
  resourceRef:
    name: skaffold-image-leeroy-web
```


## Optional Resources

By default, a resource is declared as mandatory unless optional is set to true for that resource. Resources declared as optional in a Task does not have be specified in TaskRun.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-check-optional-resources
spec:
  resources:
    inputs:
      - name: git-repo
        type: git
        optional: true
```

Similarly, resources declared as optional in a Pipeline does not have to be specified in PipelineRun.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-build-image
spec:
  resources:
    - name: workspace
      type: git
      optional: true
  tasks:
    - name: check-workspace
...
```

Similarly, resources declared as optional in a `Pipeline` does not have to be specified in `PipelineRun`.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: pipeline-build-image
spec:
  resources:
    - name: workspace
      type: git
      optional: true
  tasks:
    - name: check-workspace
...
```

> __Exercise__ create a task with Optional and mandatory resource and execute it via task run, both mandatory and optional resources should be output resources only, publish the output using the `tkn` CLI.