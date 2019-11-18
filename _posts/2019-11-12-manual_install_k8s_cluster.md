---
layout: post
title: Manual install k8s cluster
date: 2019-11-12 12:00:00 +0800
description: Setting up local k8s cluster # Add post description (optional)
img: # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [k8s, kubernetes]
---

It's time to experience the k8s cluster, since we already operated the minikube.

## Prerequisites

* 2 Ubuntu 18.04 VM, change the hostname into master/slave

* Kubeadm

* Kubectl

* Docker CE 19.03.4


Next steps all perform on the master node

## Disable all swap

We use virtual machine to build up the k8s cluster here, at first we have to disable all swap.

```sh
swapoff -a
```

## Change docker cgroup driver

It's suggestion to change default docker cgroupdriver from cgroup to systemd

```sh
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

systemctl restart docker
systemctl status docker
```

## Initialize cluster

You can change the network cidr which you want

```sh
kubeadm init --pod-network-cidr=10.244.0.0/16
```

## Copy kube config

Copy default kubeconfig from /etc/kubernetes to $HOME/.kub

```sh
mkdir -p $HOME/.kub
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Apply flannel network

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

## Join cluster on slave node

Replace the token/hash which generated after kubeadm init

```sh
kubeadm join 10.60.6.216:6443 --token q92xqc.s7r98tmschj6i08i     --discovery-token-ca-cert-hash sha256:2a55b833acf65ce42ad64bf3320ff8716cf7d9d3515b0f6d4e2672a975fc413d
```