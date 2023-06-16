+++
title = "My Kubernetes Cluster Part 2: Setup Helm chart repository and deploy an app from a chart"
date = 2023-03-12
draft = true
slug = "deploy-cluster-argocd2"

[taxonomies]
tags = ["devops"]
+++

We will connect ArgoCD to our github repo, where we will define apps and charts.

Before we define an app, we need to setup a project for that app to live in, and a repository from which we will draw the app's charts that argocd will use to sync on.

1. Create github token

Go to [settings > developer settings > personal access tokens](https://github.com/settings/tokens) and click `Generate new token`.

Select the repo scope and generate the token. Save it for now.

1. Create repository

Create repository yaml file at /cluster/argocd/templates/repositories.yaml:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <REPO_NAME>
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: <REPO_URL>
  password: <GITHUB_TOKEN>
  username: not-used
```

> Note: add it to gitignore since it contains your secret token.

Now run helm upgrade:

```sh
helm upgrade argocd ./argocd --namespace argocd
```

And you should see your repository appear in the repository settings.

2. Create project

Next we create a project which is a group for our apps to live in.

Create `cluster/argocd/templates/projects.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: example
  namespace: argocd
  # Finalizer that ensures that project is not deleted until it is not referenced by any application
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Example Project
  sourceRepos:
    - "https://github.com/yuvalb/taskdag-k8s"
  destinations:
    - namespace: taskdag
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "*"
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

> Note: We will create this namespace later.

Now run helm upgrade:

```sh
helm upgrade argocd ./argocd --namespace argocd
```

1. Create an app of apps

Create `apps/apps` chart

```
apps
- templates
  Chart.yaml
```

```yaml
# apps/apps/Chart.yaml
apiVersion: v2
name: apps
description: A Helm chart containing ArgoCD apps
type: application
version: 0.1.0
appVersion: "1.0.0"
```

Add this application to argocd:

```yaml
# cluster/argocd/templates/applications.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: <your-repo>
    targetRevision: HEAD
    path: apps/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

Upgrade helm

```sh
helm upgrade argocd ./argocd --namespace argocd
```

You might need to restart your application controller:

```sh
kubectl rollout restart statefulset.apps argocd-application-controller  -n argocd
```

Refresh UI and you should see the app healthy.

1. Create a base-service chart

Create charts directory

```sh
# From repo root
mkdir charts
cd ./charts
```

Createa a new chart

```sh
helm create base-service
```

> Note: I am using helm v3.11.1 which creates a new chart with the following resources: deployment, hpa, ingress, service, serviceaccount. If this is not the case for your version from the future, you should apply my recommendations to your version of the created chart - or create a chart equivalent to mine.

> Optional: It's better practice to separate the cluster's nodes for our apps from the nodes used for our cluster, so they will not interfere with each other. For this, my cluster on DigitalOcean has a node pool called `pool-apps`.
> Assigning apps to nodes in this pool is done by updating the base service's values file to have a default `nodeSelctor`:

```yaml
# charts/base-service/values.yaml
nodeSelector:
  doks.digitalocean.com/node-pool: pool-apps
```

Note that the implementation and label selector might change between cloud/cluster providers.

2. Create an app from this chart

```sh
# From repo root
mkdir apps
cd ./apps
helm create test
rm -rf ./test/templates
rm ./test/values.yaml
```

Add the base-serice as a dependency to Chart.yaml

```yaml
# Add this to apps/test/Chart.yaml
dependencies:
  - name: base-service
    version: 0.1.0
    repository: file://../../charts/base-service
```

Finally, since we want our app to define the base-service chart's values, we will copy the `values.yaml` file in `charts/base-service/values.yaml` and make it our app's values file:

```sh
# from cluster directory

# Copy the original values file over to our app, indented under "base-service"
{ echo "base-service:"; sed 's/^/\t/' ./charts/base-service/values.yaml; } > ./apps/test/values.yaml
```

Notice that we put the contents of the original values file under the alias used for this chart in our example app:

```yaml
base-service:
  # Contents of the original file indented by 1 tab
```

This pattern allows us to mix and match multiple charts and their values in our app.

3. Add an app to app of apps

```yaml
# apps/apps/templates/example-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example
  namespace: argocd
spec:
  project: example
  source:
    repoURL: https://github.com/yuvalb/taskdag-k8s
    targetRevision: HEAD
    path: apps/example
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: example
```

create your target namespace

```sh
kubectl create namespace example
```

Push to repo and refresh app in argocd.

You should now see your example app appear unsynced.

You can sync it and your app shuold be healthy

Congratulations!
