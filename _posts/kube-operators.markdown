---
layout: post
title:  How we manage projects on our machine learning platform
date:   2021-04-22 17:00:00 +0200
image:  project-mgmt.jpg
tags:   Kubernetes Ops
seo:
  type: BlogPosting
---

I'm building a *multi-tenant* machine learning platform for my current client.
This platform provides data scientists with everything they need to successfully productionize their models.
The platform also takes care of things like TLS and authentication of APIs and abstracts this away so the user doesn't have to deal with it.

We try to make data scientists and machine learning engineers completely *autonomous*.
They shouldn't require the help of my team to work with the platform.
By writing extensive documentation, we're doing a pretty good job at fulfilling that goal.
We're not there yet though.
One thing we initially struggled with was *project initiation*.
Let's say a data scientist wants to start productionizing a new application.
Due to security requirements, we want to completely separate this application from other applications, so we give this new application an isolated Kubernetes namespace, complete with network policies, resource quota, pod security policies, you name it.
The data scientist also requested an S3 bucket, a Keycloak client for API authentication, a Harbor project for storing their Docker images and preferably a service account for accessing all of the above.
We used to create all of this manually.
It sucked.

## Kubernetes controller

We decided to automate the entire onboarding process for new machine learning models.
After some research we decided to build a Kubernetes controller to take care of this.
In the end, we are able to create a `Project` custom resource in Kubernetes like this:
```yaml
apiVersion: project.example.com/v1
kind: Project
metadata:
  name: example-project
spec:
  owners:
    - data.scientist@example.com
    - mle@example.com
  resourceQuota:
    hard:
      cpu: "16"
      memory: 32Gi
  idp:
    redirectUris:
      - https://example.com
```
This resource contains all the information we need to create resources the data scientist needs for this project.
Also, if the resource is changed (for example, by adding or removing an entry from the `owners` list), all permissions in Kubernetes, S3, Harbor and Keycloak are updated as well.

## Kubebuilder

We used [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) to build our project controller.