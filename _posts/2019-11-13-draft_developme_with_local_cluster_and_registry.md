---
layout: post
title: Draft development with local kubuernetes cluster
date: 2019-11-13 17:00:00 +0800
description: Use draft to publish app image to local secure registry and deploy app to local k8s cluster in vs code. # Add post description (optional)
img: # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Draft, VSCode, k8s, kubernetes]
---

Since I could see the local kubenetes cluster in vscode, so I wanna try to deploy a application by draft to my local cluster. Here's my steps to archieve the goal.


## Prerequistes

* Local k8s cluster

* VS code

## Generate self-signed cerification

First, we have to generate self-signed certification, we'll use ssl.conf to generate the key & certification. You could replace the fields that you need to change, like CN/DNS/IP etc.
 

ssl.conf

```
[req]
prompt = no
default_md = sha256
default_bits = 2048
distinguished_name = dn
x509_extensions = v3_req

[dn]
C = TW
ST = TPE
L = TPE
O = Test
OU = Test
CN = localhost

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.2 = k8s-master-node
DNS.3 = localhost
IP.1 = 10.60.6.216
```

Then, use openssl to generate the crt & key

```sh
openssl req -x509 -new -nodes -sha256 -days 3650 -newkey rsa:2048 -keyout server.key -out server.crt -config ssl.conf
```

## Setup secure docker registry

Because I don't want to use public docker registry, so I decide to setup my private docker registry for my k8s cluster. Please do this step on the master node.

```sh
docker run -d --restart=always --name registry -v $PWD/certs:/certs -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.crt -e REGISTRY_HTTP_TLS_KEY=/certs/server.key -p 443:443 registry:2
```

## Let docker login the private registry

We have to let docker to login the private registry we setup,
Perform this step on both master/slave k8s node. You can enter the username/password you'd like, please remeber the username you login, we'll use it for set secret in k8s.

```sh
mkdir -p /etc/docker/certs.d/10.60.6.216/
cp server.crt /etc/docker/certs.d/10.60.6.216/
docker login 10.60.6.216 -u {username} -p {password}
```

you should see **Login Succeeded** on command line


## Create k8s local registry

If your docker login successfully, then we have to create secret for it in k8s.

```sh
kubectl create secret docker-registry local-registry \
--docker-server=10.60.6.216 \
--docker-username={username} \
```

## Docker login on developement machine (mac osx)

Because my development environment on Mac, I have to let my docker desktop to recoginze my self-signed certification.

```sh
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain server.crt
```

Restart Docker desktop in order to make the certification valid, after restarting completed. Use username/password you create before to login.

```sh
docker login 10.60.6.216 -u {username} -p {password}
```

## Add registry field in draft.toml

By default, draft will push images to docker.io, so we have add our private registry to make sure draft will push the image to it.

```toml
[environments]
  [environments.development]
    name = "example-k8s-app"
    namespace = "development"
    registry = "10.60.6.216"
    ....
```

## Draft up it !

It's time to deploy our application to the local k8s cluster.

```sh
draft up
```

You can use `draft connect` to see the app if works.



## Reference

* [k8s-with-local-registry](https://tortuemat.gitlab.io/blog/2018/03/18/kubernetes-with-local-registry/)

* [setup-secure-registry](https://docs.docker.com/registry/deploying/)

* [add-self-signed-registry-certs-docker-mac](https://blog.container-solutions.com/adding-self-signed-registry-certs-docker-mac)