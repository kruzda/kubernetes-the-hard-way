# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Copy the `encryption-config.yaml` encryption config file to each controller instance:

```
source="encryption-config.yaml"
for servername in controller-0 controller-1 controller-2; do
  server_id=$(api_call $cs_ep/servers | jq -r '.servers[] | select(.name == "'$servername'") | .id')
  target_ip=$(api_call $cs_ep/servers/$server_id/ips | jq -r '.addresses["'$netname'"][] | select(.version == 4) | .addr')
  scp -i $private_key_file -o StrictHostKeyChecking=no $source $user@$target_ip:$target_path
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)
