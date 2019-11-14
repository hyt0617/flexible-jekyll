## Description

Install k8s cluster

## Prerequisites

* 2 Ubuntu 18.04 VM, change the hostname into master/slave

* Kubeadm

* Kubectl

* Docker CE 19.03.4

## Steps

1. Disable all swap

```
swapoff -a
```

2. Change docker cgroup driver

```
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

3. Initialize cluster, you can change the network cidr in others netmask

```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

4. Copy kube config

```
mkdir -p $HOME/.kub
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5. Apply flannel network

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

6. Join cluster (slave node), according the token which generated after kubeadm init
```
kubeadm join 10.60.6.216:6443 --token q92xqc.s7r98tmschj6i08i     --discovery-token-ca-cert-hash sha256:2a55b833acf65ce42ad64bf3320ff8716cf7d9d3515b0f6d4e2672a975fc413d
```