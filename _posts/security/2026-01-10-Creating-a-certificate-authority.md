---
layout: post
title: Creating a certificate authority with openssl
categories: [Security, Networking]
introduction: "Use OpenSSL to create a certificate authority"
tags: [openssl, ca, authority, certificates, certificate authority, generate, tls. https]
---

Sometimes it is necessary to create an interal certificate to prevent insecure messages during connection (e.g. Websites). Therefore it is necessary to create a certificate authority to create self-signed certificates.

![Build Menu Publish Section]({{ '/assets/images/security/firefox_https_security_issue.png' | relative_url }})

Before the `CA` can be generated, it is preferable to have a predefined configuration (`openssl.cnf`).

```
[ v3_ca ]
subjectKeyIdentifier     = hash
authorityKeyIdentifier   = keyid:always,issuer:always
basicConstraints         = critical, CA:true, pathlen:0
keyUsage                 = critical, digitalSignature, cRLSign, keyCertSign
```

Then a key for the `CA` should be generated and in advance the `CA` itself.

``` bash
openssl genrsa -des3 -out rootCA.key 4096
openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 3650 -out rootCA.crt -config ./openssl.conf
```

To export the `rootCA` as `pfx` for `Windows` based systems:

``` bash
openssl pkcs12 -export -out rootCA.pfx -inkey rootCA.key -in rootCA.crt
```

Thats the thing. Now there is a Root-CA that can be used for self-signing certificates. How to generate certificates can be found [here]({{ 'security/networking/2026/01/11/Creating-self-signed-certificates.html'  | relative_url }}).

> The `rootCA.crt` can now be installed in the Windows/Linux Certificate Manager on every systeme the future self-signed certificates are getting used.
