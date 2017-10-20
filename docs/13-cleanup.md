# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete all the compute instances with the "kubernetes-the-hard-way" `group` metadata item:

```
for sid in $(api_call $cs_ep/servers/detail | jq -r '.servers[] | select(.metadata.group != null) | select(.metadata.group == "kubernetes-the-hard-way") | .id'); do api_call $cs_ep/servers/$sid -X DELETE; done
```

> Note: this also deletes the network port associated with the server


Delete the keypair:

```
keypair_name="kubernetes-the-hard-way"
api_call $cs_ep/os-keypairs/$keypair_name -X DELETE
```

## Networking

Delete the Cloud Load Balancer named "kubernetes-api":

```
lbname="kubernetes-api"
lb_id=$(api_call $lb_ep/loadbalancers | jq -r '.loadBalancers[] | select(.name == "'$lbname'") | .id')
api_call $lb_ep/loadbalancers/$lb_id -X DELETE
```

Delete the network named "kubernetes-the-hard-way":

```
netname="kubernetes-the-hard-way"
network_id=$(api_call $cn_ep/networks?name=$netname | jq  -r '.networks[].id')
api_call $cn_ep/networks/$network_id -X DELETE
```

> Note: this also deletes the subnet associated with this network
