# Traefik HA on Kubernetes with Cert-Manager

Traefik v2 removed support for storing ACME/Let's Encrypt certificates in a KV store, citing bugs with the raft consensus algorithm ([#4851](https://github.com/containous/traefik/issues/4851), [#3487](https://github.com/containous/traefik/issues/3487), [#5047](https://github.com/containous/traefik/issues/5047), [#3833](https://github.com/containous/traefik/issues/3833)). Automatic cert management feature moved to TraefikEE, leaving open-source users to either run a non-HA version or implement a custom solution to certificate management. 

Traefik documentation recommends using [cert-manager](https://github.com/jetstack/cert-manager) as the Certificate Controller and notes limited support for the Ingress Route CRD:

```
When using the Traefik Kubernetes CRD Provider, unfortunately Cert-Manager cannot interface directly with the CRDs yet, but this is being worked on by our team. A workaround is to enable the Kubernetes Ingress provider to allow Cert-Manager to create ingress objects to complete the challenges. Please note that this still requires manual intervention to create the certificates through Cert-Manager, but once created, Cert-Manager will keep the certificate renewed.
```

This repo walks through setting up Traefik v2 in HA mode on Kubernetes, using Cert-Manager and Cloudflare to manage the certificates. If you don't need HA and just need a quick Traefik-managed version of Let's Encrypt, you can follow [Quickstart with Traefik v2 on Kubernetes](https://medium.com/dev-genius/quickstart-with-traefik-v2-on-kubernetes-e6dff0d65216) instead.

## Prerequisites
- Kubernetes Cluster (e.g. GKE)
- Helm v3 
- DNS provider (e.g. Cloudflare)

## Install Traefik
We will deploy Traefik to `traefik` namespace:
```
$ kubectl create namespace traefik
```

Now let's deploy Traefik with 3 replicas. You can see the values in `traefik/traefik-values.yaml`.

```
$ helm repo add traefik https://containous.github.io/traefik-helm-chart

$ helm install -n traefik traefik traefik/traefik -f traefik/traefik-values.yaml
```

Wait for the deployments to come up and make note of the Load Balancer IP. 

## Install Cert-Manager
Cert-Manager is an open-source tool to automate the issuance and renewal of TLS certificates:

![cert-manager high level overview diagram](https://cert-manager.io/images/high-level-overview.svg)

We will install it in the namespace `cert-manager`:

```
$ kubectl create namespace cert-manager
```

Add the Jetstack Helm repo and install CRDs:

```
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.0 \
  --set installCRDs=true
```

Wait for all the cert-manager pods to come up:

```
$ kubectl get pods -n cert-manager -w 
```

## Deploy an Application 
For the sake of the demo, we will deploy the `whoami` app in the `default` namespace (see under `whoami` directory for deployment, service, and ingress files). You can replace this with your application or well-known Helm chart (e.g. Grafana, Kibana, etc).

Replace `whoami.example.com` with your FQDN and deploy:

```
$ kubectl apply -f whoami
```

## Create Certificates
In order to issue new certificates, we need to first define an Issuer. In this example, I'll be using Cloudflare for ACME Issuer type, using Let's Encrypt's staging server. You can also find other [supported configurations](https://cert-manager.io/docs/configuration/) (SelfSigned, CA, Vault, Venafi, and External Issuer Types) on the documentation.

Configure the `email` and `solvers` sections in `certs/issuer.yaml`. To use Cloudflare as [DNS01](https://cert-manager.io/docs/configuration/acme/dns01/) challenge solver, first create a new API token with the following settings:

* **Permissions**:
  * `Zone - DNS - Edit`
  * `Zone - Zone - Read`
* **Zone Resources**:
  * `Include - All Zones`

Mount the token as a Kubernetes secret:

```
$ kubectl create secret generic cloudflare-token --from-literal=dns-token=<my-api-token>
```

Finally, configure the certificate (modify the `commonName`, `secretName`, and `dnsNames` as needed in `certs/whoami-cert.yaml`) and deploy:

```
$ kubectl apply -f certs
```

## Set Up DNS
Check if the certificate has been generated:

```
$ kubectl describe certificate whoami-cert
```

You can also look at Traefik's debug logs to watch the cert become active. 

Finally, point the DNS record to the IP address of the Load Balancer to see a TLS enabled site backed by HA Traefik + cert-manager. Optionally, you can deploy the HTTPS redirect middleware for completeness. 
