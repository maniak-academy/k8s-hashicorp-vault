# HashiCorp Vault Enterprise on Kubernetes

This repository contains configuration for deploying HashiCorp Vault Enterprise 1.19.0 on Kubernetes with ingress access.

## Components

- HashiCorp Vault Enterprise 1.19.0
- Vault Agent Injector 
- High Availability cluster with 3 Vault pods
- Ingress for UI access at vault.maniak.lab

## Deployment Details

The deployment is configured to use:
- Longhorn storage for persistent data
- Raft storage backend for Vault in HA mode
- Enterprise license mounted from a Kubernetes secret
- Nginx Ingress for access to the Vault UI

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- Longhorn storage class
- Nginx Ingress Controller
- DNS configured for vault.maniak.lab

## Deployment Instructions

1. Create the Vault namespace:
   ```
   kubectl create namespace vault
   ```

2. Create a secret with your Vault Enterprise license:
   ```
   kubectl apply -f license-secret.yaml
   ```

3. Deploy Vault using Helm:
   ```
   helm install vault hashicorp/vault -f hashicorp-values.yaml --namespace vault
   ```

4. Initialize Vault (only needed on the first pod):
   ```
   kubectl exec -it vault-0 -n vault -- vault operator init
   ```
   This will generate unseal keys and a root token. Store these safely!

5. Unseal the first Vault pod using three of the generated unseal keys:
   ```
   kubectl exec -it vault-0 -n vault -- vault operator unseal <key-1>
   kubectl exec -it vault-0 -n vault -- vault operator unseal <key-2>
   kubectl exec -it vault-0 -n vault -- vault operator unseal <key-3>
   ```

6. Unseal the other Vault pods:
   ```
   kubectl exec -it vault-1 -n vault -- vault operator unseal <key-1>
   kubectl exec -it vault-1 -n vault -- vault operator unseal <key-2>
   kubectl exec -it vault-1 -n vault -- vault operator unseal <key-3>

   kubectl exec -it vault-2 -n vault -- vault operator unseal <key-1>
   kubectl exec -it vault-2 -n vault -- vault operator unseal <key-2>
   kubectl exec -it vault-2 -n vault -- vault operator unseal <key-3>
   ```

7. Access the Vault UI at http://vault.maniak.lab and log in with the root token.

## High Availability Setup

The Vault cluster is configured with:
- 3 Vault nodes in a highly available configuration
- Integrated Raft storage for consensus
- Automatic join for new nodes
- Leader election for fault tolerance

In this setup, if one Vault instance goes down, the cluster will continue to function as long as a quorum (2 of 3 nodes) is available. The Raft protocol ensures that data is replicated across all nodes.

## Security Notes

- The current configuration uses HTTP (not HTTPS) for accessing the Vault UI
- For production environments, consider enabling TLS
- The unseal keys and root token are stored in vault-credentials.txt (NOT checked into git)

## Maintenance

If Vault is restarted or pods are recreated, you'll need to unseal each pod again using the same process as steps 5-6.

To update the Vault configuration:
1. Edit the hashicorp-values.yaml file 
2. Apply the changes with:
   ```
   helm upgrade vault hashicorp/vault -f hashicorp-values.yaml --namespace vault
   ```