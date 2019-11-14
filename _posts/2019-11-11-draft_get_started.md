## Description

Use minikube + draft to deploy app on k8s cluster

## Prerequistes

* minikube v1.5.2

* kubectl v1.16.2

* helm v2.16.0

* draft 0.16.0

* Golang


## Main.go

```
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

## Dockerfile

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

## Draft.toml

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

## Steps

* Start a k8s local cluster
```
    minikube start
``` 

* Helm init
```
    helm init --upgrade
```

*  Configure Draft to build images directly using Minikube's Docker daemon
```
eval $(minikube docker-env)
```

* Make deployment directory

```
mkdir deployment
```

* Draft create
```
draft create deployment
```

* Move draft.toml from deployment to outside

* Change apiVersion in **deployment/charts/example/k8s-app/templates/deployment.yaml**

```
apiVersion: apps/v1
```

* Add below yaml under spec in **deployment/charts/example/k8s-app/templates/deployment.yaml**

```
selector:
   matchLabels:
     draft: {{ default "draft-app" .Values.draft }}
     app: {{ template "fullname" . }}
```

* Draft up
```
   draft up
```

## Folder struct

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
|   | .dockerignore
│   │ .draft-tasks.toml
|   | .draftignore
│   └───charts
```

## Reference

* [Developing on k8s](https://kubernetes.io/blog/2018/05/01/developing-on-kubernetes/)