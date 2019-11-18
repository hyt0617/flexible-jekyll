---
layout: post
title: Skaffold development
date: 2019-11-18 09:31:00 +0800
description: Another k8s development tool # Add post description (optional)
img: # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [k8s, kubernetes, skaffold]
---

Difference with [Draft](https://github.com/Azure/draft), [Skaffold](https://github.com/GoogleContainerTools/skaffold) not only support continuous development, also support to create build blocks in CI/CD pipelines. Here we will try to deploy a simple application to our local k8s cluster with skaffold.

## Prerequites

* local k8s cluster

* Skaffold

## Create the pod.yaml first

We'll reuse the example application that we built before, please referance [this post](https://hyt0617.github.io/notes/draft_get_started/). Since we don't use minikube as our skaffold development, so we have to create our pod.yaml. Skaffold also support [Helm](https://helm.sh/), and we'll try it at next chapter. Now create a pod.yaml under deployment folder

development/k8s-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: getting-started
  namespace: development
spec:
  containers:
  - name: getting-started
    image: skaffold-example
```

You can change namespace to your own k8s namespace.


## Create skaffold.yaml

You can enter the development folder and run `skaffold init`, skaffold will create skaffold.yaml automatically. Just like we did in draft, we move the skaffold.yaml to the root path of the project. Now, we have add some settings in skaffold.yaml.

* Change deploy/kubectl/manifests path

* Add profile for our cluster, you can reference the [profile usage](https://skaffold.dev/docs/environment/profiles/).

```yaml
apiVersion: skaffold/v1
kind: Config
metadata:
  name: development
deploy:
  kubectl:
    manifests:
    - ./development/k8s-pod.yaml
profiles:
- name: cluster
  build:
    artifacts:
    - image: skaffold-example
      context: .
      docker:
        dockerfile: ./build/Dockerfile
        buildArgs:
  activation:
    - kubeContext: dev-kubernetes-admin@kubernetes
```

## Time to skaffold

Execute the `skaffold dev` from where the skaffold.yaml is. 