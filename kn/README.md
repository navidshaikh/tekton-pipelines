# Git source to Knative Service

This tutorial is about building you git source using `buildah` and deploying it as Knative service using `kn`,
performing Knative service updates, traffic splitting etc.

## Prerequisites:
1. OpenShift / Kubernetes cluster
2. `oc` CLI binary, grab latest from [here](https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/)
3. Tekton pipelines release v0.8.0
```bash
oc apply -f https://github.com/tektoncd/pipeline/releases/download/v0.8.0/release.yaml
```
4. tkn CLI, [install](https://github.com/tektoncd/cli#installing-tkn)
5. User account exists at a container registry for example: [quay.io](https://quay.io) or [docker.io](https://hub.docker.com/)

## Operations:

We'll create a tekton pipeline which performs following operations:

1. Create a container image from your git source
2. Push the container image to a configured container registry
3. Create a Knative Service from container created

We'll also configure a service account with
1. Privileged security context for buildah task to build container images
2. ClusterRole for CRUD operations on Knative resources
3. Linked container registry `secrets` to be able to push images to registry
4. Linked `imagePullSecrets` to pull images to from registry


## One time setup:
1. Lets create a sample project `tkn-kn`, we'll reference this project/namespace in our upcoming operations.
```bash
oc new-project tkn-kn
```

2. Create `docker-regitry` secrets for pushing built images to registry
```bash
oc create secret docker-registry container-registry --docker-server=<DOCKER-REGISTRY> --docker-username=<USERNAME> --docker-password=<PASSWORD> --docker-email=<EMAIL>
```

**Note:**
- Push to (a private repo at) `docker.io` via buildah and pull via
  `imagePullSecrets` does not work well. Please [create a new](https://hub.docker.com/repository/create) empty
  **public** repository where we will push the image.
- Push to (a private repo at) `quay.io` via buildah and pull via `imagePullSecrets` works well. [Get](https://quay.io/signin/) your account created at quay.


3. Create Service Account `kn-deployer-account` and
 - link `container-registry` secret created above in step 2
 - create cluster role `kn-deployer` to access to Knative resources
 - binds the service account with cluster role `kn-deployer` in namespace `tkn-kn`

Save following YAML in for e.g. `kn_deployer.yaml` and update if required.

```yaml
# Define a ServiceAccount named kn-deployer-account that has permission to
# manage Knative services.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kn-deployer-account
  namespace: tkn-kn
# Link your container registry secrets
secrets:
  - name: container-registry
# To be able to pull the (private) image from the container registry
imagePullSecrets:
  - name: container-registry
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
oc apply -f kn_deployer.yaml
```

4. To be able to build containers using buildah, we'll need to add privileged security context and `edit` role to our service account
```bash
oc adm policy add-scc-to-user privileged -z kn-deployer-account
oc adm policy add-role-to-user edit -z kn-deployer-account
```

5. Install buildah task from catalog
```bash
oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/buildah/buildah.yaml
```

6. Install the kn task from catalog
```bash
oc apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/kn/kn.yaml
```

## Piplines:
1. Create image from git source and create Knative Service using built container [pipeline](./build_deploy/README.md)
2. Deploy new revision to your Knative Service [pipeline](./service_update/README.md)
3. Perform traffic splitting operations on your Knative Service [pipeline](./traffic_splitting/README.md)
