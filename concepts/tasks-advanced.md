# More on tasks

We'll now look at more advanced unage of tasks in this section.

## Running Step Containers as a Non Root User

*All steps that do not require to be run as a root user should make use of `TaskRun` features to designate the container for a step runs as a user without root permissions*. As a best practice, running containers as non root should be built into the container image to avoid any possibility of the container being run as root. However, as a further measure of enforcing this practice, steps can make use of a securityContext to specify how the container should run.

An example of running Task steps as a non root user is shown below:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: show-non-root-steps
spec:
  steps:
    # no securityContext specified so will use
    # securityContext from TaskRun podTemplate
    - name: show-user-1001
      image: ubuntu
      command:
        - ps
      args:
        - "aux"
    # securityContext specified so will run as  
    # user 2000 instead of 1001
    - name: show-user-2000
      image: ubuntu
      command:
        - ps
      args:
        - "aux"
      securityContext:
        runAsUser: 2000
```

Create the task `kubectl apply -f non-root-task.yml -n tekton-pipelines`, add the TaskRun

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: show-non-root-steps-run-
spec:
  taskRef:
    name: show-non-root-steps
  podTemplate:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1001
```

`kubectl apply -f non-root-task-run.yml -n tekton-pipelines`

Check run status `tkn taskrun logs --last -f -n tekton-pipelines`

## Cluster Task

You can create cluster tasks by simply repalcing the nomenclature in a task defintion

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: clustertask-v1beta1
spec:
  steps:
  - image: ubuntu
    script: echo hello
```

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: clustertask-
spec:
  taskRef:
    name: clustertask-v1beta1
    kind: ClusterTask
```

## Passing results of a task to another

This can be done by emitting results.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: print-date
  annotations:
    description: |
      A simple task that prints the date
spec:
  results:
    - name: current-date-unix-timestamp
      description: The current date in unix timestamp format
    - name: current-date-human-readable
      description: The current date in human readable format
  steps:
    - name: print-date-unix-timestamp
      image: bash:latest
      script: |
        #!/usr/bin/env bash
        date +%s | tee $(results.current-date-unix-timestamp.path)
    - name: print-date-human-readable
      image: bash:latest
      script: |
        #!/usr/bin/env bash
        date | tee $(results.current-date-human-readable.path)
```

>__Note__: The maximum size of a Task's results is limited by the container termination message feature of Kubernetes, as results are passed back to the controller via this mechanism. At present, the limit is "4096 bytes". So don't go overbaord with it!

## Using custom volumes

Let's create a dummy volume first

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-for-testing-volume-args
data:
  test.data: tasks are my jam
```

Followed by a `TaskRun` that reads data from this volume

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: task-volume-args-
spec:
  taskSpec:
    params:
    - name: CFGNAME
      description: Name of config map
      type: string
    steps:
    - name: read
      image: ubuntu
      script: |
        #!/usr/bin/env bash
        cat /configmap/test.data
      volumeMounts:
      - name: custom
        mountPath: /configmap
    volumes:
    - name: custom
      configMap:
        name: "$(params.CFGNAME)"
  params:
  - name: CFGNAME
    value: config-for-testing-volume-args
```

You can then run them with

`kubectl create -f custom-vol-tasks-run.yml -n tekton-pipelines`

`tkn taskrun logs --last -f -n tekton-pipelines`

You can also use a `TaskRun` with embedded task using a custom volume

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: workspace-volume-
spec:
  taskSpec:
    steps:
    - name: write
      image: ubuntu
      script: echo some stuff > /workspace/stuff
    - name: read
      image: ubuntu
      script: cat /workspace/stuff

    - name: override-workspace
      image: ubuntu
      # /workspace/stuff *doesn't* exist.
      script: |
        #!/usr/bin/env bash
        [[ ! -f /workspace/stuff ]]
      volumeMounts:
      - name: empty
        mountPath: /workspace

    volumes:
    - name: empty
      emptyDir: {}
```

`kubectl create -f custom-vol-taskrun-embed.yml -n tekton-pipelines`

`tkn taskrun logs --last -f -n tekton-pipelines`

You can enforce a workspace to be read only by using:

```yaml
    workspaces:
    - name: write-disallowed
      readOnly: true
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kh-pvc-2
spec:
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
```

> __Exercise:__ Create a Task or TaskRun that uses the above workspace as both ReadWrite and ReadOnly in different steps.

## Using Resources

> __Note__: Resources are depcreated, use with caution, most of the resources are replaced by Catalog tasks now

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
            value: master
          - name: url
            value: https://github.com/GoogleContainerTools/skaffold
```

Here we have a `TaskRun` which defines a git `PipelineResource`, create the TaskRun

`kubectl create -f whatever-you-named-it.yaml`

To see the results run

`tkn taskrun logs --last -f`

Let's understand what happened here, of-course we defined a resource under a `TaskRun` but let's understand the things under the hood:

1. We defined the resource, `input` to be precise, as we needed the __git repositroy__ code to do some work
2. We're passing params to the resource definiton to define details of the git repository we want to use

Let's see how this resource is replacs by a catalogue `Task` in `v1beta1` API.

## Using `git-clone` task instead of Git Resource

Given `Tekton` has deprecated resources, it is important to learn catalog tasks which can replace the resources, `git-clone`
is one such task.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
spec:
  workspaces:
    - name: output
      description: The git repo will be cloned onto the volume backing this workspace
  params:
    - name: url
      description: git url to clone
      type: string
    - name: revision
      description: git revision to checkout (branch, tag, sha, ref…)
      type: string
      default: master
    - name: refspec
      description: (optional) git refspec to fetch before checking out revision
      default: ""
    - name: submodules
      description: defines if the resource should initialize and fetch the submodules
      type: string
      default: "true"
    - name: depth
      description: performs a shallow clone where only the most recent commit(s) will be fetched
      type: string
      default: "1"
    - name: sslVerify
      description: defines if http.sslVerify should be set to true or false in the global git config
      type: string
      default: "true"
    - name: subdirectory
      description: subdirectory inside the "output" workspace to clone the git repo into
      type: string
      default: ""
    - name: deleteExisting
      description: clean out the contents of the repo's destination directory (if it already exists) before trying to clone the repo there
      type: string
      default: "false"
    - name: httpProxy
      description: git HTTP proxy server for non-SSL requests
      type: string
      default: ""
    - name: httpsProxy
      description: git HTTPS proxy server for SSL requests
      type: string
      default: ""
    - name: noProxy
      description: git no proxy - opt out of proxying HTTP/HTTPS requests
      type: string
      default: ""
  results:
    - name: commit
      description: The precise commit SHA that was fetched by this Task
  steps:
    - name: clone
      image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.12.1
      script: |
        CHECKOUT_DIR="$(workspaces.output.path)/$(params.subdirectory)"

        cleandir() {
          # Delete any existing contents of the repo directory if it exists.
          #
          # We don't just "rm -rf $CHECKOUT_DIR" because $CHECKOUT_DIR might be "/"
          # or the root of a mounted volume.
          if [[ -d "$CHECKOUT_DIR" ]] ; then
            # Delete non-hidden files and directories
            rm -rf "$CHECKOUT_DIR"/*
            # Delete files and directories starting with . but excluding ..
            rm -rf "$CHECKOUT_DIR"/.[!.]*
            # Delete files and directories starting with .. plus any other character
            rm -rf "$CHECKOUT_DIR"/..?*
          fi
        }

        if [[ "$(params.deleteExisting)" == "true" ]] ; then
          cleandir
        fi

        test -z "$(params.httpProxy)" || export HTTP_PROXY=$(params.httpProxy)
        test -z "$(params.httpsProxy)" || export HTTPS_PROXY=$(params.httpsProxy)
        test -z "$(params.noProxy)" || export NO_PROXY=$(params.noProxy)

        /ko-app/git-init \
          -url "$(params.url)" \
          -revision "$(params.revision)" \
          -refspec "$(params.refspec)" \
          -path "$CHECKOUT_DIR" \
          -sslVerify="$(params.sslVerify)" \
          -submodules="$(params.submodules)" \
          -depth "$(params.depth)"
        cd "$CHECKOUT_DIR"
        RESULT_SHA="$(git rev-parse HEAD | tr -d '\n')"
        EXIT_CODE="$?"
        if [ "$EXIT_CODE" != 0 ]
        then
          exit $EXIT_CODE
        fi
        # Make sure we don't add a trailing newline to the result!
        echo -n "$RESULT_SHA" > $(results.commit.path)
```

Create the `Task` above

`kubectl apply -f git-clone.yml -n tekton-pipelines`

You can use a task run as follows to see how the `git-clone` task works.

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: git-clone-run-
  namespace: tekton-pipelines
spec:
  taskRef:
  name: git-clone
  workspaces:
  - name: output # must match workspace name in the Task
    persistentVolumeClaim:
      claimName: mypvc # this PVC must already exist
    subPath: clone
  params:
  - name: url
    value: https://github.com/KnowledgeHut-AWS/katacoda-labs
  - name: revision
    value: master
```

> __Exercise__: complete the pre-requisites for this run to execute it successfully. How will you check if the repository is cloned?
