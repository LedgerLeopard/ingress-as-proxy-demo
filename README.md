# Demo of using Kubernetes ingress as proxy server for CI

## Prerequisites

1. Ready Kubernetes cluster (AKS for Azure)
1. Configured DNS zone
1. `kubectl` installed locally
1. `kubeconfig` for cluster obtained locally (for AKS can be received by `az aks get-credentials -n <name of cluster> -g <resource group>`)
1. Service Principal with rights to manage resources

> **NOTE:** We will use wildcard certificate because in such case we don't need to obtain new certificate for every endpoint. Limitation: all endpoints should be placed in one DNS zone and at one level. For example: `CI1.api.domain.com`, `CI2.api.domain.com`.

## Initial configuration (running once)

1. Install [helm locally](https://helm.sh/docs/intro/install/)
1. Cert-manager setup (for obtaining and updating TLS sertificates):
    1. Add cert-manager repo to helm: `helm repo add jetstack https://charts.jetstack.io`
    1. Update helm repos: `helm repo update`
    1. Install cert-manager to Kubernetes: `helm install --create-namespace -n cert-manager --set global.leaderElection.namespace="cert-manager",installCRDs="true" cert-manager jetstack/cert-manager`
    1. Wait till pods will be up and running: `kubectl get pods -n cert-manager`
    1. Prepare ClusterIssuer config by setting parameters at `ClusterIssuer.yaml` and `cert-manager-secret.yaml`:
        1. `__client-id__` - ClientID of service principal wich has access to and can manage DNS zone
        1. `__client-secret__` - secret for ClientID
        1. `__subscription__` - Azure Subscription ID
        1. `__tenant__` - Azure Tenand ID
        1. `__rg-name__` - Name of resource groupd where DNS zone is placed
        1. `__zone-name__` - DNS zone name
        1. `__email__` - email address of person who will receive notification of certificate expiration
    1. Apply configs:
        1. `kubectl apply -f cert-manager-secret.yaml`
        1. `kubectl apply -f ClusterIssuer.yaml`
    1. Prepare Certificate config by setting parameters at `Certificate.yaml`:
        1. `__zone-name__` - DNS zone name (same as for ClusterIssuer), result string should contain `*.` and quotes, for example: `"*.api.domain.com"`
    1. Apply config: `kubectl apply -f Certificate.yaml`
    1. Wait till certificate will be obtained by command: `kubectl describe certificate -n cert-manager tls-secret`: you should see `Message: Certificate is up to date and has not expired`
1. Ingress Controller setup: 
    1. Add ingress repo to helm: `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx`
    1. Update helm repos: `helm repo update`
    1. Install Ingress Controller to Kubernetes: `helm install -n nginx-ingress --create-namespace --set controller.wildcardTLS.secret="cert-manager/tls-secret" nginx-ingress ingress-nginx/ingress-nginx`
    1. Wait till pods will be up and running: `kubectl get pods -n nginx-ingress`
    1. Get IP address of Ingress Controller: `kubectl get svc -n nginx-ingress nginx-ingress-controller` (from `EXTERNAL-IP` column)
    1. Add DNS entry to DNS zone with **name**: `*`, **type**: `A` and **value**: obtained IP address - so all request to this DNS zone wwill be redirected to cluster

## Per andpoint configuration (running for every endpoint)

> **NOTE:** This part better to add to app as part of CI creating process. But at this demo manual variant is produced.

1. Create copy of `Endpoint-template.yaml` as `Endpoint.yaml`: `cp ./Endpoint-template.yaml ./Endpoint.yaml`
1. Add proper configs to `Endpoint.yaml`:
    1. `__name__` - name of CI/company/etc
    1. `__port__` - port of app at CI
    1. `__IP__` - IP address of CI
    1. `__namespace__` - namespace for placing endpoint, can be same for every endpoint or separate for each one
1. Apply configuration: `kubectl apply -f Endpoint.yaml`
1. Create copy of `Ingress-template.yaml` as `Ingress.yaml`: `cp ./Ingress-template.yaml ./Ingress.yaml`
1. Add proper configs to `Ingress.yaml`:
    1. `__name__` - name of CI/company/etc
    1. `__port__` - port of app at CI
    1. `__namespace__` - namespace for placing ingress, same as for endpoint
    1. `__zone-name__` - DNS zone name (same as for ClusterIssuer)
1. Apply configuration: `kubectl apply -f Ingress.yaml`
