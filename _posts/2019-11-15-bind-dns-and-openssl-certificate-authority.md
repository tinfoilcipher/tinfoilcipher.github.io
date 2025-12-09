---
title: "BIND DNS and OpenSSL Certificate Authority"
layout: post
date: 2019-11-15
categories: 
  - "linux"
  - "projects"
tags: 
  - "bind"
  - "certificates"
  - "dns"
  - "encryption"
  - "linux"
  - "networking"
  - "pki"
  - "projects"
---

This project came from the back of my desire to learn more about public key certificates ahead of deploying a two tier PKI for an enterprise network, ahead of this I thought it would be prudent to try something a little smaller scale and see how the nuts and bolts worked and try and deploy a simple single tier PKI at home and see how it could be leveraged.

Cryptography is a big interest of mine and this was the first time I'd dipped my toe in trying to set anything up like this. Given the low power I had to play with I decided to build on a VM for the CA and an additional VM for the DNS server, each with a single vCPU and a 1GB of RAM each given the load they would have to handle.

As with most of my builds, this is Linux based and can be run on just about any hardware and can just as easily be done with a [**RaspberryPi**](https://www.raspberrypi.org/).

**Operating System**: Ubuntu 22.04  
**Required Applications for CA**: openssl, ca-certificates  
**Required Applications for DNS**: bind9, bind9utils, bind9-doc, nslookup, dig

## Install CA Pre-Requisite Software

```bash
sudo apt-get update
sudo apt-get install openssl ca-certificates #--Technically you don't need ca-certificates, but it's good to have
```

## Create the CA Directories and Files

First of all, we'll need to create a set of directories to house our CA and it's objects:

```bash
su root
mkdir -p /root/ca/certs #--Houses issued certificates
mkdir /root/ca/crl #--Houses the CRL (Certificate Revocation List)
mkdir /root/ca/newcerts #--Houses new certificates
mkdir /root/ca/private #--Houses private keys
mkdir /root/ca/requests #--Houses CSRs (Certificate Signing Requests)
mkdir /root/ca/index.txt #--Flat file index of issued certificates
echo ‘1000’ > /root/ca/serial #--Flat file serial number index of issued certificates. Must be created as root DON'T USE SUDO
chmod 600 /root/ca #--Set permissions on the ca directory
```

With the files in place, we can start to create our CA files.

## Configuring the CA

First we need to create a private key which will be used as the root of our CA's encryption, when prompted we should provide a strong and complex passphrase

```bash
openssl genrsa -aes256 -out /root/ca/private/ca-key.key 4096
# Generating Private Key, 4096 bit long modulus (2 primes)
# ........................++++
# ........................................++++
Enter pass phrase for /root/ca/private/ca-key.key: *****************************
Verifying - Enter pass phrase for /root/ca/private/ca-key.key: *****************************
```

With the key created, we can now use it to create the root certificate for our CA. It's important that we set a high expiry time here (in this case 10 years) as this certificate expiring will invalidate every certificate that the CA issues and that will be a problem, so let's make sure we kick that can down the road. Once we try to create the certificate we'll be prompted for a bunch of configuration data:

```bash
openssl req -new -x509 -key /root/ca/private/ca-key.key -out ca-cert.crt -days 3650
# Enter pass phrase for ca-key.key:
# You are about to be asked to enter information that will be incorporated
# into your certificate request.
# What you are about to enter is what is called a Distinguished Name or a DN.
# There are quite a few fields but you can leave some blank
# For some fields there will be a default value,
# If you enter '.', the field will be left blank.
# -----
Country Name (2 letter code) [AU]: GB
State or Province Name (full name) [Some-State]: North East
Locality Name (eg, city) []: Newcastle
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Tinfoilcipher
Organizational Unit Name (eg, section) []: ITNerds
Common Name (e.g. server FQDN or YOUR name) []: security.tinfoilcipher.co.uk
Email Address []: andy@tinfoilcipher.co.uk
```

The values here like state, OU etc are all arbitrary, but will form the basis of your certificates so at least enter sensible information. If you're doing this for a company this kind of thing can come back to bite you if you have any strict checking enabled later down the line.

Now we'll need to update the openssl configuration file to specify where our files are going to live:

`nano /usr/lib/ssl/openssl.cnf`

When the file is open, update the following values in the **\[ CA\_default \]** and **\[ policy\_match \]** sections (the purpose of these should be mostly self-explanatory):

```ini
[ CA_default ]
dir                 = /root/ca
certs               = $dir/certs
crl_dir             = $dir/crl
database            = $dir/index.txt
new_certs_dir       = $dir/newcerts
certificate         = $dir/ca-cert.crt
serial              = $dir/serial
crlnumber           = $dir/crlnumber
crl                 = $dir/crl.pem
private_key         = $dir/private/ca-key.key
unique_subject      = yes
x509_extensions     = usr_cert
x509_extensions     = v3_req
default_days        = 180
default_crl_days    = 90

#--The values set here ensure that how the CA will validate any
#--certificate requests. Requests which do not meet these criteria
#--will be rejected.
[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

With these values set, we now have a solid CA which can be used to issue certificates to sign webservers, devices, encrypted sessions and anything else that can leverage a certificate. In order to do this, we first need to create a private key and a corresponding CSR (Certificate Signing Request):

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout testserver.tinfoilcipher.key -out testserver.tinfoilcipher.csr
# Generating a RSA private key
# ..................................................++++++
# .............................................................................+++++++
# Writing a new private key to 'testserver.tinfoilcipher.key'
# # You are about to be asked to enter information that will be incorporated
# into your certificate request.
# What you are about to enter is what is called a Distinguished Name or a DN.
# There are quite a few fields but you can leave some blank
# For some fields there will be a default value,
# If you enter '.', the field will be left blank.
# -----
Country Name (2 letter code) [AU]: GB
State or Province Name (full name) [Some-State]: North East
Locality Name (eg, city) []: Newcastle
Organization Name (eg, company) [Internet Widgits Pty Ltd]: Tinfoilcipher
Organizational Unit Name (eg, section) []: ITNerds
Common Name (e.g. server FQDN or YOUR name) []: testserver.tinfoilcipher.co.uk
Email Address []: andy@tinfoilcipher
```

This will generate us a CSR, which our server can ingest, process and use to issue and sign a new certificate. We will be prompted for the passphrase for our CA's private key in order to sign the certificate:

```bash
openssl ca -in testserver.tinfoilcipher.csr -out testserver.tinfoilcipher.crt
# Using configuration from /usr/lib/openssl.cnf
Enter pass phrase for /root/ca/private/ca-key.key: *****************************
```

You will be shown a print out of the certificate information before confirming it's creation and addition to the database (I have omitted it here for the sake of brevity).

Our CA is only half of the battle though, certificates are a little bit useless for a lot of applications if we cannot reliably resolve hosts, for that we'll need to set up DNS.

## Install DNS Pre-Requisite Software

```bash
sudo apt-get update
sudo apt-get install bind9 bind9utils bind9-doc nslookup dig
```

## Configure bind to use IPv4 only

Unless you're a crazy person you probably aren't working in IPv6 yet, and if you are you aren't reading this. So let's set the server to use IPv4 only:

`sudo systemctl edit --full bind9`

Locate the line that reads

`[Service]   ExecStart=/usr/sbin/named -f -u bind`

and modify it to read

`[Service]   ExecStart=/usr/sbin/named -f -u bind -4`

Save with CTRL+O and reload and restart the DNS services;

```bash
sudo systemctl daemon-reload
sudo systemctl restart bind9
```

## Configure the DNS Server

First determine the IP address of your system by running **ip addr**. You will need this.

Then edit the bind9 options file:

`sudo nano /etc/bind/named.conf.options`

In the options block set the following values:

```bash
options {
         directory "/var/cache/bind";
         recursion yes;
         allow-recursion { any; };
         allow-query { any; };
         allow-query-cache { any; };
         listen-on { ; };
         allow-transfer { none; };
         forwarders {
              208.67.222.222;
              208.67.220.220;
         };
         //dnssec-validation auto;
         //auth-nxdomain no;
}
```

**listen-on** should be set to the IP address of your server  
**forwarders** are the external DNS servers that your server should go to for external lookups outside of your network, in this example we're looking at OpenDNS, however you can use anything, I personally use my firewall which then looks up to OpenDNS.  
The **any** options for recursion, query and cache options would usually be locked to access control lists which can be set elsewhere in the config, such a config should never be done in production however since this is for a small internal network it's less of a concern.

## Configure local options

Next edit the local options file:

`sudo nano /etc/bind/named.conf.local`

Update the file so it looks as below, replacing 172.16.0.0/24 with your network ID and tinfoilcipher.co.uk with your domain:

```bash
acl internals {
    127.0.0.1/8;
    172.16.0.0/24;
};

zone "tinfoilcipher.co.uk" {
    allow-query { internals; };
    type master;
    file "/etc/bind/zones/db.tinfoilcipher.co.uk";
};

zone "0.16.172.in-addr.arpa" {
    allow-query { internals; };
    type master;
    file "/etc/bind/zones/db.0.16.172";
};
```

Save the file with **CTRL+O**

## Configure the Forward lookup zone

To configure a zone file, we first need to create a zone file. As before, replace the domain name with your own domain and network IDs with your own.

```bash
cd /etc/bind/zones
touch db.tinfoilcipher.co.uk
```

First, add your SOA (Start of Authority record) by pasting in the below (replacing dns with the hostname of your server) NOTE THE TRAILING FULL STOP:

```ini
@       IN      SOA     dns.tinfoilcipher.co.uk. admin.tinfoilcipher.co.uk. (
                               3         ; Serial
                          604800         ; Refresh
                           86400         ; Retry
                         2419200         ; Expire
                          604800 )       ; Negative Cache TTL
```

Now to the same file, we add **nameserver records**:

```ini
; name servers - NS records
     IN      NS      dns.tinfoilcipher.co.uk.
```

Then we add **A records** for the nameservers:

```ini
; name servers - A records
dns.tinfoilcipher.co.uk.          IN      A       172.16.0.100
```

Then we can finally add **the remainder of records** to the zone:

```ini
;172.16.0.0/24 - A records
server1.tinfoilcipher.co.uk.      IN      A       172.16.0.10
server2.tinfoilcipher.co.uk.      IN      A       172.16.0.20

;172.16.0.0/24 - CNAME records
ca                                IN      CNAME   server1.tinfoilcipher.co.uk.
radius                            IN      CNAME   server1.tinfoilcipher.co.uk.
vpn                               IN      CNAME   server2.tinfoilcipher.co.uk.
```

Finally, save the file with **CTRL+O**

## Configure the reverse lookup zone

**CONFIGURE THE REVERSE LOOKUP ZONE**  
The reverse lookup zone allows for reverse DNS lookup, this is essential for proper DNS configuration:

```bash
cd /etc/bind/zones
touch db.0.16.172
sudo nano /etc/bind/zones/db.0.16.172
```

The process here is much the same but for reverse entries:

First create the **SOA** record:

```ini
$TTL    604800
 @       IN      SOA     dns.tinfoilcipher.co.uk. admin.tinfoilcipher.co.uk. (
                               4         ; Serial
                          604800         ; Refresh
                           86400         ; Retry
                         2419200         ; Expire
                          604800 )       ; Negative Cache TTL
```

**BE SURE TO INCREMENT THE SERIAL NUMBER BY 1**

Add the **nameserver** record:

```ini
; name servers - NS records
       IN      NS      dns.tinfoilciper.co.uk.
```

Add the **PTR** records, one is required for each A record and the record should be named with only the subnet and host bits of the record in question:

```ini
; PTR Records
0.10   IN      PTR     server1.tinfoilcipher.co.uk.    ; 172.16.0.10
0.20   IN      PTR     server2.tinfoilcipher.co.uk.    ; 172.16.0.20
```

Finally save the file with **CTRL+O**

## Verify DNS Configuration and Complete

Run the below commands to verify the configuration files and zone files:

```bash
sudo named-checkconf
sudo named-checkzone tinfoilcipher.co.uk db.tinfoilcipher.co.uk
sudo named-checkzone 0.16.172.in-addr.arpa db.0.16.172
```

If any errors are returned, there is an error in your file syntax, usually a missing bracket, semicolon or curly brace.

Finally, restart the DNS service:

`sudo systemctl restart bind9`

You should now be able to point any clients at this DNS server and use it correctly. You can now also issue certificates to your services and webservers using CNAMEs that exist in your DNS server.
