---
title: "Kubernetes - Setting Up a PKI with cert-manager"
layout: post
date: 2024-07-02
categories: 
  - "containers"
  - "security"
tags: 
  - "cert-manager"
  - "certificates"
  - "containers"
  - "devops"
  - "integration"
  - "kubernetes"
  - "pki"
  - "secops"
  - "security"
---

I've talked a lot here about **[certificates](/understanding-certificates-the-job-everyone-hates/)** and [**how to set up a PKI**](/bind-dns-and-tinyca-certificate-authority/) in the past, it's a topic I enjoy a lot and seems to be generally loathed. I was pretty pleased to discover **[cert-manager](https://cert-manager.io/)**, which is a Kubernetes application designed to automate the creation and lifecycle management of TLS certificates within a Kubernetes environment. Despite being such a popular system, it still seems to create quite a bit of confusion, so in this short post I'm going to go through process of boostrapping a _Certificate Authority_ and issuing certificates for various purposes so you can start using _cert-manager_ as quick as possible.

The Kubernetes manifests used in this post can be found in GitHub [**here**](https://github.com/tinfoilcipher/blogexamples/tree/main/cert-manager).

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01.png)

## Installation

Installing is easiest using the Helm chart:

```bash
#--Install cert-manager. If for some reason you want to use a different
#--namespace or version, change as appropriate
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.2 \
  --set crds.enabled=true
  --set global.leaderElection.namespace=cert-manager

```

**cert-managers**'s first time boot process will take a little while to complete after Helm returns it's installation complete message. You can assume that this has completed correctly once all 3 of the main processes (**cert-manager**, **webhook** and **ca-injector** are all running):

```bash
kubectl get pod -n cert-manager
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-cainjector-775c8dd48-8ws6h   1/1     Running   0          20m
cert-manager-f4544758c-kn7wg              1/1     Running   0          19m
cert-manager-webhook-5488bc8d5f-smrvd     1/1     Running   0          20m
```

## Creating a Private Certificate Authority

With the application installed, we can set about bootstrapping a **Private CA (Certificate Authority**). Depending on your circumstances it might be more suitable for your **cert-manager** CA to be signed by an external _CA_, but I want to look at how we can bootstrap a CA completely inside Kubernetes and act as our root.

**cert-manager** uses a resource called an **Issuer** which is a type of CA that automatically responds to CSRs (Certificate Signing Requests). **Issuers** are at the heart of what lets **cert-manager** work so well, they will take our requests for certificates for whatever our services might be and spit out certificates on the other side, they will also handle the renewal of certificates for services running inside our cluster. Issuers are however just like any other X509 CA and as such if we're going to run a private one we will need to bootstrap a CA with a self-signed certificate.

The below manifest shows our bootstrap configuration, for the sake of brevity this is just a single tier CA:

```yaml
#--bootstrap-pki.yaml

#--Create a self-signed, cluster-wide Certificate Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
---
#--Create a Root CA Certificate and Key Pair, with a long lifetime
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tinfoilcipher-root-ca-cert
  namespace: cert-manager
spec:
  isCA: true
  duration: 87600h #--10 Years
  commonName: root-ca.tinfoilcipher.co.uk
  secretName: root-ca-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
    group: cert-manager.io
---
#--Create a cluster-wide Certificate Issuer, signed by the Root CA Certificate
apiVersion: cert-manager.io/v1
kind: ClusterIssuer #--"ClusterIssuers" can issue to all namespaces
metadata:           #--"Issuers" are namespace-scoped
  name: ca-issuer
  namespace: cert-manager
spec:
  ca:
    secretName: root-ca-secret

```

```bash
kubectl apply -f pki_bootstrap.yaml
# clusterissuer.cert-manager.io/selfsigned-issuer created
# certificate.cert-manager.io/tinfoilcipher-root-ca-cert created
# clusterissuer.cert-manager.io/ca-issuer created
```

Our CA is now bootstrapped and leaves us with an **Issuer** named **ca-issuer** which can issue certificates to any **Namespace** in our cluster.

## Issuing Certificates

With a private CA now ready for use, we can start issuing certificates, a couple of examples to consider are:

### Ingress (HTTPS)

By far the most commonly used function of **cert-manager** is to issue certificates to secure HTTPS ingresses, usually in conjunction with an **ingress-controller**. In the example below we are using the **nginx ingress-controller** which is a pretty popular combination.

```yaml
#--ingress.yaml

apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      cert-manager.io/cluster-issuer: ca-issuer #--Name of your issuer here
    name: someapp
    namespace: someapp-namespace
  spec:
    ingressClassName: nginx
    rules:
    - host: someapp.tinfoilcipher.co.uk #--Your desired HTTP hostname here
      http:
        paths:
        - backend:
            service:
              name: somebackend
              port:
                number: 8080
          path: /
          pathType: Prefix
    tls:
    - hosts:
      - someapp.tinfoilcipher.co.uk #--Must match above, will be on the certificate SAN list
      secretName: someapp-certificate #--Secret dynamically created based on the ingress 
                                      #--name, I.E. "name-certificate"

```

### Application

Useful in a scenario where you need to secure communication or do _Mutual TLS_ for a service that runs over TCP or UDP and you don't have the benefit of a _Service Mesh_.

```yaml
#--application.yaml

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: insecureapp
  namespace: insecurens
spec:
  isCA: false
  duration: 2160h #--90 days
  subject:
    organizations:
      - tinfoilcipher
    organizationalUnits:
      - Ops
    countries:
      - GB
  dnsNames:
    - insecureapp.tinfoilcipher.co.uk
  secretName: insecureapp-certificate
  usages:
    - digital signature #--Specifices key contstraints as a server certificate
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```

This will issue a certificate who's contents and corresponding key can be read from the **secret** **insecureapp-certificate**.

### Clients

It is also possible to issue **client** certificates, for authentication purposes. This can go hand in hand with the application certificates mentioned above for _mTLS_ but can also be useful if a client OUTSIDE the cluster needs to perform peer authentication (this is not often an ideal circumstance to be in as there is often no good way to rotate certificates to clients outside the cluster, but it is still very possible).

```yaml
#--client.yaml

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: someclient
  namespace: client-certificates
spec:
  isCA: false
  duration: 2160h #--90 days
  subject:
    organizations:
      - tinfoilcipher
    organizationalUnits:
      - Ops
    countries:
      - GB
  dnsNames:
    - someclient.tinfoilcipher.co.uk
  secretName: someclient
  usages:
    - client auth
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
    group: cert-manager.io

```

## Viewing Certificate and Key Data

All of the examples above will actually create several objects. Whilst it is transparent to the user when you create a **Certificate** resource you are actually:

- Submitting a **CSR** to the **Issuer**

- Generating a **Certificate** Resource, which waits for the **CSR** to be signed

- Once the **Certificate** is in a **Ready** state, a **Secret** is generate in the same **Namespace** as the **Certificate** which contains the **Certificate**, it's corresponding **Private Key** and the **CA Certificate** used to sign it, in base64 format.
    - The keys for each of these values is always **tls.crt**, **tls.key** and **ca.crt**

```bash
kubectl describe secret -n someapp-namespace someapp-certificate

# Name:         someapp-certificate
# Namespace:    someapp-namespace
# Labels:       <none>
# Annotations:  cert-manager.io/alt-names: someapp.madcaplaughs.co.uk
#               cert-manager.io/certificate-name: someapp-certificate
#               cert-manager.io/common-name:
#               cert-manager.io/ip-sans:
#               cert-manager.io/issuer-group: cert-manager.io
#               cert-manager.io/issuer-kind: ClusterIssuer
#               cert-manager.io/issuer-name: ca-issuer
#               cert-manager.io/uri-sans:
# 
# Type:  kubernetes.io/tls
# 
# Data
# ====
# ca.crt:   595 bytes
# tls.crt:  871 bytes
# tls.key:  1679 bytes
```

If we dig in to one of these keys, we can see that the base64 structured certificate can be viewed inside:

```bash
kubectl get secret -n someapp-namespace someapp-certificate -o json | jq -r '.data."tls.crt"' | base64 -d

# -----BEGIN CERTIFICATE-----
# MIICVzCCAf6fAwIBaaaQZkyBRvgkRxddhvn3TTxS4TAKBggqhkjOPQQDAjAlMdMw
# ...

kubectl get secret -n someapp-namespace someapp-certificate -o json | jq -r '.data."tls.key"' | base64 -d

# -----BEGIN RSA PRIVATE KEY-----
# 9PJiPjI6y+yLXaFwexFfNLsQtxt1+8PE+0i9VWi4vRdGGBk+3HDvNXLK6bXySuh3
# ...
```
