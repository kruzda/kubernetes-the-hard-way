# Prerequisites

## A Rackspace Public Cloud account

This tutorial leverages the [Rackspace Public Cloud](https://www.rackspace.com/openstack/public) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. Sign up for a [UK account](https://cart.rackspace.com/en-gb/cloud) or a [global one](https://cart.rackspace.com/cloud).

## A Linux, Mac or Cygwin terminal

## Curl
https://curl.haxx.se/

## jq
https://stedolan.github.io/jq/

## Authentication to the Rackspace Cloud API

Set the following environment variables as per your Rackspace Cloud account username, its API-key and the [region](https://support.rackspace.com/how-to/about-regions) you wish to the cluster in.

```
RS_USER=""
RS_APIKEY=""
RS_REGION="" # the API refers to the region in all capitals. current regions available are: LON, DFW, IAD, ORD, HKG and SYD)
```

Once that is done you can run the one-liner below to receive an authentication token and a service catalog from wich the endpoints used in this guide are saved as variables.

```
catalog=$(curl -s https://identity.api.rackspacecloud.com/v2.0/tokens -d '{"auth":{"RAX-KSKEY:apiKeyCredentials":{"username": "'$RS_USER'", "apiKey": "'$RS_APIKEY'"}}}' -X POST -H "Content-type: application/json") && token=$(echo $catalog | jq -r .access.token.id) && cs_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "compute") | .endpoints[] | select(.region == "'$RS_REGION'") | .publicURL') && cn_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "network") | .endpoints[] | select(.region == "'$RS_REGION'") | .publicURL') && cf_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "object-store") | .endpoints[] | select(.region == "'$RS_REGION'") | .publicURL') && cfcdn_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "rax:object-cdn") | .endpoints[] | select(.region == "'$RS_REGION'") | .publicURL') && dns_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "rax:dns") | .endpoints[].publicURL') && lb_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "rax:load-balancer") | .endpoints[] | select(.region == "'$RS_REGION'") | .publicURL') && cm_ep=$(echo $catalog | jq -r '.access.serviceCatalog[] | select(.type == "rax:monitor") | .endpoints[].publicURL')
```

Set the following alias. This helps reducing the lenght of commands used in this guide making them easier to read.

```
alias api_call='curl -s -H "X-Auth-Token: '$token'" -H "Content-Type: application/json"'
```

Next: [Installing the Client Tools](02-client-tools.md)
