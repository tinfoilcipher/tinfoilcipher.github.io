---
title: "Understanding Certificates - The Job That Everyone Hates"
layout: post
date: 2020-03-01
categories: 
  - "opinions"
  - "security"
tags: 
  - "certificates"
  - "opinions"
  - "pki"
  - "secops"
  - "security"
---

I'm just going to throw it out there, I love working with security, cryptography and certificates. it wasn't always that way and like a lot of people I used to recoil in horror of the idea of having to work with certificates. In my experience that's not an uncommon scenario to be in, it's almost a universally loathed task to have to work with certs and it boils down to two things really:

1. **Horrible documentation** (this is an old problem, going back to the birth of X.509)
2. **Bad implementations** (Microsoft are the biggest culprits here, what a shocker)

## Documentation (Or Lack Of It)

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/wtf.jpg">
  <figcaption>X.509 and PKCS - It's no wonder people are confused</figcaption>
</figure>

In the old days (of 1988) the _X.509_ format was initially defined as part of _X.500_, and eventually stumbled it's way in to its own _RFC_ (v1) by 1993 as way to secure email as the _PEM_ format (Privacy Enhanced Mail), yes the format we're still using to secure websites was designed for email, but that was the tip of the iceberg really (the poor sysadmins of the early 90s). The proposals were flawed however, and it took until 1996 until a finished RFC was added to the IETF's working track and until 1999 until everyone could actually agree, [**RFC2459**](https://tools.ietf.org/html/rfc2459) finally put an end to this.

At the same time however, parallel to this, RSA were attempting to define _PKCS_ (Public Key Cryptography Standards), these were defined as a number of RFCs for PKCS1-15 which should have helped to simplify matters but created a whole bunch of other problems, as the legend goes a large portion of the documentation was lost and remains lost to this day (**PKCS13** and **14** still remain undocumented around 30 years later).

The intention of _PKCS_ was to define how public key cryptography standards which RSA had patented should be implemented, however as these weren't industry standards it just caused more problems, these did eventually start to creep in to the standards track.

## X.509 - What Exactly is it?

X.509 defines the standard for **SSL_ and _TLS** certificates. For the purposes of understanding how a certificate works SSL and TLS can be used **fairly** interchangeability (though they aren't the same thing by a far cry, it's not uncommon to hear them used in the same breath). It's really easier to think of **SSL** (Secure Sockets Layer) as the old technology and **TLS** (Transport Layer Security) as the modern protocol designed to **replace SSL**. If you're still using SSL in this day and age you should stop now.

When we talk about a certificate we're really talking about a **Public Key Certificate**. The certificate is effectively a digital identity document that proves that you own a corresponding **Public Key** which has been signed by the **Private Key** of the Certificate Authority that issued the certificate in the first place. The certificate solves this problem by carrying a bunch of other identifying information that the key cannot (website names, email addresses etc.) and linking the public key to the certificate.

Since certificates have to conform to the standard defined by X.509, it's safe to stick with the same structure over all systems (except those that use Java, but that's a whole other mess). The data carried in an X.509 certificate is basically defined as:

```yaml
Certificate:
  "Version Number": "Version" 3 #--X.509 Standard Version
  "Serial Number": "03:0F:F1:DA:68:94:87:3A:6A:9E:80:1C:9C:75:06:9F:B9:7B" #--Unique to the Certificate
  "Signature Algorithm ID": "PKCS #1 SHA-256 With RSA Encryption" #--As defined by PKCS#1
  "Issuer Name": #--Distinguished name of the CA issuing the Certificate
    "CN = Let's Encrypt Authority X3"
    "O = Let's Encrypt"
    "C = US"
  Validity period: 
    "Not Before": "29/02/2020, 12:31:45 GMT" #--Starting valid date of the Certificate ISO standard
    "Not After": "29/05/2020, 13:31:45 BST" #--Expiry date of the Certificate ISO standard
  "Subject Name": "CN = tinfoilcipher.co.uk" #--Can contain a list of multiple FQDNs/IPs
  Subject Public Key Info: 
    "Public Key Algorithm": "PKCS #1 RSA Encryption" #--As defined by PKCS#1
    "Subject Public Key": #--Public key represented in base64
  Extensions:
    # --Optional extension data declared here such as constraints and specific uses of the Public Key
    # --I.E if the certificate can only be used to identify websites or for email etc.
  "Certificate Signature Algorithm": "PKCS #1 SHA-256 With RSA Encryption" #--As defined by PKCS#1
  "Certificate Signature": #--Hashed digest derived from public key
```

It's important to understand that in a chain of certificates that each certificate carries it's own data set.

## Encoding and Base64

So that's all good so far, but you've probably seen a file that starts something like:

**-----BEGIN CERTIFICATE-----**

How about?

**-----BEGIN RSA PRIVATE KEY-----**

Or... 

**-----BEGIN PRIVATE KEY-----**

How about one that contains multiple? Or combinations of both? Doesn't help that they're almost identical does it? Take a look at these:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/sosimiliar-1024x399.png">
  <figcaption>You can see how these would be easy to confuse....<br/>These are just dummy examples, don't try and hack me :)</figcaption>
</figure>

I'm sure that stuff is easy to find out? Well, not really, and from my experience it can lead to people sending the wrong files around attached to emails.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/certificates.jpg">
  <figcaption>Experienced sysadmins in certificate hell, on a daily basis</figcaption>
</figure>

Let's clear it up. What we're looking at here is a [base64](https://en.wikipedia.org/wiki/Base64) encoded format of a certificate and a private keys The private key is used to sign your certificate, you don't want that in the hands of a third party, otherwise they can just decrypt any traffic that your certificate encrypts. If you use [**openssl**](https://www.openssl.org/) or another similar tool you can quite easily decode the key and see what it contains, even if it's passphrase protected you can start _bruteforcing_ it without restriction.

As a rule, if you have the option to work with _base64_ encoding when working with certificates, use it, you'll just make your life easier and avoid incompatibles generated by shoddy implementations. Most of the time your certificate and key are actually already in this format even if you don't realise it, try opening any random certificate you have in a text editor and you'll see.

## Extensions and more extensions

PEM, DER, CER, CRT, PFX, PKCS12, P12, KEY, PKEY. I've never seen a system screw up so badly with file extensions for no good reason, this isn't aided by the fact by some system will **only** take certificates in certain formats.

- **pem, der, cer, crt** - These are **ALL** certificates. With rare exception, these are almost always identical, there are **some** strange exceptions that can crop up with **pem** where the encoding doesn't work, but for the most part, you can just change the file extension, as a rule, if you can view it in a text editor and see a **\-----BEGIN CERTIFICATE-----** header, you can just change the extension.
- **key, pkey** - These are private keys, there's a multitude of other formats these can crop up in (including **.der** and **.pem** in some horrible systems would you believe) again, if you see and appropriate header, just rename it as you need it.
- **p12, pkcs12** - This is a bundle of both **Certificate and Private Key**, usually protected with a passphrase as defined by **PKCS#12**.
- **pfx** - **Personal Information Exchange**. The Microsoft implementation of **PKCS#12**. You can just renamed a .p12 or .pkcs12 to .pfx and end up with the same thing. Microsoft complicating things even further.

Was there really any need for that?

## Certificate Signing Requests (CSR)

So how do you actually get a certificate? With a CSR (Certificate Signing Request). In a nutshell, a person or system submits a request to a CA (Certificate Authority) with a set of information it wants to include. This is specified under **PKCS#10** (yes, another one), and luckily, this too can waver wildly between implementations and even installations.

Usually, you'll generate the CSR on your own system and then provide it to the CA, either an external provider (a big favourite of mine being the free to use public CA [**LetsEncrypt**](https://letsencrypt.org/)) or your own internal CA. The data you usually need to provide is:

- **CN** - Common Name, The FQDN of the website you're trying to secure
- **O** - The Organisational (or business name)
- **OU** - The Organisational Unit (or department)
- **L** - Location (Town/City)
- **ST** - Street
- **C** - Country, ISO country code E.G. GB
- **MAIL** - email address for the administrator, this is key as it will often be used for renewals.

Whilst all of this sounds good, few CAs actually enforce any of this information and you can often just stuff it with nonsense, not that I endorse that kind of behaviour, but the internet is rife with dodgy CAs these days. As a rule if you try and provide dodgy information you're only going to make a rod for your own back later.

Once you have generated the CSR, the process looks mostly like:

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/CSR-Flow-1.png">
  <figcaption>Standard Certificate Request Process</figcaption>
</figure>

## Chains, CA certificates and Intermediaries

Usually when working at scale (in an enterprise setting or with a public certificate provider) it is very rare to see just a single CA and we typically see the concept of **Intermediate CAs** arising (within an enterprise setting this is often part of a wider _PKI_ (or Public Key Infrastructure), that is to say a wider infrastructure which handles the distribution and revocation of certificates), but it is worth understanding the role that _Intermediate CAs_ play.

The CA itself, even in a single-CA deployment, has it's own certificate. This is used to sign all certificates that it issues and thus provide proof of authenticity. To this end, if you are to be issued a certificate from a CA, you must also carry the certificate from the CA in the first place. On the internet this is handled by most major browsers automatically shipping with the CA certificates from the major certificate providers already baked-in (go and look at your browsers security settings and you'll see what's in there, it's worrying the things that you already trust without having made that decision).

If we look at a web server on my LAN for example, we can see that there is a certificate chain present, certificates are signed by the CA certificate **mc-strongbox-ca**. If this certificate was not already loaded on to the system in advance then the certificate would be considered invalid as the website's certificate is signed by the CA and the CA would be unknown to the browser:

![](/assets/{{ page.path | split: '/' | last | split: '.' | first }}/certchain-1024x422.png)

Chaining is a critical component when working with certificates at any scale and works on the principle of the **[_Chain of Trust_](https://en.wikipedia.org/wiki/Chain_of_trust)**, in effect matching values defined in the earlier X.509 structure to the certificate higher in the chain until a certificate known by your browser (or whatever system is reading the certificate) can be found.

<figure>
  <img src="/assets/{{ page.path | split: '/' | last | split: '.' | first }}/chainoftrust.png">
  <figcaption>Chain of Trust model<br/>Source: https://en.wikipedia.org/wiki/Chain_of_trust</figcaption>
</figure>

You could overcome this requirement by importing **every single certificate of every single website you want to trust** in to your browser, but as that is somewhat impractical, the reasons for using chains becomes apparent.

Hopefully this clears up the logic behind certificates, for more practical work I would suggest taking a look at my [previous project of setting up your own CA]({% post_url 2019-11-15-bind-dns-and-openssl-certificate-authority %}), it's a pretty painless system and teaches you a lot about the actual functions along the way.
