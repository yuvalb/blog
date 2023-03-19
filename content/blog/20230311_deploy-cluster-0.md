+++
title = "My Kubernetes Cluster Part 0: Overview"
date = 2023-03-11
draft = true
slug = "my-k8s-cluster-0"

[taxonomies]
tags = ["devops"]
+++

This is the first in a series of blog posts in which I will document and walk you through how I deploy my flavor of Kubernetes cluster.

My cluster's goal will be to enable deploying my hobby applications and passion projects. In a sense, this cluster is also a passion project 😄.

## 1. The Requirements

Before starting most projects, I find it best to first describe my goals in the form of requirements to meet - so I can later measure my success.

1. **Scale**: Applications should be able to scale within reason.
2. **Agility**: New applications and projects should be deployed easily.
3. **Security**: The cluster and its secrets should be secure.
4. **Fun**: Must be maximized at all costs.

## 2. The Plan

1. The cluster will be based on Kubernetes.
2. The templating and package management system will be Helm and our own chart repository.
3. Applications and their deployment will be managed via ArgoCD.
4. Secret injections will be done with Hashicorp's Vault.

## 3. The Prerequisites

Before you begin, you must have access to some crucial resources.

1. A working Kubernetes cluster: I will be using [DigitalOcean's offering](https://www.digitalocean.com/products/kubernetes), but you can use an alternative or [Minikube](https://minikube.sigs.k8s.io/docs/start/) if you wish to play with a local cluster.
2. A git repository: I am using [GitHub](https://github.com).
3. The recommended Kubernetes CLIs: `kubectl`, `kubens` and `kubectx` if you dabble in more than one cluster.

These are the basic requirements to start this walkthrough. Each of the sections to follow will detail its own prerequisites.
