# Hashicorp Vault integration to Openshift using Vault Secret Broker (VSO) Demo
Hahiscorp Vault Demo for Vault Secret Operator (VSO) integration with Openshift (OCP) or Kubernetes.

## Demo Description
This demo will show how to use the Vault Secret Operator to manage secrets in Kubernetes.
Summary for the Demo :
- Create 2 Vault KV Secrets for 2 different Apps
- Create 2 Vault KV AppRoles for each apps authentications
- Deploy 2 Apps on OCP on different Namespaces.
- Application is using the same Image, what it does is display the Pod name (where it has been deployed) and the Secret's value (which it has been defined).
- Install Vault Secret Operator (VSO) on OCP to manage secrets via OperatorHub
- Create 2 CRD Sets of VSO for each deployed apps.
- The Vault Static Secret will inject the Secrets from Vault KV into the OCP Secrets.


Demo Walkthrough : [demo-walkthrough.md](demo-walkthrough.md)