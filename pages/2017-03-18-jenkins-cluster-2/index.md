---
title: Production Grade Jenkins Cluster in AWS (2)
date: "2017-03-18T13:11:54Z"
layout: post
path: "/production-grade-jenkins-cluster-aws-2/"
tags:
  - Jenkins
  - AWS
  - enterprise solution
  - production grade
---

(continue.)

## Tools and Plugins

Jenkins allows us to run all kind of jobs for automations as long as Jenkins can find the binaries, e.g. maven, ant, JDK.

### challenges:
1. The installation path for each tool has to be same on all nodes. Jenkins administrators probably need a configurations management tools to install and config tools.
2. For some tools, we may need install multiple versions to the Jenkins server. e.g. JDK 6,7,8 need all be installed and maven or ant need able to run with different JDK versions. This makes the configuration very complicate.

### solutions:
I baked the tools to docker containers. And instead of run the tools on Jenkins node directly, we run it in a Docker container with the workspace mounted.

e.g. [java-compiler docker ](https://github.com/c4po/java-compiler)

```
def mvnBuild(target, options = "") {
    sh "docker pull kubernetesio/java-compiler"
    sh "docker run --rm \
        -v `pwd`:/root \
        -v ${JENKINS_HOME}/.m2:/root/.m2 \
        kubernetesio/java-compiler mvn ${target} ${options}"
}

mvnBuild("package", "-Dmaven.test.skip=true")
```


## Agents auto discovery

### challenges:
1. Agents will be put into an ASG. When new agent is launched, it need to be able to find the master node.

### solutions:
I'm using the Jenkins Swarm plugin to manage the agent registration. When the Jenkins master is created in AWS, I added some tags to the EC2 instance for agents be able to find it.

here is what I put in the user-data for Agent Launch configuration.
```
...
JENKINS_MASTER=$(aws ec2 describe-instances --region=us-east-1\
 --filters "Name=tag:JenkinsCluster,Values=${jenkins_FQDN}" "Name=tag:jenkins/master,Values=1"\
 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text)

java -Djava.io.tmpdir=/root/tmp -jar /root/swarm-client.jar -fsroot /root/ -name ${jenkins_FQDN} -executors 10 -master http://$${JENKINS_MASTER}:8080/ >> /root/jenkins.log &
```
