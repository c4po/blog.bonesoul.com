---
title: SSHD Jump Server in Kubernetes
date: "2017-02-04T17:41:54Z"
layout: post
path: "/jumpserver-kubernetes/"
tags:
  - sshd
  - Kubernetes
---

# SSHD Jump Server in Kubernetes

## Motivation:

1. Trouble shooting pods/services deployed in Kubernetes with internal DNS
2. Temporary sandbox linux server

## TL;DR

[GitRepo][a2e8ebdf]

## Docker image

* The docker image can be created from `Centos:7` and just install the openssh module.

```
FROM centos:7

RUN yum -y install openssh openssh-clients openssh-server
EXPOSE 22

COPY entrypoint.sh /
CMD ["/entrypoint.sh"]
```

* We don't want hard code the public key in the image. So we use `entrypoint.sh` to install the public key which get from environment variable.

```
#!/bin/bash

ssh-keygen -f /etc/ssh/ssh_host_rsa_key -N '' -t rsa
ssh-keygen -f /etc/ssh/ssh_host_ecdsa_key -N '' -t ecdsa
ssh-keygen -f /etc/ssh/ssh_host_ed25519_key -N '' -t ed25519

mkdir -p /root/.ssh
touch /root/.ssh/authorized_keys
echo ${PUBLIC_KEY} > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

/usr/sbin/sshd -D
```

## Deploy to kubernetes

* Generate the ssh key or using existing ssh key

```
ssh-keygen -q -f sshkeys/id_rsa -N '' -t rsa
```

* Encode the ssh key with bas64 and create secrets file for Kubernetes

```
KEY=$$(cat sshkeys/id_rsa.pub |base64) ;\
sed "s/PUBLIC_KEY/$${KEY}/" secret.yaml.tmpl	> secret.yaml
```

secret.yaml.tmpl

```
apiVersion: v1
kind: Secret
metadata:
  name: sshkey
type: Opaque
data:
  authorizedkeys: PUBLIC_KEY
```

## Run the docker image in Kubernetes

```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sshd-jumpserver-deployment
spec:
  replicas: 1
  selector:
    app: sshd-jumpserver
  template:
    metadata:
      labels:
        app: sshd-jumpserver
    spec:
      containers:
      - name: sshd-jumpserver
        image: kubernetesio/sshd-jumpserver
        ports:
          - containerPort: 22
        env:
          - name: PUBLIC_KEY
            valueFrom:
              secretKeyRef:
                name: sshkey
                key: authorizedkeys

---
apiVersion: v1
kind: Service
metadata:
  name: sshd-jumpserver-svc
  labels:
    name: sshd-jumpserver-svc
spec:
  ports:
    - name: ssh
      port: 22
  type: "LoadBalancer"
  selector:
    app: sshd-jumpserver

```

## find the endpoint and ssh to the jump server

```
kubectl describe service sshd-jumpserver-svc

Name:           sshd-jumpserver-svc
Namespace:      default
Labels:         name=sshd-jumpserver-svc
Selector:       app=sshd-jumpserver
Type:           LoadBalancer
IP:         10.0.43.1
LoadBalancer Ingress:   ac646353e0e3e11e6bd02065967720c2-558922547.us-west-1.elb.amazonaws.com
Port:           ssh 22/TCP
NodePort:       ssh 30583/TCP
Endpoints:      10.244.4.10:22
Session Affinity:   None
No events.
```

then you can ssh to the jump server with the private key

```
ssh -i sshkeys/id_rsa root@ac646353e0e3e11e6bd02065967720c2-558922547.us-west-1.elb.amazonaws.com

Warning: Permanently added the ECDSA host key for IP address '54.219.157.181' to the list of known hosts.
[root@sshd-jumpserver-rc-oj6bv ~]#
```


[a2e8ebdf]: https://github.com/kubernetes-contrib/jumpserver "jumpserver-kubernetes"
