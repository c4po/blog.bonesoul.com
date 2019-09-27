---
title: Production Grade Jenkins Cluster in AWS (1)
date: "2017-03-04T13:11:54Z"
description: Production Grade Jenkins Cluster in AWS
---


Jenkins is the most popular CI/CD tool used by enterprise. It's very easy to install and setup a Jenkins server for a small team. However, to build a production grade Jenkins cluster for a big organization is still a big challenge. Before we compare different solutions, let's define some criteria which we think is important for a "Production Grade" Jenkins server.

1. Maintenance

  One common situation I found in big organization is they usually let application team build and manage their own Jenkins server.

  First, more Jenkins server means more Jenkins administrators you need to manage them.

  Secondly, these Jenkins servers may have totally different version and different plugins installed. A job running on one Jenkins server may not able to run on another Jenkins server because it may not have the required tools, plugin or credential installed.

2. Auto scaling

  Usually the CI/CD server are only busy during business hours. A production grade Jenkins cluster should able to handle the load during the peak hours and release the resource when off peak.

3. Versatile

  Be able to provide different build environments at same time. For example, we may need different version of JDK, Maven, Ant, Ruby etc. available on the Jenkins cluster. We may need different AWS profiles, docker accounts, maven settings available on the Jenkins cluster for different team to use at same time.

4. Up to date

  Jenkins community is one of the most active open source community. There are new releases of Jenkins and plugins every week. A production grade Jenkins cluster should be easy to upgrade without impact the end user.

5. Reliability

  A production grade Jenkins cluster should be high available in the company and easy to recover from any kind of disaster.

6. Infrastructure as code

  The Jenkins server should be consider as part of infrastructure in an organization. The creation of the Jenkins cluster should be fully managed by code. Which means the administrator should be able to build the cluster with a single command or a click. And the cluster configurations should be stored in a git repository.

As I'm a fan of [Kubernetes][54556c11], I was building a Jenkins cluster in Docker and run it in a Kubernetes cluster for one of my client. It's easy to manage and upgrade meet many our criteria. However, we get some issue when run docker in docker on the Kuberenets cluster.

In the next blog, I will show the solution that I designed for one of my client how to build a production grade Jenkins cluster in AWS with Terraform code.

[54556c11]: https://kubernetes.io/ "Kubernetes"
