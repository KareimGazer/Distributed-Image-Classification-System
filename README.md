# Distributed Image Classication System

## Motivation

Scaling up machine learning models from personal devices to large distributed clusters is one of the biggest challenges faced by practitioners. Distributing machine learning systems allows us to handle extremely large datasets across multiple clusters, and benefit from hardware accelerations.

In this project we introduce how you can build your own distributed image classication system that handels all aspects from data ingestion to model serving in production.

## Assumptions

we use K3d as a wrapper around K3s "the lightweight Kubernetes distribution" to be able to run the cluster on a single local machine.

we assume that the production cluster is heterogeneous meaning that it contains machines that have GPUs while other machines don't.

## Infrastructure Setup

### Cluster

Or via `k3d`:

```bash
k3d cluster create distml --image rancher/k3s:v1.25.3-k3s1
```


```bash
# create a dedicated namespace to separate your resources
kubectl create ns kubeflow

# switch to that namespace
kns kubeflow

# run the yaml manifests to setup the K8s cluster and all related CRDs properly
kubectl kustomize manifests | kubectl apply -f -
```

### Clean-up

run this clean up to remove the cluster and free all used resources after you delete/clean up all Kubernetes resources.

```bash
k3d cluster rm distml
kind delete cluster --name distml
```

## Run Workflow
