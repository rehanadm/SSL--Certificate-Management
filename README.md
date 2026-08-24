# SSL Certificate Management Guide

## Overview

SSL/TLS certificates are used to secure communication between clients and servers. In enterprise environments, administrators frequently need to convert certificates between formats, extract certificate chains, install certificates on web servers and load balancers, and troubleshoot SSL-related issues.

This document covers:

* Understanding SSL certificate components
* Common certificate formats
* Converting certificates between formats
* Extracting certificates and private keys from PFX files
* Building complete certificate chains
* Verifying certificate chains
* Installing certificates on Linux servers
* Troubleshooting SSL issues

---

# SSL Certificate Components

A complete SSL certificate deployment typically consists of:

### Server Certificate

Issued for a specific domain.

Example:

```text
wildcard.globalworld.com.crt
```

### Private Key

Used for encryption and certificate validation.

Example:

```text
wildcard.globalworld.com.key
```

### Intermediate Certificate

Issued by the Certificate Authority (CA).

Example:

```text
DigiCertCA.crt
```

### Root Certificate

Top-level trusted CA certificate.

Example:

```text
DigiCertRoot.crt
```

---

# Common SSL Certificate Formats

| Format | Extension | Contains                          |
| ------ | --------- | --------------------------------- |
| PEM    | .pem      | Certificate, Key, Chain           |
| CRT    | .crt      | Certificate                       |
| CER    | .cer      | Certificate                       |
| KEY    | .key      | Private Key                       |
| CSR    | .csr      | Certificate Signing Request       |
| PFX    | .pfx      | Certificate + Private Key + Chain |
| P12    | .p12      | Same as PFX                       |
| DER    | .der      | Binary Certificate                |
| JKS    | .jks      | Java Keystore                     |

---

# PEM Format Structure

A PEM file may contain:

```text
-----BEGIN CERTIFICATE-----
MIIF....
-----END CERTIFICATE-----
```

or

```text
-----BEGIN PRIVATE KEY-----
MIIE...
-----END PRIVATE KEY-----
```

or both.

---

# Verify Certificate Information

Display certificate details:

```bash
openssl x509 -in certificate.pem -text -noout
```

Check expiration date:

```bash
openssl x509 -in certificate.pem -noout -enddate
```

Check issuer:

```bash
openssl x509 -in certificate.pem -noout -issuer
```

Check subject:

```bash
openssl x509 -in certificate.pem -noout -subject
```

---

# PFX to PEM Conversion

## View PFX Content

```bash
openssl pkcs12 -info -in certificate.pfx
```

You will be prompted for the PFX password.

---

## Extract Certificate Only

```bash
openssl pkcs12 -in certificate.pfx \
-clcerts -nokeys \
-out server.crt
```

---

## Extract Private Key

```bash
openssl pkcs12 -in certificate.pfx \
-nocerts -nodes \
-out server.key
```

---

## Extract CA Certificates

```bash
openssl pkcs12 -in certificate.pfx \
-cacerts -nokeys \
-out intermediate.crt
```

---

## Extract Everything into PEM

```bash
openssl pkcs12 -in certificate.pfx \
-out full.pem \
-nodes
```

---

# P12 to PEM Conversion

Convert P12 file:

```bash
openssl pkcs12 -in certificate.p12 \
-out certificate.pem \
-nodes
```

---

# CRT to PEM Conversion

Most CRT files are already PEM encoded.

Verify:

```bash
cat certificate.crt
```

If it begins with:

```text
-----BEGIN CERTIFICATE-----
```

rename it:

```bash
mv certificate.crt certificate.pem
```

---

# DER to PEM Conversion

Convert binary certificate:

```bash
openssl x509 \
-in certificate.der \
-inform DER \
-out certificate.pem \
-outform PEM
```

---

# CER to PEM Conversion

```bash
openssl x509 \
-in certificate.cer \
-out certificate.pem \
-outform PEM
```

If CER is DER encoded:

```bash
openssl x509 \
-in certificate.cer \
-inform DER \
-out certificate.pem
```

---

# Extracting Certificates from a PEM File

Display certificate count:

```bash
grep "BEGIN CERTIFICATE" full.pem
```

---

## Extract Server Certificate

```bash
awk '
/BEGIN CERTIFICATE/ {i++}
i==1 {print}
' full.pem > server.crt
```

---

## Extract Intermediate Certificates

```bash
awk '
/BEGIN CERTIFICATE/ {i++}
i>1 {print}
' full.pem > intermediates.crt
```

---

# Building a Complete Certificate Chain

A complete chain must follow this order:

```text
Server Certificate
Intermediate Certificate 1
Intermediate Certificate 2
Root Certificate
```

---

## Example

### Server Certificate

```bash
cat server.crt
```

### Intermediate Certificate

```bash
cat intermediate.crt
```

### Root Certificate

```bash
cat root.crt
```

---

## Create Full Chain

```bash
cat server.crt \
intermediate.crt \
root.crt \
> fullchain.pem
```

---

# Create Combined PEM with Private Key

Many applications require:

```text
Private Key
Server Certificate
Intermediate Certificate
Root Certificate
```

Create:

```bash
cat server.key \
server.crt \
intermediate.crt \
root.crt \
> ssl_bundle.pem
```

---

# Verify Complete Chain

Verify certificate chain:

```bash
openssl verify \
-CAfile fullchain.pem \
server.crt
```

Expected output:

```text
server.crt: OK
```

---

# Verify SSL Connectivity

```bash
openssl s_client \
-connect wildcard.globalworld.com:443 \
-showcerts
```

---

# Verify Remote Certificate Expiry

```bash
echo | openssl s_client \
-connect wildcard.globalworld.com:443 \
-servername wildcard.globalworld.com 2>/dev/null \
| openssl x509 -noout -enddate
```

---

# Verify Certificate and Private Key Match

## Certificate Hash

```bash
openssl x509 -noout -modulus \
-in server.crt \
| openssl md5
```

## Private Key Hash

```bash
openssl rsa -noout -modulus \
-in server.key \
| openssl md5
```

Both values must match.

---

# Nginx SSL Configuration

```nginx
server {
    listen 443 ssl;

    ssl_certificate     /etc/ssl/fullchain.pem;
    ssl_certificate_key /etc/ssl/server.key;
}
```

Validate:

```bash
nginx -t
```

Reload:

```bash
systemctl reload nginx
```

---

# Apache SSL Configuration

```apache
SSLCertificateFile      /etc/httpd/ssl/server.crt
SSLCertificateKeyFile   /etc/httpd/ssl/server.key
SSLCertificateChainFile /etc/httpd/ssl/intermediate.crt
```

Restart:

```bash
systemctl restart httpd
```

---

# HAProxy SSL Configuration

Create combined PEM:

```bash
cat server.key \
server.crt \
intermediate.crt \
> haproxy.pem
```

Configure:

```haproxy
bind *:443 ssl crt /etc/haproxy/haproxy.pem
```

Restart:

```bash
systemctl restart haproxy
```

---

# Common SSL Troubleshooting

## Unable to Load Certificate

Verify:

```bash
openssl x509 -in certificate.pem -text -noout
```

---

## Key Mismatch

Compare moduli:

```bash
openssl x509 -noout -modulus -in server.crt | openssl md5
openssl rsa -noout -modulus -in server.key | openssl md5
```

---

## Incomplete Chain

Verify:

```bash
openssl s_client -connect hostname:443 -showcerts
```

Missing intermediates indicate an incomplete chain.

---

## Certificate Expired

Check:

```bash
openssl x509 -in server.crt -noout -enddate
```

---

# Best Practices

* Always keep private keys secure.
* Never email private keys.
* Use 2048-bit or 4096-bit RSA keys.
* Maintain certificate inventory.
* Automate expiry monitoring.
* Use strong TLS versions.
* Regularly renew certificates before expiration.
* Store backups of certificates and keys securely.
* Verify certificate chains before deployment.

---

# Quick Reference Commands

```bash
# View certificate
openssl x509 -in cert.pem -text -noout

# Check expiry
openssl x509 -in cert.pem -noout -enddate

# PFX to PEM
openssl pkcs12 -in cert.pfx -out cert.pem -nodes

# Extract key
openssl pkcs12 -in cert.pfx -nocerts -nodes -out key.pem

# Verify chain
openssl verify -CAfile fullchain.pem server.crt

# Test SSL
openssl s_client -connect hostname:443 -showcerts
```

