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

In our case, we have local registry, so you have to tell skaffold where to push image

```sh
skaffold dev --default-repo 10.60.6.216 --profile cluster
```

You can use `kubectl get pods --all-namespaces` to check if the application if successfully deployed.

## Make port forward to local

Above steps perform how to deploy app to k8s cluster by skaffold, but in actual developement, we often use port forward to directly access our app on local machine, and skaffold also support this function.

Add portForward section in your skaffold.yaml

```yaml
profiles:
    ....
    portForward:
    - resourceType: pod
      resourceName: gettting-started
      namespace: development
      port: 8080
      localPort: 9000
```

Now, we expose the pod 8080 port to local 9000

```sh
skaffold dev --default-repo 10.60.6.216 --profile cluster --port-forward
```

We can access our web service on local port 9000


## Remote debug your deployment

Skaffold support the remote debug and Google's [cloud code](https://cloud.google.com/code/) is an integration tools in vs code. It simplize the skaffold usage, and you can adjust the k8s cluster settings on the control pane. The next, I'll record the steps how to enable remote debug on k8s cluster in vs code.

* Enable dlv in your docker image

[Dlv](https://github.com/go-delve/delve) is the debugger for go-lang, for debugging, we have to install dlv in our container

```dockerfile
FROM golang:1.12-alpine AS go-builder
LABEL stage=build

WORKDIR /go/src/example-k8s-app
COPY . .
RUN apk add git build-base && \
    go get -u github.com/go-delve/delve/cmd/dlv
RUN cd /go/src/example-k8s-app && \
    CGO_ENABLE=0 GO111MODULE=on GOOS=linux GOARCH=amd64 go build -gcflags "all=-N -l" -o app cmd/main.go
ENV PORT 8080
EXPOSE 8080
CMD ["dlv", "exec", "/go/src/example-k8s-app/app", "--headless", "--listen=:3000", "--log", "--continue", "--accept-multiclient", "--api-version=2"]
```

Notice the listen argument in cmd, you can change the port number you want, here we use 3000 as demostration. The port number will use in later, please remember.

* Add debug configuration in vs code

launch.json

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "type": "cloudcode",
            "language": "Go",
            "request": "attach",
            "debugPort": 3000,
            "localRoot": "${workspaceFolder}",
            "remoteRoot": "/go/src/example-k8s-app",
            "name": "Attach App on Kubernetes Cluster: Go",
            "podSelector": {
                "app": "getting-started"
            },
            "imageRegistry": "10.60.6.216"
        }
    ]
}
```

Replace the debug port number that you filled in the previous step.

* Deploy app

Open a new terminal to run

```sh
skaffold dev --default-repo 10.60.6.216 --profile cluster --port-forward
```

* Attach the container's debug to vs code

Click the debug icon in control pane, and run the launch we add before.

Set the breakpoint in handler in your main.go, and connect http://localhost:9000 in browser. You will see we hit the breakpoint in vs code like

![this](https://hyt0617.github.io/notes/assets/img/debug-cluster.png)