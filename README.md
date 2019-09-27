# Kubernetes Workshop

## Contents

- [Overview](#overview)
- [Project Setup](#project-setup)
- [Infrastructure Setup](#infrastructure-setup)
  - [Install Istio](#install-istio)
  - [Install Knative](#install-knative)
- [Deploying Examples](#deploying-examples)
- [Usage](#usage)
- [Cleanup](#cleanup)

## Overview

## Project Setup

- Install the [Google Cloud SDK](https://cloud.google.com/sdk)
- Create a [Google Cloud](https://console.cloud.google.com) project (with billing)
- Enable the following [APIs](https://console.cloud.google.com/apis/library):
  - Cloud Build
  - Container Registry
  - Cloud Run
  - Kubernetes Engine

```
gcloud services enable \ 
  container.googleapis.com \
  containerregistry.googleapis.com \
  cloudbuild.googleapis.com \
  run.googleapis.com
```

## Infrastructure Setup

- Create a GKE cluster

```
gcloud beta container clusters create workshop \
  --addons=HorizontalPodAutoscaling,HttpLoadBalancing \
  --machine-type=n1-standard-4 \
  --cluster-version=latest \
  --enable-stackdriver-kubernetes \
  --enable-ip-alias \
  --scopes cloud-platform
```

- Grab the cluster credentials so you can run `kubectl` commands

`gcloud container clusters get-credentials workshop`

- Create a `cluster-admin` role binding so you can deploy and configure Istio and Knative

```
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```

### Install Istio

- Download and unpack Istio

`curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.5 sh -`

- Install the `demo` profile

`kubectl apply -f istio-1.2.5/install/kubernetes/istio-demo.yaml`

- Install Stackdriver support

`kubectl apply -f https://github.com/GoogleCloudPlatform/istio-samples/raw/master/stackdriver-metrics/istio-stackdriver-metrics.yaml`

- Configure Istio sidecar auto-injection in the `default` namespace

`kubectl label ns default istio-injection=enabled`

### Install Knative

- Install the Knative container resource definitions (CRDs)

```
kubectl apply --selector knative.dev/crd-install=true \
   -f https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
   -f https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
   -f https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml
```

- Install the Knative `serving`, `eventing`, and `monitoring` components

```
kubectl apply \
  -f https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
  -f https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
  -f https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml
```

- Find the `istio-ingressgateway` external IP address

```
INGRESS=$(kubectl get svc -n istio-system istio-ingressgateway -o jsonpath={.status.loadBalancer.ingress..ip})
```

- Update the internal Knative domain configuration with the IP address from the previous step, using [xip.io](http://xip.io) as the dynamic DNS provider
- **Note**: this is for demo purposes only. For production deployments, refer to [Setting up a custom domain](https://knative.dev/docs/serving/using-a-custom-domain/) for Knative on GKE

```
kubectl patch configmap config-domain --namespace knative-serving --patch \
  '{"data": {"example.com": null, "[INGRESS].xip.io": ""}}'
```

## Deploying Examples

- Install [Hipstershop](https://github.com/GoogleCloudPlatform/microservices-demo)

```
kubectl apply -f hipstershop-istio.yaml
kubectl apply -f hipstershop-kubernetes.yaml
```

- Install the [Office Space](http://github.com/crcsmnky/cloud-run-office-space) sample app

`kubectl apply -f officespace-knative.yaml`

## Usage

Check out monitoring, logging, and tracing using Istio and Stackdriver [link](https://github.com/GoogleCloudPlatform/istio-samples/tree/master/istio-stackdriver/)

Generate transactions, check bank holdings, and check the account balance using the Office Space app [link](https://github.com/crcsmnky/cloud-run-office-space#usage)

## Cleanup

`gcloud container clusters delete workshop`
