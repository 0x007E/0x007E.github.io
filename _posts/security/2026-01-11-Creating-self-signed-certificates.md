---
layout: post
title: Creating a self-signed certificate from a previously generated CA with openssl
categories: [Security, Networking]
introduction: "Use OpenSSL to create a self-certificate form a given certificate authority using openssl"
tags: [openssl, ca, authority, self-signed, certificates, certificate authority, generate, tls, https, ftps]
---

For internal encryption usage it is reconommed to create self-signed certificates. Therefore it is necessary to create a global root certificate authority (see [Creating a certificate authority with openssl]({{ 'security/networking/2026/01/10/Creating-a-certificate-authority.html' | relative_url }})).

Before the self-signed certificates can be generated, it is preferable to adapt the (`openssl.cnf`) that was created before [here]({{ 'security/networking/2026/01/10/Creating-a-certificate-authority.html' | relative_url }}).

```
[ req ]
default_bits        = 2048
default_md          = sha256
default_keyfile     = rootCA.key
distinguished_name  = req_distinguished_name
x509_extensions     = v3_ca
req_extensions      = v3_req
string_mask         = utf8only

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = AT
countryName_min                 = 2
countryName_max                 = 2

stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = ...

localityName                    = Locality Name (eg, city)
localityName_default            = ...

organizationName                = Organization Name (eg, company)
organizationName_default        = ...

organizationalUnitName          = Organizational Unit Name (eg, section)
organizationalUnitName_default  = Department of Network and Data Science

commonName                      = Common Name (eg, YOUR name)
commonName_default              = subdomain.domain.local
commonName_max                  = 64

emailAddress                    = Email Address
emailAddress_default            = my@mail.at
emailAddress_max                = 64

[ v3_ca ]
subjectKeyIdentifier     = hash
authorityKeyIdentifier   = keyid:always,issuer:always
basicConstraints         = critical, CA:true, pathlen:0
keyUsage                 = critical, digitalSignature, cRLSign, keyCertSign

[ v3_req ]
subjectKeyIdentifier     = hash
basicConstraints         = critical, CA:false
keyUsage                 = digitalSignature, nonRepudiation, keyEncipherment
extendedKeyUsage         = serverAuth
subjectAltName           = @alternate_names

[ alternate_names ]
DNS.1       = subdomain1.domain.local
DNS.2       = subdomain2.domain.local

IP.1       = 10.0.0.1
IP.2       = 192.168.0.0
```

Now it is quite easy to generate a self-signed certificate. Just generate a key with your personal required strength.

``` bash
openssl genrsa -out ./certs/aaa.domain.local.key 2048
```

Now a certificate signing request can be generated.

``` bash
openssl req -config ./openssl.conf -new -key ./certs/aaa.domain.local.key -out ./certs/aaa.domain.local.csr
```

Verifiying the request before signing is optional

``` bash
openssl req -text -noout -verify -in ./certs/aaa.domain.local.csr
```

After all sign the certificate with the `rootCA`. Mayge the validity in days should be adapted.

``` bash
openssl x509 -req -extfile ./openssl.conf -extensions v3_req -in ./certs/aaa.domain.local.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out ./certs/aaa.domain.local.crt -days 3650 -sha256
```

Everything done. Now it is possible to use the generated certificate inside webservers,FTP-servers and many more...