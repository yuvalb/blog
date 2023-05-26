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

3. Connect your domain

In your DNS provider (usually where you bought the domain. I use namecheap.com), you need to define an A record leading to your ingress' IP.

4. Create the ingress for our chart and couple it with the staging issuer

Now this part is a bit tricky and depends on your chart. Ultimately you will want an ingress that looks like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/backend-protocol: HTTP
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  labels:
    app.kubernetes.io/instance: test
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: base-service
    app.kubernetes.io/version: 1.16.0
    helm.sh/chart: base-service-0.1.8
  name: test-base-service
  namespace: taskdag
spec:
  rules:
    - host: test.taskdag.com
      http:
        paths:
          - backend:
              service:
                name: test-base-service
                port:
                  number: 80
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - test.taskdag.com
      secretName: test-tls
```

If you followed this guide from the beginning and created your base-service chart using `helm create`, this can be achieved by editing the `ingress` part of your `values.yaml` file:

```yaml
ingress:
  enabled: true
  className: ""
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  hosts:
    - host: test.taskdag.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: test-tls
      hosts:
        - test.taskdag.com
```

Behind the scenes: cert-manager will issue a certificate and secret containing your app's TLS data which the ingress will use (\*maybe research this part better)

Once you visit your example host and see it has a letsencrypt staging certificate, you can change `letsencrypt-staging` to `letsencrypt-prod`, refresh and sync your application- and congratulations! You have your own app accessable via HTTPS with a TLS certificate.

5. (Optional) Expose your ArgoCD server with an Ingress

While port-fowarding directly from the cluster works, it's tedious to authenticate with the cluster and run the port-forward command every time you wish to interact with your ArgoCD server.

By exposing your ArgoCD server externally, you will get rid of all this friction for interacting with it, though you will still need to authenticate yourself.

Be warned though, by exposing it you are giving access to the server not only to yourself, but to anyone on the internet as well - with only your password guarding the access. If someone gets your password, or abuses some intrinsic security flaw in the ArgoCD server - they might be able to access all your apps and settings!

A better practice would be to expose it only through your organization's VPN and make it accessible only that way - thus adding obfuscating it from the outside world entirely.
Having said that, for most light usecases having it password protected can be enough.

So after I've warned you about the potential risk - follow the rest of this guide if you wish to proceed.

### Adding a DNS record

As before, you will need to add an A record to your DNS provider with your desired subdomain. My recommendation is that if your domain is `example.com`, add an `argocd` A record. The final URL of your ArgoCD server this way will be `argocd.example.com`.

It may take a few minutes for the changes to take effect. You can ping your host to verify it indeed leads to your cluster's IP:

```sh
ping argocd.example.com
```

### Creating an Ingress to connect the host to your argocd-server service

Since we've already installed cert-manager and nginx-controller ClusterIssuers, all that is left for us is to add an Ingress for our argocd-server service.

In `cluster/argocd/templates` add a new file `argocd-server-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # If you encounter a redirect loop or are getting a 307 response code
    # then you need to force the nginx ingress to connect to the backend using HTTPS.
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
    - host: argocd.staging.taskdag.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  name: https
  tls:
    - hosts:
        - argocd.staging.taskdag.com
      secretName: argocd-secret # do not change, this is provided by Argo CD
```

Once you've done this, you can upgrade the helm chart on the cluster:

```sh
# From root directory
helm upgrade argocd ./cluster/argocd --namespace argocd
```

Once the upgrade is successful, verify there is an Ingress created in the `argocd` namespace:

```sh
 kubectl -n argocd describe ingress argocd-server-ingress
```

Finally, once the Ingress is up you can access the UI via the browser at `argocd.example.com` and also connect to this address via the argocd CLI.

Congratulations!
