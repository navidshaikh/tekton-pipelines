# Buildah build and kn service create Pipeline

Lets create a container image from your git source having a Dockerfile and
deploy the built container to a Knative Service. In this guide, we'll create

- A pipeline for buildah build and kn service create
- Pipeline resources for git source and resulting container image repository
- A pipeline run to tie above together and trigger the pipeline

## Pipline:

We'll create a tekton pipeline which performs following operations:

1. Create a container image from your git source having a Dockerfile
2. Push the container image to a configured docker registry
3. Create a Knative Service from built container

Lets create a pipeline for git source build and service creation

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: buildah-build-kn-create
spec:
  resources:
  - name: source
    type: git
  - name: image
    type: image
  params:
  - name: ARGS
    type: array
    description: Arguments to pass to kn CLI
    default:
      - "help"
  tasks:
  - name: buildah-build
    taskRef:
      name: buildah
    resources:
      inputs:
        - name: source
          resource: source
      outputs:
        - name: image
          resource: image
  - name: kn-service-create
    taskRef:
      name: kn
    runAfter:
      - buildah-build
    resources:
      inputs:
      - name: image
        resource: image
        from:
          - buildah-build
    params:
    - name: kn-image
      value: "gcr.io/knative-nightly/knative.dev/client/cmd/kn"
    - name: ARGS
      value:
        - "$(params.ARGS)"
```

Save the above YAML in for e.g. `build_deploy_pipeline.yaml` and update
names if required or you can use above pipeline as is using
```
oc create -f https://raw.githubusercontent.com/navidshaikh/tekton-pipelines/master/kn/build_deploy/build_deploy_pipeline.yaml
```

## PipelineResource

Let's create pipeline resource to input to our pipeline.
As we see above, there are a couple of resources referenced in pipeline,
we'll need to create these resources to make available during pipeline run.
Make sure your git repository has a Dockerfile at root of the repo.
Make sure the docker repository URL is correct and you have push access to.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: buildah-build-kn-create-source
spec:
  type: git
  params:
    - name: url
      value: "https://github.com/navidshaikh/go-helloworld"
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: buildah-build-kn-create-image
spec:
  type: image
  params:
    - name: url
      value: "docker.io/swordphilic/go-helloworld:tkn"
```

Save the above YAML in for e.g. `resources.yaml` and update values for your git repo and container repository to
push built image to and create the resources

```bash
oc apply -f resources.yaml
```

3. Create the pipeline run to trigger our pipeline, pipeline run references the pipline resouces created above and
mentions the parameters required to run our pipeline and perform the task

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: buildah-build-kn-create-
spec:
  serviceAccount: kn-deployer-account
  pipelineRef:
    name: buildah-build-kn-create
  resources:
    - name: source
      resourceRef:
        name: buildah-build-kn-create-source
    - name: image
      resourceRef:
        name: buildah-build-kn-create-image
  params:
    - name: ARGS
      value:
        - "service"
        - "create"
        - "hello"
        - "--force"
        - "--image=$(inputs.resources.image.url)"
        - "--env=TARGET=Tekton"
```

Save the above YAML in for e.g. `pipeline_run.yaml` and update parameters value if required and
create the pipelien run

```bash
oc apply -f pipeline_run.yaml
```

Lets monitor the logs of our pipeline run using `tkn`
```bash
tkn pr list
tkn pr logs <pipelinerun-name> logs -f
```

After the successful run of the pipeline, we should have the Knative Service created
```bash
oc get ksvc
```
