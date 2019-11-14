---
layout: post
title: Use draft to deploy k8s app
date: 2019-11-11 15:00:00 +0800
description: Let's meet draft # Add post description (optional)
img: # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Draft, VSCode, k8s, kubernetes]
---

[Draft](https://github.com/Azure/draft) is the k8s development tool which released by microsoft, you can easily to deploy the k8s app to vary kind of k8s cluster. Let try it on the minikube

## Prerequistes

* minikube v1.5.2

* kubectl v1.16.2

* helm v2.16.0

* draft 0.16.0

* Golang

* VS code


First, please install all the necessary tools which we need.
And create a project in vs code, the folder struct like this:

```
example-k8s-app
│   draft.toml    
│
└───build
│   | Dockerfile
│   
└───cmd
│   │ main.go
│   
└───deployment
    | .dockerignore
    │ .draft-tasks.toml
    | .draftignore
    └───charts
```

This folder struct is slightly different with draft default struct, and I prefer to use the [golang's project layout](https://github.com/golang-standards/project-layout), I'll show how to ajust it to the standard's layout. And Create the deployment folder first,

```
mkdir deployment
```

Create draft files in the deployment folder

```
draft create deployment
```

You can find draft.toml under the deployment folder, let's move it to the project root. And change the configuration of docker file and add **image-build-args** the make sure the docker could build image successfully.

```
[environments]
  [environments.development]
    name = "example-k8s-app"
    namespace = "default"
    wait = true
    watch = false
    watch-delay = 2
    auto-connect = false
    dockerfile = "build/Dockerfile"
    image-build-args = ".."
    chart = "deployment/charts/example-k8s-app"
```


We'll use a simple go web server to be our first k8s app, put this file to the cmd/

```go
package main

import (
	"fmt"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World, Update Golang!")
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8080", nil)
}
```

You could move the dockerfile under the deployment to the build/folder or create your dockerfile in build folder. Here we use multi-stage to simplify our docker image size.

```
FROM golang:1.12-alpine AS go-builder
LABEL stage=build

ENV GO111MODULE on

WORKDIR /go/src/example-k8s-app
COPY . .
RUN apk add git && \
    cd /go/src/example-k8s-app && \
    CGO_ENABLE=0 GOOS=linux GOARCH=amd64 go build -o app cmd/main.go

FROM alpine AS base
WORKDIR /app
COPY --from=go-builder /go/src/example-k8s-app/app /app
ENV PORT 8080
EXPOSE 8080

CMD ["/app/app"]
```

Ok, I assume you have installed minibuke. Let's start the minikube.

```
    minikube start
``` 

Next, we have to initialize the helm, because the kubectl is v1.16, it's suggest to add `--upgarde` argument when you initialize the helm.

```
    helm init --upgrade
```

Configure Draft to build images directly using Minikube's Docker daemon

```
eval $(minikube docker-env)
```

Change apiVersion in **deployment/charts/example k8s-app/templates/deployment.yaml**, since kubectl v1.16 no longer support api version: *extensions/v1beta1*, so we have to change the apiVersion to *apps/v1*

Add below yaml under spec in **deployment/charts/example/k8s-app/templates/deployment.yaml**, we have to add this section in kubectl v1.16

```
selector:
   matchLabels:
     draft: {{ default "draft-app" .Values.draft }}
     app: {{ template "fullname" . }}
```

Ok, we can sail our first k8s app to the minikube!

```
   draft up
```


## Reference

* [Developing on k8s](https://kubernetes.io/blog/2018/05/01/developing-on-kubernetes/)