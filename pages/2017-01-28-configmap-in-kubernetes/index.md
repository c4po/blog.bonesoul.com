---
title: Mount config file or license file in Kubernetes
date: "2017-01-28T15:13:54Z"
layout: post
path: "/configmap-kubernetes/"
tags:
  - docker
  - Kubernetes
  - configmap
---

# How to mount config file to container in Kubernetes

## Using [configmap][a2e8ebdf]

Many applications require configuration via some combination of config files, command line arguments, and environment variables. Kubernetes provides the [configmap][a2e8ebdf] api to store configuration data which can be consumed by pods.

## Limitation

`Configmap` or `secrets` can be consumed by pods in 2 ways. Using as files and using as values. In most cases, we'd like to let `configmap` be consumed as a whole file instead of a set of values by the application. To consume `configmap` or `secrets` as files in container, we need mount it as a volume to the container. for example,
```
spec:
  containers:
    - name: test-container
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: dev-config
```
In this case, if the mount path `/etc/config` already exists inside the container image, the `configmap` mount overlays the pre-existing contents. The folders and files under the path will be not accessible by application.

## Solution

In most case, the application config file is sit with other files in the container. We can not simply create a mount path to mount the `configmap` to the container and overlay other files. When we build the image, we create a symbolic link for the config file or license file inside the container.

* Dockerfile
```
FROM ...
...
RUN ln -s /kubernetes/secrets/license.properties /${APP_HOME}/license.properties
RUN ln -s /kubernetes/config/database.properties /${APP_HOME}/database.properties
...
```

* Create configmap from yaml or command line
```
$ kubectl create configmap dev-config --from-file=database.properties
...
```

* Mount the configmap to pods
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: myapp
      volumeMounts:
      - name: config-volume
        mountPath: /kubernetes/config
      - name: secrets-volume
        mountPath: /kubernetes/secrets
  volumes:
    - name: config-volume
      configMap:
        name: dev-config
    - name: secrets-volume
      secret:
        name: dev-secrets
```
[a2e8ebdf]: https://kubernetes.io/docs/user-guide/configmap/ "configmap"
