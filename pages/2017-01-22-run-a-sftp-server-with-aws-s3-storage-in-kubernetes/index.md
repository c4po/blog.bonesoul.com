---
title: Hello World
date: "2017-01-23T00:13:54Z"
layout: post
path: "/sftp-s3-kubernetes/"
tags:
  - aws s3
  - Kubernetes
  - sftp
---

# Run a SFTP server with AWS S3 storage in Kubernetes

## TL;DR

```
git clone https://github.com/c4po/docker-s3fs.git
export AWS_ACCESS_KEY_ID=xxxxx
export AWS_SECRET_ACCESS_KEY=xxxx
export SFTP_USER=admin
export SFTP_PASSWORD=password
export SSH_KEY=~/.ssh/id_rsa.pub
export S3_BUCKET=mybucket
export S3_KEY=/
make
```

```
sftp -i ~/.ssh/id_rsa -P 30022 admin@aa.bb.cc.dd
```

### Step 1: Mount AWS S3 to Linux

[s3fs-fuse][81f28669] allows Linux and Mac OS X to mount an S3 bucket via FUSE.

### Step 2: Dockerize s3fs-fuse

Dockerize [s3fs-fuse][81f28669] is straightforward. However, there are some considerations about how to run s3fs and sftp server in the container. They can be both run as daemon process and foreground application. Here we choose to run s3fs as daemon process and sshd in foreground mode.

I also tried to run s3fs and sshd in 2 containers and use [data volume][07eeff1a] to share data between them. However, [s3fs-fuse][81f28669] is using [libfuse][92d36792] to manage the filesystem mount. It cannot be recognized by docker volume.

### Step 3: Run in Kubernetes

There are 2 things we should consider when run this in Kubernetes.

- Secrets

We don't want to put the AWS key to the pod definitions, so they need be read from secrets.

Load password into Kubernetes secets.
```
apiVersion: v1
kind: Secret
metadata:
  name: s3fs-secret
type: Opaque
data:
  sftp_user: YWRtaW4= # admin
  sftp_password: cGFzc3dvcmQ= # password
  aws_accesskey: xxxx
  aws_secretkey: xxxx
```

use secrets as environment variables:
```
spec:
  containers:
    env:
    - name: AWS_ACCESSKEY
      valueFrom:
        secretKeyRef:
          name: s3fs-secret
          key: aws_accesskey
```

- docker privilege

We need give the container enough privilege to allow [s3fs-fuse][81f28669] to manage filesystem.
```
spec:
  containers:
    securityContext:
      privileged: true
```
## Something about base64

When we manually create secrets in Kubernetes, the value need be base64 encoded. `echo 'admin' | base64` is different than `echo -n 'admin' | base64`. We don't want to have the newline character in the value. So be careful to use `echo -n` or `printf` to remove them.

## ToDo:
- Using AWS IAM role instead of pass AWS KEY to container


[81f28669]: https://github.com/s3fs-fuse/s3fs-fuse "s3fs"
[92d36792]: https://github.com/libfuse/libfuse "FUSE"
[07eeff1a]: https://docs.docker.com/engine/tutorials/dockervolumes/ "dockervolume"
