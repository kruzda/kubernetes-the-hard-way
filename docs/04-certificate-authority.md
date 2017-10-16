# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kubelet, and kube-proxy.

## Certificate Authority

In this section you will provision a Certificate Authority that can be used to generate additional TLS certificates.

Create the CA configuration file:

```
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Create the CA certificate signing request:

```
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem
ca.pem
```

## Client and Server Certificates

In this section you will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes `admin` user.

### The Admin Client Certificate

Create the `admin` client certificate signing request:

```
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the `admin` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Results:

```
admin-key.pem
admin.pem
```

### The Kubelet Client Certificates

Kubernetes uses a [special-purpose authorization mode](https://kubernetes.io/docs/admin/authorization/node/) called Node Authorizer, that specifically authorizes API requests made by [Kubelets](https://kubernetes.io/docs/concepts/overview/components/#kubelet). In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:

```
for instance in worker-0 worker-1 worker-2; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
	"algo": "rsa",
	"size": 2048
  },
  "names": [
	{
	  "C": "US",
	  "L": "Portland",
	  "O": "system:nodes",
	  "OU": "Kubernetes The Hard Way",
	  "ST": "Oregon"
	}
  ]
}
EOF

netname="public"
server_id=$(api_call $cs_ep/servers | jq -r '.servers[] | select(.name == "'$instance'") | .id ')
EXTERNAL_IP=$(api_call $cs_ep/servers/$server_id | jq -r '.server.addresses["'$netname'"][] | select(.version == 4) | .addr')

netname="kubernetes-the-hard-way"
INTERNAL_IP=$(api_call $cs_ep/servers/$server_id | jq -r '.server.addresses["'$netname'"][] | select(.version == 4) | .addr')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Results:

```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
worker-2-key.pem
worker-2.pem
```

### The kube-proxy Client Certificate

Create the `kube-proxy` client certificate signing request:

```
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the `kube-proxy` client certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Results:

```
kube-proxy-key.pem
kube-proxy.pem
```

### The Kubernetes API Server Certificate

The public IP of the `kubernetes-api` load balancer will be included in the list of subject alternative names for the Kubernetes API Server certificate. This will ensure the certificate can be validated by remote clients.

Retrieve the public IP address of the `kubernetes-api` load balancer:

```
lbname="kubernetes-api"
ipv="4"
iptype="PUBLIC"
KUBERNETES_PUBLIC_ADDRESS=$(api_call $lb_ep/loadbalancers | jq -r '.loadBalancers[] | select(.name == "'$lbname'") | .virtualIps[] | select(.ipVersion == "IPV'$ipv'" and .type == "'$iptype'") | .address')
```

Create the Kubernetes API Server certificate signing request:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the Kubernetes API Server certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.pem
```

## Distribute the Client and Server Certificates

Copy the appropriate certificates and private keys to each worker instance:

```
user="root"
target_path="~/"
netname="public"
for servername in worker-0 worker-1 worker-2; do
  server_id=$(api_call $cs_ep/servers | jq -r '.servers[] | select(.name == "'$servername'") | .id')
  target_ip=$(api_call $cs_ep/servers/$server_id/ips | jq -r '.addresses["'$netname'"][] | select(.version == 4) | .addr')
  source="ca.pem ${servername}-key.pem ${servername}.pem"
  ssh-keygen -R $target_ip
  scp -i $private_key_file -o StrictHostKeyChecking=no $source $user@$target_ip:$target_path
done
```

Copy the appropriate certificates and private keys to each controller instance:

```
source="ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem"
for servername in controller-0 controller-1 controller-2; do
  server_id=$(api_call $cs_ep/servers | jq -r '.servers[] | select(.name == "'$servername'") | .id')
  target_ip=$(api_call $cs_ep/servers/$server_id/ips | jq -r '.addresses["'$netname'"][] | select(.version == 4) | .addr')
  ssh-keygen -R $target_ip
  scp -i $private_key_file -o StrictHostKeyChecking=no $source $user@$target_ip:$target_path
done
```

> The `kube-proxy` and `kubelet` client certificates will be used to generate client authentication configuration files in the next lab.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)
