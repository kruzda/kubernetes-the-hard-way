# Cleaning Up

In this labs you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
for class in controller worker; do for i in 0 1 2; do servername="$class-$i"; server_id=$(api_call $cs_ep/servers | jq -r '.servers[] | select(.name == "'$servername'") | .id'); api_call -X DELETE $cs_ep/servers/$server_id; done; done
```

Delete the keypair:

```
api_call $cs_ep/os-keypairs/$keypair_name -X DELETE
```

## Networking

Delete the Cloud Load Balancer:

```
lb_id=$(api_call $lb_ep/loadbalancers | jq -r '.loadBalancers[] | select(.name == "'$lbname'") | .id')
api_call $lb_ep/loadbalancers/$lb_id -X DELETE
```

Delete the `kubernetes-the-hard-way` network:

```
netname="kubernetes-the-hard-way"
network_id=$(api_call $cn_ep/networks | jq  -r '.networks[] | select(.name == "'$netname'") | .id')
api_call $cn_ep/networks/$network_id -X DELETE
```
