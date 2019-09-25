# Perform traffic splitting on the Knative Service

Lets perform traffic splitting on the Knative Service using
pipeline. Make sure you have a couple of revisions available for your
Knative Service.

## Pipeline:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: kn-service-traffic-splitting
spec:
  params:
  - name: ARGS
    type: array
    description: Arguments to pass to kn CLI
    default:
      - "help"
  tasks:
  - name: kn-service-traffic-splitting
    taskRef:
      name: kn
    params:
    - name: kn-image
      value: "gcr.io/knative-nightly/knative.dev/client/cmd/kn"
    - name: ARGS
      value:
        - "$(params.ARGS)"
```

Save the above YAML in for e.g. `kn_service_traffic_splitting_pipeline.yaml` and
update if required or you can create above pipeline as is using

```
oc create -f https://raw.githubusercontent.com/navidshaikh/tekton-pipelines/master/kn/traffic_splitting/kn_service_traffic_splitting_pipeline.yaml
```

2. Create the pipeline run to trigger our pipeline and input the kn CLI
parameters to pass to perform the traffic splitting.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: kn-service-traffic-splitting-
spec:
  serviceAccount: kn-deployer-account
  pipelineRef:
    name: kn-service-traffic-splitting
  params:
    - name: ARGS
      value:
        - "service"
        - "update"
        - "hello"
        - "--tag=hello-v1=v1"
        - "--tag=hellov-2=v2"
        - "--traffic=v1=90"
        - "--traffic=v2=10"
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
