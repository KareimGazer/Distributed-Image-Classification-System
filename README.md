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

## Setup

```
cd ./code
```

Build the image
```bash
# build and tag the docker image
docker build -f Dockerfile -t kubeflow/multi-worker-strategy:v0.1 .

# import the image into K3d
k3d image import kubeflow/multi-worker-strategy:v0.1 --cluster distml
```

Switch to "kubeflow" namespace:
```bash
kubectl config set-context --current --namespace=kubeflow
```

Specify the `storageClassName` and create a persistent volume claim to save the different trained models and checkpoints
```bash
kubectl create -f multi-worker-pvc.yaml
```

## Submitting Training Job

Create a TFJob:
```bash
kubectl create -f multi-worker-tfjob.yaml
```

In case of making any code changes, run the following to resubmit the job:
```bash
kubectl delete tfjob --all; docker build -f Dockerfile -t kubeflow/multi-worker-strategy:v0.1 .; kind load docker-image kubeflow/multi-worker-strategy:v0.1 --name distml; kubectl create -f multi-worker-tfjob.yaml
```

## Model selection

Now select the best model out the three trained ones
```bash
python3 /model-selection.py
```

## Model loading & prediction

If you want to test the best model on the fly
```bash
kubectl create -f predict-service.yaml
kubectl exec --stdin --tty predict-service -- bin/bash
python3 /predict-service.py
```

## Model serving

```bash
# Install KServe
curl -s "https://raw.githubusercontent.com/kserve/kserve/v0.10.0-rc1/hack/quick_install.sh" | bash

# Create inference service
kubectl create -f inference-service.yaml

# https://kserve.github.io/website/master/get_started/first_isvc/#4-determine-the-ingress-ip-and-ports
INGRESS_GATEWAY_SERVICE=$(kubectl get svc --namespace istio-system --selector="app=istio-ingressgateway" --output jsonpath='{.items[0].metadata.name}')
kubectl port-forward --namespace istio-system svc/${INGRESS_GATEWAY_SERVICE} 8080:80
# start another terminal
export INGRESS_HOST=localhost
export INGRESS_PORT=8080

MODEL_NAME=flower-sample                                                                                                      
INPUT_PATH=@./inference-input.json
SERVICE_HOSTNAME=$(kubectl get inferenceservice ${MODEL_NAME} -o jsonpath='{.status.url}' | cut -d "/" -f 3)
curl -v -H "Host: ${SERVICE_HOSTNAME}" "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict" -d $INPUT_PATH

## TODO: gRPC serving. Not working yet

# Client-side requirements
python3 -m pip install tensorflow-metal
python3 -m pip install tensorflow-macos==2.11.0
python3 -m pip install tensorflow-serving-api==2.11.0
```

Autoscaled inference service:
```bash
# https://github.com/rakyll/hey
brew install hey
kubectl create -f autoscaled-inference-service.yaml

hey -z 30s -c 5 -m POST -host ${SERVICE_HOSTNAME} -D inference-input.json "http://${INGRESS_HOST}:${INGRESS_PORT}/v1/models/$MODEL_NAME:predict"
```

## Complete Workflow

using Argo Workflows
```bash
kubectl create -f workflow.yaml
```

## Debugging

Access the trained model
```bash
kubectl create -f access-model.yaml 
kubectl exec --stdin --tty access-model -- ls /trained_model
# Manually copy
# kubectl cp trained_model access-model:/pv/trained_model -c model-storage
```

Run `TFServing` commands in the KServe container:
```bash
kubectl exec --stdin --tty flower-sample-predictor-default-00001-deployment-84759dfc5f6wfj -c kserve-container -- /usr/bin/tensorflow_model_server --model_name=flower-sample \
      --port=9000 \
      --rest_api_port=8080 \
      --model_base_path=/mnt \
      --rest_api_timeout_in_ms=60000
```

## Cleanup

```bash
kubectl delete tfjob --all
kubectl delete wf --all
kubectl delete inferenceservice flower-sample
kubectl delete pods --selector=app=flower-sample-predictor-default-00001 --force --grace-period=0
kubectl delete pod access-model --force --grace-period=0
kubectl delete pod predict-service --force --grace-period=0
kubectl delete pvc strategy-volume
```