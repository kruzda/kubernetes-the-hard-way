# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single region.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Cloud Network

In this section a dedicated [Cloud Network](https://developer.rackspace.com/docs/cloud-networks/v2/) will be setup for the Kubernetes cluster communicate through.

Create the `kubernetes-the-hard-way` Cloud Network:

```
netname="kubernetes-the-hard-way"
network_id=$(api_call $cn_ep/networks -X POST -d '{"network": {"name": "'$netname'"}}' | jq -r '.network.id')
```

A [subnet](https://developer.rackspace.com/docs/cloud-networks/v2/getting-started/concepts/#subnet-concepts) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet associated with the `kubernetes-the-hard-way` Cloud Network:

```
subnetname="kubernetes"
cidr="10.240.0.0/24"
subnet_id=$(api_call $cn_ep/subnets -X POST -d '{"subnet": {"name": "'$subnetname'", "network_id": "'$network_id'", "ip_version": "4", "cidr": "'$cidr'", "host_routes": [{"destination": "10.200.0.0/24", "nexthop": "10.240.0.20"}, {"destination": "10.200.1.0/24", "nexthop": "10.240.0.21"}, {"destination": "10.200.2.0/24", "nexthop": "10.240.0.22"}]}}' | jq -r .subnet.id)
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

Create [Cloud Network Ports](https://developer.rackspace.com/docs/cloud-networks/v2/getting-started/concepts/#port-concepts) for the controller nodes:

```
for i in 0 1 2; do
  limit=$(api_call $cn_ep/limits | jq '.limits.rate[] | select(.uri == "DefaultPortsPOST") | .limit[]')
  while [ $(echo $limit | jq -r '.remaining') -lt 1 ]; do
    delay=$(($(date -d $(echo $limit | jq -r '.["next-available"]') +%s) - $(date +%s) + 1))
    echo "Rate limited! Next API call may be sent in $delay seconds"
    sleep $delay
    limit=$(api_call $cn_ep/limits | jq '.limits.rate[] | select(.uri == "DefaultPortsPOST") | .limit[]')
  done
  api_call $cn_ep/ports -X POST -d '{"port": {"name": "controller-'$i'", "network_id": "'$network_id'", "fixed_ips": [{"subnet_id": "'$subnet_id'", "ip_address": "10.240.0.1'$i'"}]}}' | jq
done
```

Create the network ports for the worker nodes:

```
for i in 0 1 2; do
  limit=$(api_call $cn_ep/limits | jq '.limits.rate[] | select(.uri == "DefaultPortsPOST") | .limit[]')
  while [ $(echo $limit | jq -r '.remaining') -lt 1 ]; do
    delay=$(($(date -d $(echo $limit | jq -r '.["next-available"]') +%s) - $(date +%s) + 1))
    echo "Rate limited! Next API call may be sent in $delay seconds"
    sleep $delay
    limit=$(api_call $cn_ep/limits | jq '.limits.rate[] | select(.uri == "DefaultPortsPOST") | .limit[]')
  done
  api_call $cn_ep/ports -X POST -d '{"port": {"name": "worker-'$i'", "network_id": "'$network_id'", "fixed_ips": [{"subnet_id": "'$subnet_id'", "ip_address": "10.240.0.2'$i'"}]}}' | jq
done
```

### Kubernetes Public IP Address

> A [Cloud Load Balancer](https://developer.rackspace.com/docs/cloud-load-balancers/v1/) will be used to expose the Kubernetes API Servers to remote clients.

Allocate the load balancer fronting the Kubernetes API Servers:

```
lbport="6443"
lbname="kubernetes-api"
lb_details=$(api_call $lb_ep/loadbalancers -X POST -d '{"loadBalancer": {"name": "'$lbname'", "port": '$lbport', "protocol": "HTTPS", "virtualIps": [{"type": "PUBLIC"}], "healthMonitor": {"type": "HTTPS", "delay": 10, "timeout": 5, "attemptsBeforeDeactivation": 2, "path": "/version", "statusRegex": "^200$"}}}')
lb_id=$(echo $lb_details | jq -r '.loadBalancer.id')
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/kubernetes-incubator/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Setting up an SSH key to log in with

Create a new SSH keypair to be used (a) or upload the public key (b) that you wish to deploy on the servers:

a)
```
keypair_name="kubernetes-the-hard-way"
api_call $cs_ep/os-keypairs -X POST -d '{"keypair": {"name": "'$keypair_name'"}}' | jq -r '.keypair.private_key' | tee $HOME/$keypair_name.pem
chmod 600 $keypair_name.pem
private_key_file="$HOME/$keypair_name.pem"
```

b)
```
public_key_file="$HOME/.ssh/id_rsa.pub"
private_key_file="$HOME/.ssh/id_rsa"
keypair_name="kubernetes-the-hard-way"
echo '{"keypair": {"name": "'$keypair_name'", "public_key": "'$(cat $public_key_file)'"}}' >/tmp/pubkey-to-upload
api_call $cs_ep/os-keypairs -X POST -d @/tmp/pubkey-to-upload | jq
```


### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
controller_flavor="general1-2"
compute_image="0401bbaf-ea2c-4863-b6fc-001b48f3cb3c"
pubnetuuid="00000000-0000-0000-0000-000000000000"
sernetuuid="11111111-1111-1111-1111-111111111111"
for i in 0 1 2; do
  servername="controller-$i"
  port_id=$(api_call $cn_ep/ports?name=$servername | jq -r '.ports[].id')
  api_call $cs_ep/servers -X POST -d '{"server": {"name": "'$servername'", "imageRef": "'$compute_image'", "flavorRef": "'$controller_flavor'", "key_name": "'$keypair_name'", "metadata": {"group": "kubernetes-the-hard-way"}, "networks": [{"uuid": "'$pubnetuuid'"}, {"uuid": "'$sernetuuid'"}, {"port": "'$port_id'"}], "config_drive": true, "user_data": "I2Nsb3VkLWNvbmZpZwoKcGFja2FnZXM6CiAtIGpxCg=="}}' | jq
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.


Create three compute instances which will host the Kubernetes worker nodes:

```
worker_flavor="general1-2"
for i in 0 1 2; do
  servername="worker-$i"
  port_id=$(api_call $cn_ep/ports?name=$servername | jq -r '.ports[].id')
  api_call $cs_ep/servers -X POST -d '{"server": {"name": "'$servername'", "imageRef": "'$compute_image'", "flavorRef": "'$worker_flavor'", "key_name": "'$keypair_name'", "metadata": {"pod-cidr": "10.200.'$i'.0/24", "group": "kubernetes-the-hard-way"}, "networks": [{"uuid": "'$pubnetuuid'"}, {"uuid": "'$sernetuuid'"}, {"port": "'$port_id'"}], "config_drive": true, "user_data": "I2Nsb3VkLWNvbmZpZwoKcGFja2FnZXM6CiAtIGpxCg=="}}' | jq
done
```

### Verification

List the Cloud Servers:

```
api_call $cs_ep/servers/detail | jq '.servers[] | select(.metadata["group"] != null) | select(.metadata.group == "kubernetes-the-hard-way") | {name: .name, status: .status, task_status: .["OS-EXT-STS:task_state"], progress: .progress, internal_ip: .addresses["kubernetes-the-hard-way"][] | .addr}'
```

> output

```
{
  "name": "worker-2",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.22"
}
{
  "name": "worker-1",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.21"
}
{
  "name": "worker-0",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.20"
}
{
  "name": "controller-2",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.12"
}
{
  "name": "controller-1",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.11"
}
{
  "name": "controller-0",
  "status": "ACTIVE",
  "task_state": null,
  "progress": 100,
  "internal_ip": "10.240.0.10"
}
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
