+++
title = "Deploying JupyterHub locally on K8S for development"
date = "2025-01-15"
draft = true
[taxonomies]
tags=["jupyterhub", "k8s"]
[extra]
comment = true
+++

You need to install [kind](https://kind.sigs.k8s.io/),
[kubectl](https://kubernetes.io/docs/tasks/tools/) and
[helm](https://helm.sh/docs/intro/install/).


Wildberries research cluster [repository](https://gitlab.wildberries.ru/recommendation/mlops/research)


JupyterHub GitHub [repository](https://github.com/jupyterhub/jupyterhub)
KubeSpawner [repository](https://github.com/jupyterhub/kubespawner)

Zero to [jupyterhub k8s](https://github.com/jupyterhub/zero-to-jupyterhub-k8s) 

```
kind create cluster --name=jupyterhub-dev
kubectl cluster-info --context jupyterhub-dev
kubectl create ns jupyter
```

```
helm create jupyterhub-dev
```

* `Chart.yaml`: Contains metadata about the chart, such as name, version, and description.
* `values.yaml`: Default configuration values for your chart. These can be overridden when installing the chart.
* `templates/`: Contains Kubernetes manifest templates (YAML) with Helm template directives (logic).
* `charts/`: Holds sub-charts that your chart depends on.

```
docker buildx build -f docker/Dockerfile.hub \
               -t jupyterhub-hub-dev:3.1.0 ./jupyterhub && \
kind load docker-image jupyterhub-hub-dev:3.1.0 --name jupyterhub-dev

docker buildx build -f docker/Dockerfile.user \
               -t jupyterhub-user-dev:3.1.0 --build-arg BASE_IMAGE="python:3.9.19" ./jupyterhub && \
kind load docker-image jupyterhub-user-dev:3.1.0 --name jupyterhub-dev

docker buildx build -f docker/Dockerfile.rss \
               -t jupyterhub-rss-dev:3.1.0 ./jupyterhub && \
kind load docker-image jupyterhub-rss-dev:3.1.0 --name jupyterhub-dev
```




Local K8S Setup for JupyterHub Development

Introduction

Overview

Set of minimal components

Local cluster setup

Writing helm-chart

Deploying JupyterHub