+++
title = "My Kubernetes Cluster Part 1: ArgoCD"
date = 2023-03-11
draft = true
slug = "deploy-cluster-argocd"

[taxonomies]
tags = ["devops"]
+++

aim: deploy argocd cluster with user accounts

1. create git repo
2. install helm (explain why we use helm)
3. create `cluster` directory to manage our charts
4. create argocd chart

```sh
cd cluster
helm create argocd
```

clean templates directory

```sh
cd argocd
rm -rf templates/* # contains useful starter deployment, but we just wrap a chart here
```

clear the values.yaml file

add dependency on argocd

```sh
# cluster/argocd/Chart.yaml

dependencies:
  - name: argo-cd
    version: 5.24.1
    repository: https://argoproj.github.io/argo-helm
```

5. create argocd namespace

```sh
kubectl create namespace argocd
kubectl get namespace # verify it's created
```

6. Installing the chart

```sh
helm dependency build ./argocd
helm install argocd ./argocd --namespace argocd

kubectl -n argocd get all # verify it's there
```

7. Accessing the API

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

you can also direct your browser to https://localhost:8080 for the ui

9. Create users
   Create new file `/cluster/argocd/tempaltes/argocd-accounts.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
data:
  # add an additional local user with apiKey and login capabilities
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.<name>: apiKey, login
```

replace `<name>` with your account name

upgrade the chart

```sh
helm upgrade argocd ./argocd --namespace argocd
```

Install [argocd cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

get a temporary admin password

```sh
argocd admin initial-password -n argocd
```

login to your cluster exposed on localhost:8080, using the username `admin` and the password from previous step

```sh
argocd login localhost:8080
```

verify accounts are present

```sh
argocd account list
```

update admin password

```sh
argocd account update-password
```

update account password

```sh
argocd account update-password \
  --account <name> \
  --current-password <current-user-password> \ # By default it is current admin password
  --new-password "<new-user-password>"
```

> Changes might not take effect until we restart the `argocd-server`

restart server

```sh
kubectl rollout restart deployment argocd-server -n argocd
```

port-forward again and go to localhost:8080. login to your account to verify that it worked.

```sh

```

10. disable admin user

Edit the `argocd-accounts.yaml` template:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
data:
  # add an additional local user with apiKey and login capabilities
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.<name>: apiKey, login

  # Disable admin
  admin.enabled: "false"
```

upgrade the helm chart, delete admin secret and restart:

```sh
helm upgrade argocd ./argocd --namespace argocd
kubectl delete secret argocd-initial-admin-secret -n argocd
kubectl rollout restart deployment argocd-server -n argocd
```

Congratulations!
