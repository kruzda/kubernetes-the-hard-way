# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [cri-containerd](https://github.com/kubernetes-incubator/cri-containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Prerequisites

The commands in this lab must be run on each controller instance: `worker-0`, `worker-1`, and `worker-2`. The following command can be used to login to the server specified in the `servername` environment variable:

> Note: you must set the private_key_file environment variable to the private member of the keypair specified when creating the compute servers

```
private_key_file="$HOME/kubernetes-the-hard-way.pem"
servername="worker-0"; user="root"; netname="public"; ipv=4; target_ip=$(api_call $cs_ep/servers/detail?name=$servername | jq -r '.servers[].addresses["'$netname'"][] | select(.version == '$ipv') | .addr') && ssh -i $private_key_file -o StrictHostKeyChecking=no $user@$target_ip
```

## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
apt-get -y install socat
```

> The socat binary enables support for the `kubectl port-forward` command.

### Download and Install Worker Binaries

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://github.com/kubernetes-incubator/cri-containerd/releases/download/v1.0.0-alpha.0/cri-containerd-1.0.0-alpha.0.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.8.0/bin/linux/amd64/kubelet
```

Create the installation directories:

```
mkdir -pv \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
```

```
tar -xvf cri-containerd-1.0.0-alpha.0.tar.gz -C /
```

```
install -v -m 700 kubectl kube-proxy kubelet /usr/local/bin/
```

### Configure CNI Networking

Retrieve the Pod CIDR range for the current compute instance:

```
POD_CIDR=$(xenstore-read vm-data/user-metadata/pod-cidr | tr -d '"')
```

Disable the route that would conflict with the one being set up in the next steps by CNI:

```
i=$(hostname | cut -d'-' -f2)
ip r del 10.200.$i.0/24 via 10.240.0.2$i dev eth2
sed -i 's/\(.*10\.200\.'$i'\.0\)/#\1/g' /etc/network/interfaces
```

Create the `bridge` network configuration file:

```
cat > /etc/cni/net.d/10-bridge.conf <<EOF
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create the `loopback` network configuration file:

```
cat > /etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.3.1",
    "type": "loopback"
}
EOF
```

### Configure the Kubelet

```
mv -v ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
```

```
mv -v ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
```

```
mv -v ca.pem /var/lib/kubernetes/
```

Create the `kubelet.service` systemd unit file:

```
cat > /etc/systemd/system/kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=cri-containerd.service
Requires=cri-containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --allow-privileged=true \\
  --anonymous-auth=false \\
  --authorization-mode=Webhook \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --cluster-dns=10.32.0.10 \\
  --cluster-domain=cluster.local \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/cri-containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --pod-cidr=${POD_CIDR} \\
  --register-node=true \\
  --require-kubeconfig \\
  --runtime-request-timeout=15m \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.pem \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy

```
mv -v kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the `kube-proxy.service` systemd unit file:

```
cat > /etc/systemd/system/kube-proxy.service <<EOF
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --cluster-cidr=10.200.0.0/16 \\
  --kubeconfig=/var/lib/kube-proxy/kubeconfig \\
  --proxy-mode=iptables \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Worker Services

```
systemctl daemon-reload
```

```
systemctl enable --now containerd cri-containerd kubelet kube-proxy
```

> Remember to run the above commands on each worker node: `worker-0`, `worker-1`, and `worker-2`.

## Verification

Login to one of the controller nodes:

```
servername="controller-0"; user="root"; netname="public"; ipv=4; target_ip=$(api_call $cs_ep/servers/detail?name=$servername | jq -r '.servers[].addresses["'$netname'"][] | select(.version == '$ipv') | .addr') && ssh -i $private_key_file -o StrictHostKeyChecking=no $user@$target_ip
```

List the registered Kubernetes nodes:

```
kubectl get nodes
```

> output

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    1m        v1.8.0
worker-1   Ready     <none>    1m        v1.8.0
worker-2   Ready     <none>    1m        v1.8.0
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
