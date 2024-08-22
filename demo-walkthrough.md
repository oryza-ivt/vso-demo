### Prerequisites
- RedHat Openshift / Kubernetes Cluster already installed
- Hashicorp Vault already installed
- Image registry (e.g. DockerHub) that can be accessed by Openshift / Kubernetes Cluster.
- Required authentications manage cluster, create secret and manage policies.

# A. Vault Setup
On Vault Server or from a client that has access to Vault Server

## Create KV for Demo App's secret
1. Login to Vault
	```bash
	# Login using vault token (that has priviledge to enable KV Secret Engine and Create KV Secrets)
	vault login <VAULT_TOKEN>
	```
    - *Replace `<VAULT_TOKEN>` with the token that has priviledge to enable KV Secret Engine and Create KV Secrets.*

2. Enable KV Engine
	```bash
	## Enable KV Secret Engine on mount path `kvv2`
	vault secrets enable -path=kvv2 kv-v2
	```

3. Create KV Secrets

    Create KV Secrets for App Two   
	```bash
    vault kv put kvv2/app-two-secrets/app-config username="app_two_user" password="P@ssw0rd_app_two"
    ```

	Create KV Secrets for App Three
    ```bash
    vault kv put kvv2/app-three-secrets/app-config username="app_three_user" password="P@ssw0rd_app_three"
	```

## AppRole Setup for VSO auth
1. Enable AppRole
    ```bash
    ## Enable AppRole on default mount path `approle`
    vault auth enable approle
    ```

2. Create Policies

    Create policy for App Two, has read and list access to `kvv2/app-two-secrets`
    
    ```bash
    vault policy write app-two-policy - <<EOF
    path "kvv2/*" {
        capabilities = ["list"]
    }
    path "kvv2/data/app-two-secrets/*" {
        capabilities = ["read","list"]
    }
    EOF
    ```
    
    Create policy for App Three, has read and list access to `kvv2/app-three-secrets`
    ```bash
    vault policy write app-three-policy - <<EOF
    path "kvv2/*" {
        capabilities = ["list"]
    }
    path "kvv2/data/app-three-secrets/*" {
        capabilities = ["read","list"]
    }
    EOF
    ```

3. Create AppRoles

    Create AppRole for App Two
    ```bash
    # Create AppRole auth for App Two
    vault write auth/approle/role/app-two-auth \
        token_type=batch \
        secret_id_ttl=24h \
        token_ttl=24h \
        token_max_ttl=720h \
        policies=app-two-policy

    # Get the `role_id` for App Two
    vault read auth/approle/role/app-two-auth/role-id

    # Get the `secret_id` for App Two
    vault write -f auth/approle/role/app-two-auth/secret-id
    ```

    Create AppRole for App Three
    ```bash
    # Create AppRole auth for App Three
    vault write auth/approle/role/app-three-auth \
        token_type=batch \
        secret_id_ttl=24h \
        token_ttl=24h \
        token_max_ttl=720h \
        policies=app-three-policy

    # Get the `role_id` for App Three
    vault read auth/approle/role/app-three-auth/role-id

    # Get the `secret_id` for App Three
    vault write -f auth/approle/role/app-three-auth/secret-id
    ```

# B. [Optional] Test KV Secret retrieval using AppRoles
On Vault Server or from a client that has access to Vault Server
1. Testing App Two Secrets retrieval using CURL
    ```bash
    # Login using AppRole
    curl \
    --request POST \
    --data '{"role_id":"<APP_TWO_ROLE_ID>","secret_id":"<APP_TWO_SECRET_ID>"}' \
    http://<VAULT_ADDR>:8200/v1/auth/approle/login | jq

    # Read the secret
    curl -H "X-Vault-Token: <APPROLE_AUTH_TOKEN>" -H "X-Vault-Request: true" http://<VAULT_ADDR>:8200/v1/kvv2/data/app-two-secrets/app-config | jq
    ```
    - *Replace `<VAULT_ADDR>` with the Vault Server IP address.*
    - *Replace `<APP_TWO_ROLE_ID>` with the AppRole role_id for App Two.*
    - *Replace `<APP_TWO_SECRET_ID>` with the AppRole secret_id for App Two.*
    - *Replace `<APPROLE_AUTH_TOKEN>` with returned token from the AppRole authentication.*

2. Testing App Three Secrets retrieval using CURL
    ```bash
    # Login using AppRole
    curl \
    --request POST \
    --data '{"role_id":"<APP_THREE_ROLE_ID>","secret_id":"<APP_THREE_SECRET_ID>"}' \
    http://<VAULT_ADDR>:8200/v1/auth/approle/login | jq

    # Read the secret
    curl -H "X-Vault-Token: <APPROLE_AUTH_TOKEN>" -H "X-Vault-Request: true" http://<VAULT_ADDR>:8200/v1/kvv2/data/app-three-secrets/app-config | jq
    ```
    - *Replace `<VAULT_ADDR>` with the Vault Server IP address.*
    - *Replace `<APP_THREE_ROLE_ID>` with the AppRole role_id for App Three.*
    - *Replace `<APP_THREE_SECRET_ID>` with the AppRole secret_id for App Three.*
    - *Replace `<APPROLE_AUTH_TOKEN>` with returned token from the AppRole authentication.*


# C. Deploy Apps into Openshift
On OCP Server or from a client that has `kubectl` or `oc` access to OCP Server
## Create Namespaces
```bash
# Create Namespace for App Two
oc create namespace app-two-namespace

# Create Namespace for App Three
oc create namespace app-three-namespace
```

## Deploy Apps
Please review the [app-two-deployments.yaml](/deployments/app-two-deployments.yaml) and [app-three-deployments.yaml](/deployments/app-three-deployments.yaml) before executing this deployments command.
1. Deploy App Two
    ```bash
    oc apply -f deployments/app-two-deployments.yaml
    oc get all -n app-two-namespace
    ```
2. Deploy App Three
    ```bash
    oc apply -f deployments/app-three-deployments.yaml
    oc get all -n app-three-namespace
    ```
- *The Apps are using image from [oryzaivt/demo-vso-app:latest](https://hub.docker.com/r/oryzaivt/demo-vso-app) on **Docker Hub**. That Image is built from Github repo https://github.com/oryza-ivt/demo-vso-app. You could build it from the source code and publish it into your Private Registry, do not forget to change image Url on the deployment files if thats the case.*
- *The Apps are exposed using `NodePort` services for testing purpose, you might adjust it to using services exposed by ingress or other methods.*

## Test Apps
1. Open browser from any nodes that has access to OCP Server, then access the service URL.
2. Secrets value should be displayed.


# D. VSO Setup
## Install VSO via OperatorHub
1. Login to OpenShift Console
    <br/>Open your OpenShift web console and log in with appropriate credentials.

2. Navigate to OperatorHub
    <br/>On the left sidebar, go to `Operators` > `OperatorHub`.

3. Search for Vault Secrets Operator
    <br/>In the OperatorHub catalog, search for `Vault Secrets Operator`. It is typically provided by Red Hat or the Kubernetes community.

4. Select and Install the Operator
    <br/>Click on the `Vault Secrets Operator` tile.
    <br/>Review the information and click **Install**.

5. Configure Installation Options
    <br/>In the installation options:
    - Installation Mode: Choose either `"All namespaces on the cluster"` (Cluster-wide) or `"A specific namespace on the cluster"` (Namespace-scoped) depending on your requirements.
    - Update Channel: Select the appropriate update channel.
    - Approval Strategy: Choose either `"Automatic"` (automatically approves new updates) or `"Manual"` (requires manual approval).
    - Click **Install** to start the installation process.

6. Monitor the Installation
    <br/>You can monitor the progress by navigating to Installed Operators under the Operators section.
    <br/>Once the status shows as **Succeeded**, the operator is ready to use.

## Edit VSO CRD Deployment files
1. Open [app-two-crd.yaml](/deployments/app-two-deployments.yaml).
    - Replace `<APP_TWO_SECRET_ID>` with `secret_id` for App Two.
    - Replace `<APP_TWO_ROLE_ID>` with  `role_id` for App Two.
    - Replace `<VAULT_ADDR>` with the Vault Server IP address.

2. Open [app-three-crd.yaml](/deployments/app-three-deployments.yaml).
    - Replace `<APP_THREE_SECRET_ID>` with `secret_id` for App Three.
    - Replace `<APP_THREE_ROLE_ID>` with  `role_id` for App Three.
    - Replace `<VAULT_ADDR>` with the Vault Server IP address.

- For complete reference, please refer to : https://developer.hashicorp.com/vault/docs/platform/k8s/vso/sources/vault



## VSO Custom Resource Setup for Demo App
On OCP Server or from a client that has `kubectl` or `oc` access to OCP Server.

Please review the [app-two-crd.yaml](/deployments/app-two-deployments.yaml) and [app-three-crd.yaml](/deployments/app-three-deployments.yaml) before executing this deployments command.

1. Create CRD for App Two
    ```bash
    # Deploy VSO CRD for App Two : `VaultConnection`, `VaultAuth`, `VaultStaticSecret`
    oc apply -f deployments/app-two-crd.yaml

    # Check deployed CRD
    oc get VaultConnection -n app-two-namespace
    oc get VaultAuth -n app-two-namespace
    oc get VaultStaticSecret -n app-two-namespace
    ```

2. Create CRD for App Three
    ```bash
    # Deploy VSO CRD for App Three : `VaultConnection`, `VaultAuth`, `VaultStaticSecret`
    oc apply -f deployments/app-three-crd.yaml

    # Check deployed CRD
    oc get VaultConnection -n app-three-namespace
    oc get VaultAuth -n app-three-namespace
    oc get VaultStaticSecret -n app-three-namespace
    ```


# D. Test OCP-Vault Integration
## Refresh Application
- Refresh the browser to see the updated secrets.

## Test : Update Secret / Secret rotation
1. Update the secret in Vault, then wait several secoonds for VSO to detect the change.
2. Refresh apps on the browser to see the updated secrets.


<br/>

# References
- https://developer.hashicorp.com/vault/docs/platform/k8s/vso/openshift
- https://developer.hashicorp.com/vault/docs/platform/k8s/vso/sources/vault
- https://developer.hashicorp.com/vault/docs/platform/k8s/vso/api-reference
- https://hub.docker.com/r/oryzaivt/demo-vso-app
- https://github.com/oryza-ivt/demo-vso-app
