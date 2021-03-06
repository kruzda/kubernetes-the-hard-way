# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Retrieve the `kubernetes-the-hard-way` static IP address:

```
lb_name="kubernetes-api"
ipv=4
iptype="PUBLIC"
KUBERNETES_PUBLIC_ADDRESS=$(api_call $lb_ep/loadbalancers | jq -r '.loadBalancers[] | select(.name == "'$lbname'") | .virtualIps[] | select(.ipVersion == "IPV'$ipv'" and .type == "'$iptype'") | .address')
```

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443
```

```
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem
```

```
kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin
```

```
kubectl config use-context kubernetes-the-hard-way
```

## Verification

Check the health of the remote Kubernetes cluster:

```
kubectl get componentstatuses
```

> output

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
NAME       STATUS    ROLES     AGE       VERSION
worker-0   Ready     <none>    2m        v1.9.0
worker-1   Ready     <none>    2m        v1.9.0
worker-2   Ready     <none>    2m        v1.9.0
```

Next: [Deploying the DNS Cluster Add-on](11-dns-addon.md)
