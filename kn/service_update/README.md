# Deploy new revision to a Knative Service

Lets deploy a new revision to a Knative Service, the update may contain
configuration update for the service for e.g. container image, environment
variables, etc.

## Pipeline:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: kn-service-update
spec:
  resources:
  - name: image
    type: image
  params:
  - name: ARGS
    type: array
    description: Arguments to pass to kn CLI
    default:
      - "help"
  tasks:
  - name: kn-service-update
    taskRef:
      name: kn
    resources:
      inputs:
      - name: image
        resource: image
    params:
    - name: kn-image
      value: "gcr.io/knative-nightly/knative.dev/client/cmd/kn"
    - name: ARGS
      value:
        - "$(params.ARGS)"
```

Save the above YAML in for e.g. `kn_service_update_pipeline.yaml` and
update if required or you can create above pipeline as is using

```
oc create -f https://raw.githubusercontent.com/navidshaikh/tekton-pipelines/master/kn/service_update/kn_service_update_pipeline.yaml
```

## PipelineResource

Let's create pipeline resource to input to our pipline. Create this
pipeline resource if you'd like to deploy a new image to your service

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: kn-service-update-image
spec:
  type: image
  params:
    - name: url
      value: "gcr.io/knative-samples/helloworld-go"
```

Save the above YAML in for e.g. `resources.yaml` and update values for
image URL if required and create the resource

```bash
oc apply -f resources.yaml
```

3. Create the pipeline run to trigger our pipeline and input the kn CLI
parameters to pass to deploy a new revision to our service

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: kn-service-update-
spec:
  serviceAccount: kn-deployer-account
  pipelineRef:
    name: kn-service-update
  resources:
    - name: image
      resourceRef:
        name: kn-service-update-image
  params:
    - name: ARGS
      value:
        - "service"
        - "update"
        - "hello"
        - "--revision-name=hello-v2"
        - "--image=$(inputs.resources.image.url)"
        - "--env=TARGET=v2"
```

Save the above YAML in for e.g. `pipeline_run.yaml` and create it

```bash
oc apply -f pipeline_run.yaml
```

Lets monitor the logs of our pipeline run using `tkn`
```bash
tkn pr list
tkn pr logs <pipelinerun-name> logs -f
```

After the successful run of the pipeline, we should have the Knative Service cr
```bash
oc get ksvc
```
