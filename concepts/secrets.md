# Secrets

Tekton supports authentication via the Kubernetes first-class Secret types listed below.

| Git | Docker |
|---|---|
| kubernetes.io/basic-auth | kubernetes.io/basic-auth |
| kubernetes.io/ssh-auth | kubernetes.io/dockercfg |
|   | kubernetes.io/dockerconfigjson |
|  |  |

A Run gains access to these `Secrets` through its associated `ServiceAccount`. Tekton requires that each supported Secret includes a Tekton-specific annotation.

Tekton converts properly annotated Secrets of the supported types and stores them in a Step's container as follows:

- __Git__: Tekton produces a `~/.gitconfig` file or a `~/.ssh` directory
- __Docker__: Tekton produces a `~/.docker/config.json` file

Each Secret type supports multiple credentials covering multiple domains and establishes specific rules governing credential formatting and merging. Tekton follows those rules when merging credentials of each supported type. To consume these Secrets, Tekton performs credential initialization within every Pod it instantiates, before executing any Steps in the Run. During credential initialization, Tekton accesses each Secret associated with the Run and aggregates them into a /`tekton/creds` directory. Tekton then copies or symlinks files from this directory into the user's `$HOME` directory.

## Configuring authentication for Git

This section describes how to configure the following authentication schemes for use with Git:

- [Secrets](#secrets)
  - [Configuring authentication for Git](#configuring-authentication-for-git)
    - [Configuring `basic-auth` authentication for Git](#configuring-basic-auth-authentication-for-git)
    - [Configuring `ssh-auth` authentication for Git](#configuring-ssh-auth-authentication-for-git)
    - [Using a custom port for SSH authentication](#using-a-custom-port-for-ssh-authentication)
    - [Using SSH authentication in `git` type `Tasks`](#using-ssh-authentication-in-git-type-tasks)
  - [Configuring authentication for Docker](#configuring-authentication-for-docker)
    - [Configuring `basic-auth` authentication for Docker](#configuring-basic-auth-authentication-for-docker)
  - [Configuring `docker*` authentication for Docker](#configuring-docker-authentication-for-docker)

### Configuring `basic-auth` authentication for Git

This section descibes how to configure a `basic-auth` type `Secret` for use with Git. In the example below,
before executing any `Steps` in the `Run`, Tekton creates a `~/.gitconfig` file containing the credentials
specified in the `Secret`. When the `Steps` execute, Tekton uses those credentials to retrieve
`PipelineResources` specified in the `Run`.

1. In `secret.yaml`, define a `Secret` that specifies the username and password that you want Tekton
   to use to access the target Git repository:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: basic-user-pass
     annotations:
       tekton.dev/git-0: https://github.com # Described below
   type: kubernetes.io/basic-auth
   data:
     username: <cleartext username>
     password: <cleartext password>
   ```

   In the above example, the value for `tekton.dev/git-0` specifies the URL for which Tekton will use this `Secret`,
   as described in [Understanding credential selection](#understanding-credential-selection).

1. In `serviceaccount.yaml`, associate the `Secret` with the desired `ServiceAccount`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: basic-user-pass
   ```

1. In `run.yaml`, associate the `ServiceAccount` with your `Run` by doing one of the following:

   - Associate the `ServiceAccount` with your `TaskRun`:

    ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: TaskRun
     metadata:
       name: build-push-task-run-2
     spec:
       serviceAccountName: build-bot
       taskRef:
         name: build-push
     ```
   - Associate the `ServiceAccount` with your `PipelineRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: PipelineRun
     metadata:
       name: demo-pipeline
       namespace: default
     spec:
       serviceAccountName: build-bot
       pipelineRef:
         name: demo-pipeline
     ```

1. Execute the `Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

### Configuring `ssh-auth` authentication for Git

This section descibes how to configure an `ssh-auth` type `Secret` for use with Git. In the example below,
before executing any `Steps` in the `Run`, Tekton creates a `~/.ssh/config` file containing the SSH key
specified in the `Secret`. When the `Steps` execute, Tekton uses this key to retrieve `PipelineResources`
specified in the `Run`.

1. In `secret.yaml`, define a `Secret` that specifies your SSH private key:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: ssh-key
     annotations:
       tekton.dev/git-0: github.com # Described below
   type: kubernetes.io/ssh-auth
   data:
     ssh-privatekey: <private-key>
     # This is non-standard, but its use is encouraged to make this more secure.
     # If it is not provided then the git server's public key will be requested
     # when the repo is first fetched.
     known_hosts: <known-hosts>
   ```

   In the above example, the value for `tekton.dev/git-0` specifies the URL for which Tekton will use this `Secret`,
   as described in [Understanding credential selection](#understanding-credential-selection).

1. Generate the `ssh-privatekey` value. For example:

   `cat ~/.ssh/id_rsa`

1. Set the value of the `known_hosts` field to the generated `ssh-privatekey` value from the previous step.

1. In `serviceaccount.yaml`, associate the `Secret` with the desired `ServiceAccount`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: ssh-key
   ```

1. In `run.yaml`, associate the `ServiceAccount` with your `Run` by doing one of the following:

   - Associate the `ServiceAccount` with your `TaskRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: TaskRun
     metadata:
       name: build-push-task-run-2
     spec:
       serviceAccountName: build-bot
       taskRef:
         name: build-push
     ```

   - Associate the `ServiceAccount` with your `PipelineRun`:

   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: PipelineRun
   metadata:
     name: demo-pipeline
     namespace: default
   spec:
     serviceAccountName: build-bot
     pipelineRef:
       name: demo-pipeline
   ```

1. Execute the `Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

### Using a custom port for SSH authentication

You can specify a custom SSH port in your `Secret`. In the example below,
any `PipelineResource` referencing a repository at `example.com` will connect
to it on port 2222.

```
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key-custom-port
  annotations:
    tekton.dev/git-0: example.com:2222
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: <private-key>
  known_hosts: <known-hosts>
```

### Using SSH authentication in `git` type `Tasks`

You can use SSH authentication as described earlier in this document when invoking `git` commands
directly in the `Steps` of a `Task`. Since `ssh` ignores the `$HOME` variable and only uses the
user's home directory specified in `/etc/passwd`, each `Step` must symlink `/tekton/home/.ssh`
to the home directory of its associated user.

**Note:** This explicit symlinking is not necessary when using a `git` type `PipelineResource` or the
[`git-clone` `Task`](https://github.com/tektoncd/catalog/tree/v1beta1/git) from Tekton Catalog.

For example usage, see [`authenticating-git-commands`](../examples/v1beta1/taskruns/authenticating-git-commands.yaml).

## Configuring authentication for Docker

This section describes how to configure the following authentication schemes for use with Docker:

- [Configuring `basic-auth` authentication for Docker](#configuring-basic-auth-authentication-for-docker)
- [Configuring `docker*` authentication for Docker](#configuring-docker-authentication-for-docker)

### Configuring `basic-auth` authentication for Docker

This section describes how to configure the `basic-auth` (username/password pair) type `Secret` for use with Docker.

In the example below, before executing any `Steps` in the `Run`, Tekton creates a `~/.docker/config.json` file containing
the credentials specified in the `Secret`. When the `Steps` execute, Tekton uses those credentials when retrieving
`PipelineResources` specified in the `Run`.

1. In `secret.yaml`, define a `Secret` that specifies the username and password that you want Tekton
   to use to access the target Docker registry:

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: basic-user-pass
     annotations:
       tekton.dev/docker-0: https://gcr.io # Described below
   type: kubernetes.io/basic-auth
   data:
     username: <cleartext username>
     password: <cleartext password>
   ```

   In the above example, the value for `tekton.dev/git-0` specifies the URL for which Tekton will use this `Secret`,
   as described in [Understanding credential selection](#understanding-credential-selection).

1. In `serviceaccount.yaml`, associate the `Secret` with the desired `ServiceAccount`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: basic-user-pass
   ```

1. In `run.yaml`, associate the `ServiceAccount` with your `Run` by doing one of the following:

   - Associate the `ServiceAccount` with your `TaskRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: TaskRun
     metadata:
       name: build-push-task-run-2
     spec:
       serviceAccountName: build-bot
       taskRef:
         name: build-push
     ```

   - Associate the `ServiceAccount` with your `PipelineRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: PipelineRun
     metadata:
       name: demo-pipeline
       namespace: default
     spec:
       serviceAccountName: build-bot
       pipelineRef:
         name: demo-pipeline
     ```

1. Execute the `Run`:

   ```shell
   kubectl apply --filename secret.yaml serviceaccount.yaml run.yaml
   ```

## Configuring `docker*` authentication for Docker

This section describes how to configure authentication using the `dockercfg` and `dockerconfigjson` type
`Secrets` for use with Docker. In the example below, before executing any `Steps` in the `Run`, Tekton creates
a `~/.docker/config.json` file containing the credentials specified in the `Secret`. When the `Steps` execute,
Tekton uses those credentials to access the target Docker registry.
f
**Note:** If you specify both the Tekton `basic-auth` and the above Kubernetes `Secrets`, Tekton merges all
credentials from all specified `Secrets` but Tekton's `basic-auth` `Secret` overrides either of the
Kubernetes `Secrets`.

1. Define a `Secret` based on your Docker client configuration file.
   
   ```bash
   kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
   ```
   For more information, see [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
   in the Kubernetes documentation.

1. In `serviceaccount.yaml`, associate the `Secret` with the desired `ServiceAccount`:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: build-bot
   secrets:
     - name: regcred
   ```

1. In `run.yaml`, associate the `ServiceAccount` with your `Run` by doing one of the following:

   - Associate the `ServiceAccount` with your `TaskRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: TaskRun
     metadata:
       name: build-with-basic-auth
     spec:
       serviceAccountName: build-bot
       steps:
       ...
     ```

   - Associate the `ServiceAccount` with your `PipelineRun`:

     ```yaml
     apiVersion: tekton.dev/v1beta1
     kind: PipelineRun
     metadata:
       name: demo-pipeline
       namespace: default
     spec:
       serviceAccountName: build-bot
       pipelineRef:
         name: demo-pipeline
     ```

1. Execute the build:

   ```shell
   kubectl apply --filename secret.yaml --filename serviceaccount.yaml --filename taskrun.yaml
   ```
