
# Traefik with Cert Manager + Cloudflare
This is simple tutorial that presents how to configure Traefik with Cert Manager and DNS challenge with Cloudflare. 

## Configuring Traefik through the official Helm Chart Repo

Add Traefik's chart repository to Helm:

```sh
helm repo add traefik https://helm.traefik.io/traefik
```

```sh
helm repo update
```

Install Traefik with custom values:

```sh
kubectl create namespace traefik
```

```sh
helm upgrade --install traefik -f traefik/values.yaml traefik/traefik -n traefik
```

## Configuring Cert Manager

1. Install Cert-Manager [1.5.3](https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml) 

```sh
kubectl apply -f cert-manager/
```

2. According to Cert-manager documentation, in order to use Cloudflare you have to create the appropriate API Token. In order to do that
you need to create create at User Profile -> API Tokens -> API Token. 

The token needs to have the following settings:
 - Permissions
   - Zone - DNS - Edit
   - Zone - Zone - Read
 - Zone Resources:
   - Include - All Zones or Include - Specific Zone and Select the domain from the drop down list. 

3. The API token should be places as the Kubernetes Secret. It can be created with the following command:

```sh
kubectl create secret generic cloudflare-api-token-secret --from-literal=api-token=<API_TOKEN> -n cert-manager --dry-run=client -o yaml > cloudflare-api-token-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: <API Token>
```

4.  Then you need to create the Cluster Issuer that can be consumed in multiple namespaces. From the other hand Issuer is a namespaced scope.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <email@domain.org> # fix-me
    # name of a secret that is used to store the ACME private account
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          email: <cloudflare-email-address> # fix-me
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
```

## Obtain a TLS certificate using created Cluster Issuer

Create the certificate request manifest:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: whoami-prod
  namespace: app
spec:
  commonName: whoami.ds36.net
  secretName: whoami-prod
  issuerRef:
    name: cloudflare-issuer
    kind: ClusterIssuer
  dnsNames:
    - "whoami.ds36.net"
    - "whoami-prod.ds36.net"  
 ```
## Deploying sample application

Just deploy the manifest using the command: 

```sh
kubectl apply -f whoami/
```