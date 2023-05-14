+++
title = "My Kubernetes Cluster Part 3: Expose your service with cert-manager and an Ingress controller"
date = 2023-03-13
draft = true
slug = "deploy-cluster-argocd3"

[taxonomies]
tags = ["devops"]
+++

Requirements: Your own domain

Our test app is up and running, but we can only access it by port-forwarding directly from the cluster.

Obviously, you will want to showcase your cool projects to the public. For this we will need to use an Ingress.

You probably know that it is best practice today to communicate over HTTPS, but this requires a TLS certificate.

Before setting up our ingress, we will set up a cert-manager ClusterIssuer that will grant out Ingress the right certificates.

1. Set up cert-manager ClusterIssuer

We will create a new helm chart for `cert-manager`in `cluster/cert-manager`:

```sh
# From cluster dir:

# Create cert-manager dir for our chart
mkdir cert-manager

# Create the chart
cat >> cert-manager/Chart.yaml <<EOF

apiVersion: v2
name: cluster
description: A Helm chart for Kubernetes
type: application
version: 0.1.2
appVersion: "1.16.0"

dependencies:
  - name: cert-manager
    version: v1.11.0
    repository: https://charts.jetstack.io
EOF
```

It is best practice to have both a staging and production cluster issuer (\*\*explain difference).

We will create them both:

```sh
# Make a templates/tls dir to hold our cluster issuer resources
mkdir -p cert-manager/templates/tls

# Create a staging cluster issuer
cat >> cert-manager/templates/tls/staging-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: support@taskdag.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class: nginx
EOF

# Create a production cluster issuer
cat >> cert-manager/templates/tls/prod-issuer.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: support@taskdag.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class: nginx
EOF

```

Finally the chart structure should look like this:

```yaml
cluster
- cert-manager
- templates
- tls
- staging-issuer.yaml
- prod-issuer.yaml
Chart.yaml
```

Now that we have the chart set up, it's time to install it under the `cert-manager` namespace.

```sh
helm dependency build ./cert-manager
helm install cert-manager ./cert-manager --create-namespace --namespace cert-manager
```

Verify the cluster issuers were installed:

```sh
kubectl -n cert-manager get clusterissuer
```

You should see:

```sh
NAME                  READY   AGE
letsencrypt-prod      True    10s
letsencrypt-staging   True    10s
```

2. Install an ingress controller

We will use ingress-nginx ingress controller that will be installed in its own namepsace `ingress-nginx`.

We will create its chart under `cluster` directory:

```sh
# In cluster directory

mkdir ingress-nginx

cat >> ./ingress-nginx/Chart.yaml <<EOF
apiVersion: v2
name: cluster
description: A Helm chart for Kubernetes
type: application
version: 0.1.2
appVersion: "1.16.0"

dependencies:
  - name: ingress-nginx
    version: 4.6
    repository: https://kubernetes.github.io/ingress-nginx
EOF
```

Note that I'm using version 4.6 which is the latest version that fits my k8s cluter's version when writing this. You can find the best version for you [here](https://github.com/kubernetes/ingress-nginx#supported-versions-table).

Now we will install it:

```sh
# In cluster directory
helm dependency build ./ingress-nginx
helm install ingress-nginx ./ingress-nginx --create-namespace --namespace ingress-nginx
```

Verify it was installed correctly. Get the load balancer and wait until it gets an external IP. That might take a few minutes, but ultimately you will see:

```sh
k -n ingress-nginx get service ingress-nginx-controller
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.245.249.136   X.X.X.X   80:32018/TCP,443:32738/TCP   2m37s
```

> Note: I obfuscated my external IP, but you should see a real one and not `X.X.X.X`

If you visit this IP via your browser, you will get an "nginx 404 Not Found" error - which is great! That means we reach our nginx controller from outside the cluster.

3. Create the ingress for our chart and couple it with the staging issuer

Now this part is a bit tricky and depends on your chart. Ultimately you will want an ingress that looks like this:

4. Connect your domain

In your DNS provider (usually where you bought the domain. I use namecheap.com), you need to define an A record leading to your ingress' IP.
