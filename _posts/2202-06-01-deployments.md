---
title: What are kubernetes deployments
author:
  name: Robert Kozak
date: 2022-06-01
categories: 
 - kubernetes
 - 'deployment resource'
tags: ['kubernetes', 'k8s']
---

# What are kubernetes deployments?

![Containers](/assets/images/containers.jpg)

## Deployment resource

---

Kubernetes resources describes the **desired** state of pods in the kubernetes cluster

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: ops-system
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
```

