# Installation

To install the core component of Tekton, run the command below:

`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`

__Note__: If your container runtime does not support image-reference:tag@digest (for example, like cri-o used in OpenShift 4.x), use release.notags.yaml instead:

`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.notags.yaml`

It may take a few moments before the installation completes. You can check the progress with the following command:

`kubectl get pods --namespace tekton-pipelines`
