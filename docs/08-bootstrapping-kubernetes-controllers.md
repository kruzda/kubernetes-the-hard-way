# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also add the controller nodes to the Cloud Load Balancer which exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `controller-0`, `controller-1`, and `controller-2`. The following command can be used to login to the server specified in the `servername` environment variable:

> Note: you must set the private_key_file environment variable to the private member of the keypair specified when creating the compute servers

```
private_key_file="$HOME/kubernetes-the-hard-way.pem"
servername="controller-0"; user="root"; netname="public"; ipv=4; target_ip=$(api_call $cs_ep/servers/detail?name=$servername | jq -r '.servers[].addresses["'$netname'"][] | select(.version == '$ipv') | .addr') && ssh -i $private_key_file -o StrictHostKeyChecking=no $user@$target_ip
```

## Provision the Kubernetes Control Plane

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl"
```

Install the Kubernetes binaries:

```
install -v -m 700 kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin
```

### Configure the Kubernetes API Server

```
mkdir -pv /var/lib/kubernetes/
```

```
mv -v ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem encryption-config.yaml /var/lib/kubernetes/
```

The instance internal IP address will be used advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

```
INTERNAL_IP=$(for n in $(xenstore-list vm-data/networking); do xenstore-read vm-data/networking/$n; done | jq -r '. | select(.label == "kubernetes-the-hard-way") | .ips[] | .ip')
```

Create the `kube-apiserver.service` systemd unit file:

```
cat > /etc/systemd/system/kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --admission-control=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --insecure-bind-address=127.0.0.1 \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-ca-file=/var/lib/kubernetes/ca.pem \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

Create the `kube-controller-manager.service` systemd unit file:

```
cat > /etc/systemd/system/kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/ca-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler

Create the `kube-scheduler.service` systemd unit file:

```
cat > /etc/systemd/system/kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --leader-elect=true \\
  --master=http://127.0.0.1:8080 \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

```
systemctl daemon-reload
```

```
systemctl enable --now kube-apiserver kube-controller-manager kube-scheduler
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.


### Verification

```
kubectl get componentstatuses
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

> Remember to run the above commands on each controller node: `controller-0`, `controller-1`, and `controller-2`.

## RBAC for Kubelet Authorization

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

> Note: this step will have to be run on a controller node. The following command can be used to login to the server specified in the `servername` environment variable:

```
servername="controller-0"; user="root"; netname="public"; ipv=4; target_ip=$(api_call $cs_ep/servers/detail?name=$servername | jq -r '.servers[].addresses["'$netname'"][] | select(.version == '$ipv') | .addr') && ssh -i $private_key_file -o StrictHostKeyChecking=no $user@$target_ip 
```

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## The Kubernetes Frontend Load Balancer

In this section you attach the Kubernetes API Servers to the Cloud Load Balancer.

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances.


```
lb_name="kubernetes-api"
netname="private"
lb_id=$(api_call $lb_ep/loadbalancers | jq -r '.loadBalancers[] | select(.name == "'$lb_name'") | .id')
for servername in controller-0 controller-1 controller-2; do
  server_ip=$(api_call $cs_ep/servers/detail?name=$servername | jq -r '.servers[].addresses["'$netname'"][] | select(.version == 4) | .addr')
  api_call $lb_ep/loadbalancers/$lb_id/nodes -X POST -d '{"nodes": [{"address": "'$server_ip'", "condition": "ENABLED", "port": "'$lbport'"}]}' | jq
  while [ $(api_call $lb_ep/loadbalancers/$lb_id | jq -r '.loadBalancer.status') != "ACTIVE" ] ; do
	echo "...waiting for CLB to become ACTIVE..."
	sleep 3
  done
done
```

### Verification

Check for node status:

```
api_call $lb_ep/loadbalancers/$lb_id/nodes | jq
```

> output

```
{
  "nodes": [
    {
      "address": "10.181.65.181",
      "id": 799325,
      "type": "PRIMARY",
      "port": 6443,
      "status": "ONLINE",
      "condition": "ENABLED"
    },
    {
      "address": "10.181.67.209",
      "id": 799327,
      "type": "PRIMARY",
      "port": 6443,
      "status": "ONLINE",
      "condition": "ENABLED"
    },
    {
      "address": "10.181.67.232",
      "id": 799329,
      "type": "PRIMARY",
      "port": 6443,
      "status": "ONLINE",
      "condition": "ENABLED"
    }
  ]
}
```

Make an HTTP request for the Kubernetes version info:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```
{
  "major": "1",
  "minor": "8",
  "gitVersion": "v1.8.0",
  "gitCommit": "6e937839ac04a38cac63e6a7a306c5d035fe7b0a",
  "gitTreeState": "clean",
  "buildDate": "2017-09-28T22:46:41Z",
  "goVersion": "go1.8.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
