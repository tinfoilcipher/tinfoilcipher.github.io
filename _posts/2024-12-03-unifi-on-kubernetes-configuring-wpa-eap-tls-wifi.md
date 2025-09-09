---
title: "UniFi on Kubernetes - Configuring WPA-EAP-TLS WiFi"
layout: post
date: 2024-12-03
categories: 
  - "containers"
  - "devops"
  - "integrations"
  - "projects"
  - "security"
tags: 
  - "certificates"
  - "devops"
  - "integration"
  - "kubernetes"
  - "networking"
  - "pki"
  - "security"
  - "unifi"
---

A little while ago I migrated my **[UniFi Controller to Kubernetes]({% post_url 2023-08-10-unifi-on-kubernetes-deploying-the-controller %})**, part of that process involved migrating my **[WPA2 Enterprise WiFi]({% post_url 2019-11-15-wpa-eap-tls-wifi %})** network in to the cluster. It's quite an involved process and not one I've seen anyone try to do, so this post is going to look at how you can do that integration...as well as some of the reasons you might not want to do it in the real world! This is probably the most niche thing I've ever written and I struggle to imagine that anyone other than me will ever want to do such a thing...but hopefully someone else will find this useful one day :)

The configs for this post can be found in GitHub [**here**](https://github.com/tinfoilcipher/blogexamples/tree/main/wpa-tls-k8s).

<img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/01-1.png" class="scaled-img-75">

This project has 4 main requirements:

1. A Certificate Authority which can issue TLS certificates. I **[_have already covered setting up a PKI within Kubernetes using cert-manager_](/kubernetes-setting-up-a-pki-with-cert-manager)**, and that's what I'm going to be using in this example.

2. A RADIUS server which can function as part of a PKI with our Certificate Authority.

3. A **LoadBalancer** controller (I will also be exposing our RADIUS server via an external **LoadBalancer**, for reasons we'll come back to), we'll be using **MetalLB**, I have **[already covered the setup of MetalLB here]({% post_url 2023-03-17-building-a-bare-metal-kubernetes-lab-part-2 %})**. Any LoadBalancer is fine.

4. The UniFi Controller.

## Configuring freeradius

I'm going to be building on the deployment method suggested by [**freeradius-k8s by stanislaspiron**](https://github.com/stanislaspiron/freeradius-k8s). It is a wrapper around the official **freeradius** image that works well and means we don't have to go re-inventing the wheel too much.

First we'll need to configure our **freeradius** config files as below:

```bash
#--eap

eap {
    default_eap_type = tls
    timer_expire = 60
    ignore_unknown_eap_types = no
    cisco_accounting_username_bug = no
    max_sessions = ${max_requests}
    tls-config tls-common {
        private_key_file = /etc/ssl/private/tls.key
        certificate_file = /etc/ssl/certs/tls.crt
        ca_file = /etc/ssl/certs/ca.crt
        dh_file = ${certdir}/dh
        random_file = /dev/urandom
        ca_path = ${cadir}
        cipher_list = "DEFAULT"
        ecdh_curve = "prime256v1"
        cache {
            enable = no
        }
        ocsp {
            enable = no
        }
    }
    tls {
        tls = tls-common
    }
}

```

```bash
#--radiusd.conf

prefix = /opt
exec_prefix = ${prefix}
sysconfdir = ${prefix}/etc
localstatedir = ${prefix}/var
sbindir = ${exec_prefix}/sbin
logdir = ${localstatedir}/log/radius
raddbdir = ${sysconfdir}/raddb
radacctdir = ${logdir}/radacct
name = radiusd
confdir = ${raddbdir}
modconfdir = ${confdir}/mods-config
certdir = ${confdir}/certs
cadir   = ${confdir}/certs
run_dir = ${localstatedir}/run/${name}
db_dir = ${raddbdir}
libdir = ${exec_prefix}/lib
pidfile = ${run_dir}/${name}.pid
correct_escapes = true
max_request_time = 30
cleanup_delay = 5
max_requests = 16384
hostname_lookups = no
log {
        destination = stdout
        colourise = yes
        file = ${logdir}/radius.log
        syslog_facility = daemon
        stripped_names = no
        auth = yes
        auth_badpass = no
        auth_goodpass = no
        msg_denied = "You are already logged in - access denied"
}

checkrad = ${sbindir}/checkrad

ENV {
}

security {
        allow_core_dumps = no
        max_attributes = 200
        reject_delay = 1
        status_server = yes
        allow_vulnerable_openssl = no
}

proxy_requests  = yes
$INCLUDE proxy.conf
$INCLUDE clients.conf

thread pool {
        start_servers = 5
        max_servers = 32
        min_spare_servers = 3
        max_spare_servers = 10
        max_requests_per_server = 0
        auto_limit_acct = no
}

modules {
        $INCLUDE mods-enabled/
}

instantiate {
}

policy {
        $INCLUDE policy.d/
}

$INCLUDE sites-enabled/

```

```bash
#--authorize. UPDATE THE PASSWORDS TO SOMETHING OF YOUR CHOSING

rad_adm    Cleartext-Password := "supersecretpasswordforadmins"
	F5-LTM-User-Role = 0,
	F5-LTM-User-Info-1 = mgmt,
	F5-LTM-User-Partition = Common,
	F5-LTM-User-Shell = tmsh
rad_guest    Cleartext-Password := "supersecretpasswordforguests"
	F5-LTM-User-Role = 700,
	F5-LTM-User-Info-1 = mgmt,
	F5-LTM-User-Partition = Common,
	F5-LTM-User-Shell = tmsh
```

```bash
#--clients.conf

client localhost {
    ipaddr = 10.244.0.0/16 #--Set this to the IP range of your pod network
    secret = somelongandcomplicatedstringofrandomgibberish #--Update this to something of your choice
    require_message_authenticator = no
    nastype = other
}
```

Note: Pay particular attention to the **ipaddr** attribute in your **clients.conf**. As RADIUS requests are going to be coming from the UniFi controller and this could be running on any **Pod**, you will need to consider the entire address space that your **Pods** could be running within. As I am using an overlay network I have allowed the entire overlay subnet, this is far from ideal and presents a significant security issue if running in the real world.

## Deploying freeradius

With these configuration files set up, let's deploy **freeradius**:

```yaml
#--freeradius.yaml

kind: Deployment
apiVersion: apps/v1
metadata:
  name: freeradius
  namespace: freeradius
spec:
  replicas: 2
  selector:
    matchLabels:
      name: freeradius
  template:
    metadata:
      name: freeradius
      labels:
        name: freeradius
    spec:
      volumes:
      - name: raddb-files
        configMap:
          name: freeradius-files
          optional: true
      - name: raddb-secrets
        secret:
          defaultMode: 420
          secretName: freeradius-secrets
      - name: radius-server-certificate
        secret:
          defaultMode: 420
          secretName: freeradius-server-certificate
      containers:
        - name: freeradius
          image: 'freeradius/freeradius-server:latest-alpine'
          ports:
            - containerPort: 1812
              protocol: UDP
            - containerPort: 1813
              protocol: UDP
          volumeMounts:
            - name: raddb-secrets
              mountPath: /etc/raddb/mods-config/files/authorize
              subPath: authorize
              readOnly: true
            - name: raddb-secrets
              mountPath: /etc/raddb/clients.conf
              subPath: clients.conf
              readOnly: true
            - name: raddb-files
              mountPath: /etc/raddb/radiusd.conf
              subPath: radiusd.conf
              readOnly: true
            - mountPath: /etc/raddb/mods-enabled/eap
              name: raddb-files
              readOnly: true
              subPath: eap
            - mountPath: /etc/ssl/certs/ca.crt
              name: radius-server-certificate
              readOnly: true
              subPath: ca.crt
            - mountPath: /etc/ssl/certs/tls.crt
              name: radius-server-certificate
              readOnly: true
              subPath: tls.crt
            - mountPath: /etc/ssl/private/tls.key
              name: radius-server-certificate
              readOnly: true
              subPath: tls.key
---
kind: Service
apiVersion: v1
metadata:
  name: lb-freeradius
  namespace: radius
  annotations:
    metallb.universe.tf/allow-shared-ip: 'true'
spec:
  ports:
    - name: 'radius'
      protocol: UDP
      port: 1812
      targetPort: 1812
    - name: 'radius-acct'
      protocol: UDP
      port: 1813
      targetPort: 1813
  selector:
    name: freeradius
  type: LoadBalancer
  loadBalancerIP: 10.0.0.199 #--Set as appropriate for your loadbalancer
```

Note here that we are exposing the service as a **LoadBalancer** which means exposing it to services outside of the Kubernetes cluster. This is not exactly desirable as the only client we want to interact with our RADIUS is UniFi and this would be ideally done via the **Pod network** the same as our client requests, however, the UniFi console only allows us to configure a RADIUS connection via a single IP address. This rules out using a **ClusterIP** which would be preferable as we cannot make our requests via hostname, less than ideal and it will add latency to each of our requests.

```yaml
#--server-certificate.yaml

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: freeradius
  namespace: freeradius
spec:
  isCA: false
  subject:
    organizations:
      - tinfoilcipher
    organizationalUnits:
      - Ops
    countries:
      - GB
  dnsNames:
    - freeradius.freeradius.svc.cluster.local #--Include ClusterIP in SAN
    - freeradius.tinfoilcipher.co.uk          #--Update as suitable for your domain
  secretName: freeradius-server-certificate
  usages:
    - client auth
  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048
  issuerRef:
    name: ca-issuer         #--Update issuer details
    kind: ClusterIssuer     #--as suitable to 
    group: cert-manager.io  #--your cluster

```

```bash
kubectl create ns freeradius
# namespace/freeradius created

kubectl create configmap -n freeradius \
    freeradius-files --from-file eap --from-file radiusd.conf
# configmap/freeradius-files created

kubectl create secret generic -n freeradius \
    freeradius-secrets --from-file authorize --from-file clients.conf
# secret/freeradius-secrets created

kubectl apply -f server-certificate.yaml
# certificate.cert-manager.io/freeradius created

kubectl apply -f freeradius.yaml
# deployment.apps/freeradius created
# service/lb-freeradius created
```

and finally verify that our server is up and running:

```bash
kubectl get po -n freeradius
# NAME                          READY   STATUS    RESTARTS   AGE
# freeradius-5db9ab96b4-rbf12   1/1     Running   0          4m

kubectl get svc -n freeradius
# NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                         AGE
# freeradius   LoadBalancer   10.107.204.88   10.0.0.199      1812:31508/UDP,1813:30370/UDP   4m
```

## **Configure The UniFi Controller**

Within the UniFi console, configure a RADIUS profile under **Settings** > **Profiles** and configured as below, noting that:

- The **IP Address** should correspond to the **freeradius Servic_e External IP** as shown above

- The **Shared Secret** should correspond to the secret configured in **client.conf** earlier

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/03-1.png)

With the profile set up, ensure that it is attached to a SSID using **WPA Enterprise**:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/04-1.png)

## Issuing Client Certificates

With the _PKI_, _RADIUS_ and _WPA Enterprise_ _SSID_ now all in place, we can finally start connecting WiFi clients. To issue a new client certificate:

```yaml
#--client-certificate.yaml

apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: somelaptop
  namespace: client-certificates
spec:
  isCA: false
  duration: 26280h #--1 year. Really this is too long but practical, as
  subject:         #--we have no innate means of rotating certificates.
    organizations:
      - tinfoilcipher
    organizationalUnits:
      - Ops
    countries:
      - GB
  dnsNames:
    - somelaptop.tinfoilcipher.co.uk
  secretName: somelaptop
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

```bash
kubectl create namespace client-certificates
kubectl apply -f client-certificate.yaml
# certificate.cert-manager.io/somelaptop created

kubectl get secret -n client-certificates somelaptop-certificate -o json | jq -r '.data."tls.crt"' | base64 -d >> somelaptop.crt
kubectl get secret -n client-certificates somelaptop-certificate -o json | jq -r '.data."tls.key"' | base64 -d >> somelaptop.key
kubectl get secret -n client-certificates somelaptop-certificate -o json | jq -r '.data."ca.crt"' | base64 -d >> ca.crt
openssl pkcs12 -export -in somelaptop.crt -inkey somelaptop.key -name somelaptop.tinfoilcipher.co.uk -CAfile ca.crt -out somelaptop.p12
# Enter passphrase: ***************************
rm ca.crt somelaptop.key somelaptop.crt

```

This process will produce a PKCS12 file (or .p12 file) which can be used by most clients to connect to your newly created WPA Enterprise network. That was simple wasn't it!
