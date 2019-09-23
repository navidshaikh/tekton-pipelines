# Git source to Knative Service

This tutorial is about building you git source using `buildah` and deploying it as Knative service using `kn`.

## Prerequisites:
1. OpenShift / Kubernetes cluster
2. `oc` CLI binary, grab latest from [here](https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/)
2. Tekton pipelines release v0.6.0
```bash
oc apply -f https://github.com/tektoncd/pipeline/releases/download/v0.6.0/release.yaml
```
3. tkn CLI, [install](https://github.com/tektoncd/cli#installing-tkn)


## Operations:

We'll create a tekton pipeline which performs following operations.

1. Create a container image from your git source
2. Push the container image to a configured docker registry
3. Create a Knative Service from container created

We'll also configure a service account with
1. Privileged security context for buildah task to build container images
2. ClusterRole for CRUD operations on Knative resources
3. Linked docker registry secrets to be able to push images to registry


## One time setup:
1. Lets create a sample project `tkn-kn`, we'll reference this project/namespace in our upcoming operations.
```bash
oc new-project tkn-kn
```

2. Create docker-regitry secrets for pushing built images to your registry
```bash
oc create secret docker-registry dockerreg --docker-server=docker.io --docker-username=<USERNAME> --docker-password=<PASSWORD> --docker-email=<EMAIL>
```

3. Create Service Account kn-deployer-account and
 - link `dockerreg` secrets created above in step 2
 - create cluster role `kn-deployer` to access to Knative resources
 - binds the service account with cluster role `kn-deployer` in namespace `tkn-kn`

Save following YAML in for e.g. `kn-deployer.yaml` and update names if required.

```yaml
# Define a ServiceAccount named kn-deployer-account that has permission to
# manage Knative services.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kn-deployer-account
  namespace: tkn-kn
# Link your docker registry secrets
secrets:
  - name: dockerreg

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kn-deployer
rules:
  - apiGroups: ["serving.knative.dev"]
    resources: ["services"]
    verbs: ["get", "list", "create", "update", "delete", "patch", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kn-deployer-binding
subjects:
- kind: ServiceAccount
  name: kn-deployer-account
  namespace: tkn-kn
roleRef:
  kind: ClusterRole
  name: kn-deployer
  apiGroup: rbac.authorization.k8s.io
```

Apply the config we created

```bash
oc apply -f kn-deployer.yaml
```

4. To be able to build containers using buildah, we'll need to add privileged security context and `edit` role to our service account
```bash
oc adm policy add-scc-to-user privileged -z kn-deployer-account
oc adm policy add-role-to-user edit -z kn-deployer-account
```

## Pipline:

Lets create the actual pipeline now, we'll proceed as
- Install buildah task
- Install kn-create task
- Create a pipeline for build and kn deploy
- Create pipeline resources to input to our pipeline
- Create a pipeline run to trigger the pipeline

1. Install buildah task
```bash
oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/buildah/buildah.yaml
```

2. Install the kn-create task
```bash
oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/kn/kn-create.yaml
```
3. Create a pipeline for build and deploy
We'll use above installed tasks in our pipeline as steps for our pipeline

Save following YAML in for e.g. `build-kn-deploy.yaml` and update names if required.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: build-kn-deploy
spec:
  resources:
  - name: source
    type: git
  - name: image
    type: image
  params:
  - name: service
    description: Knative service name
  - name: force
    description: Whether force creation of service to overwrite existing one with same name
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
  - name: kn-deploy
    taskRef:
      name: kn-create
    runAfter:
      - buildah-build
    resources:
      inputs:
      - name: image
        resource: image
        from:
          - buildah-build
    params:
    - name: service
      value: "$(params.service)"
    - name: force
      value: "$(params.force)"
```

Create the pipeline
```bash
oc apply -f build-kn-deploy.yaml
```

4. Create pipeline resource to input to our pipeline
As we see above, there are a couple of resources referenced in pipeline, we'll need to create these resources
to make available during pipeline run

Save following YAML in for e.g. `resources.yaml` and update values for your Git repo and container repository to
push built image to.

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: build-kn-deploy-source
spec:
  type: git
  params:
    - name: url
      value: "https://github.com/navidshaikh/go-helloworld"
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: build-kn-deploy-image
spec:
  type: image
  params:
    - name: url
      value: "docker.io/swordphilic/go-helloworld:buildah"
```

Create the resources
```bash
oc apply -f resources.yaml
```

5. Create the pipeline run to trigger our pipeline, pipeline run mentions the parameters required
to run our pipeline

Save following YAML in for e.g. `pipeline-run.yaml` and update parameters value if required.
```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: build-kn-deploy-
spec:
  serviceAccount: kn-deployer-account
  pipelineRef:
    name: build-kn-deploy
  params:
    - name: service
      value: "svc"
    - name: force
      value: "true"
  resources:
    - name: source
      resourceRef:
        name: build-kn-deploy-source
    - name: image
      resourceRef:
        name: build-kn-deploy-image
```

Create the pipelien run
```bash
oc apply -f pipeline-run.yaml
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
