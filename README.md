# HashiCorp Vault Deployment on Kubernetes (Minikube) with Helm

## jq Installation

```bash
# Installing jq for JSON data manipulation
sudo apt-get install jq
```
## Namespace Creation
```bash
# Creating the 'vault' namespace to organize resources
bashkubectl create namespace vault

# Switching to the created namespace
bashkubectl config set-context --current --namespace=vault
```
## Helm Configuration
```bash
# Adding the HashiCorp Helm repository
helm repo add hashicorp https://helm.releases.hashicorp.com
# Searching for the version of the HashiCorp Vault chart in the repository
helm search repo hashicorp/vault
# Displaying available versions of the HashiCorp Vault chart
helm search repo hashicorp/vault --versions
```
## Helm Values File Creation
Create the file helm-vault-raft-values.yml with the following content:
```yaml
# Custom configuration for Vault deployment using Helm
# Filename helm-vault-raft-values.yml
server:
  affinity: ""
  ha:
    enabled: true
    raft:
      enabled: true
  ui:
    enabled: true
  serviceType: "NodePort"
  injector:
    enabled: true
```
```bash
# Installing Vault with Helm using custom values
helm install vault hashicorp/vault --values helm-vault-raft-values.yml -n vault
```
## Initialization and Unsealing of Vault
```bash
# Initializing Vault and obtaining keys for unsealing
kubectl exec vault-0 -- vault operator init \
  -key-shares=1 \
  -key-threshold=1 \
  -format=json > cluster-key.json

# Extracting the unseal key from the initialization output
VAULT_UNSEAL_KEY=$(jq -r '.unseal_keys_b64[]' cluster-key.json)

# Unsealing Vault with the obtained key on vault-0
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

# Joining vault-1 to the Raft cluster hosted by vault-0
kubectl exec -ti vault-1 -- vault operator raft join http://vault-0.vault-internal:8200

# Joining vault-2 to the Raft cluster hosted by vault-0
kubectl exec -ti vault-2 -- vault operator raft join http://vault-0.vault-internal:8200

# Unsealing vault-1 with the obtained key
kubectl exec vault-1 -- vault operator unseal $VAULT_UNSEAL_KEY

# Unsealing vault-2 with the obtained key
kubectl exec vault-2 -- vault operator unseal $VAULT_UNSEAL_KEY

# Port forwarding to access Vault locally
kubectl port-forward vault-0 8200:8200
```
## Access to Vault
```bash
# Accessing Vault from within the container vault-0
kubectl exec -ti vault-0 -- /bin/sh
vault login # It will prompt for your token obtained from the cluster-key.json file
```
## Official Documentation
 For more details and advanced configurations, refer to the [Official HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-minikube-raft)

