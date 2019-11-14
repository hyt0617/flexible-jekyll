## Description

Connect to the local k8s cluster in vs code

## Prerequsites

* k8s cluster

* vs kubernetes extension


## Steps

* Create client key

```
openssl genrsa -out client.key 4096
```

* Prepare conf file to generate CSR

```
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
[ dn ]
CN = development => replace the common name you want
O = development => replace the organize name you want
[ v3_ext ]
authorityKeyIdentifier=keyid,issuer:always
basicConstraints=CA:FALSE
keyUsage=keyEncipherment,dataEncipherment
extendedKeyUsage=serverAuth,clientAuth
```

* Generate CSR

```
openssl req -config ./csr.cnf -new -key client.key -nodes -out 
```

* Create csr.yaml for kubectl

```
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: mycsr
spec:
  groups:
  - system:authenticated
  request: ${BASE64_CSR}
  usages:
  - digital signature
  - key encipherment
  - server auth
  - client auth
```

* Apply csr.yaml

```
export BASE64_CSR=$(cat ./client.csr | base64 | tr -d '\n')
cat csr.yaml | envsubst | kubectl apply -f -
```

* Check csr in kubectl, the mysrc condition should be **Pending**

```
kubectl get csr
```

* Approve it

```
kubectl certificate approve mycsr
```

* Create a namespace, for example: development

```
kubectl create ns development
```

* Setting up RBAC role/cluster-role

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 namespace: development
 name: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "namespaces"]
  verbs: ["create", "get", "update", "list", "delete", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "get", "update", "list", "delete"]

----------------------------------------------------------------

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 namespace: development
 name: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services", "nodes", "namespaces"]
  verbs: ["create", "get", "update", "list", "delete", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "get", "update", "list", "delete"]

---------------------------------------------------------------

kubectl apply -f /path/file.yaml
```

* Setting up role/cluster-role binding

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: dev
 namespace: development
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
 kind: Role
 name: dev
 apiGroup: rbac.authorization.k8s.io

------------------------------------------------------------

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: dev
 namespace: development
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
 kind: ClusterRole
 name: dev
 apiGroup: rbac.authorization.k8s.io

------------------------------------------------------------

kubectl apply -f /path/file
```

* View kube config on cluster

```
kubectl config view --raw -o json
```

* Add kube config in vs code, replace all the necessary fields in previous step

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: {replace your ceritificate data here}
    server: {cluster-ip}
  name: kubernetes-admin@kubernetes
users:
- name: dev
  user: 
    client-certificate-data: client.crt
    client-key: client.key
contexts:
- context:
    cluster: kubernetes-admin@kubernetes
    user: dev
  name: dev-kubernetes-admin@kubernetes
current-context: dev-kubernetes-admin@kubernetes

```

* Apply config

```
export KUBECONFIG=$KUBECONFIG:$HOME/.kube/config:$PWD/config
```

## Reference

* [Access multiple cluster](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

* [Access k8s cluster](https://medium.com/better-programming/k8s-tips-give-access-to-your-clusterwith-a-client-certificate-dfb3b71a76fe)


