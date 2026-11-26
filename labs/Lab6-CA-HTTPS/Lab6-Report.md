# Lab 6 – Certificate Authority & HTTPS (Reverse Proxy)

## 1. Reverse Proxy Analysis

The `web` server does not run the application directly — it acts as a reverse proxy in front of a backend Java application.

```bash
# Identify webserver software
rpm -qa | grep httpd
ss -tlnp

# Locate the reverse proxy configuration
grep -n "ProxyPass" /etc/httpd/conf/httpd.conf

# Identify the backend service
ps aux | grep java
sudo find / -name "*.jar" 2>/dev/null
systemctl status <service-name>
```

**Findings:**
- Webserver software: **Apache httpd** (`mod_proxy`, `mod_proxy_http`).
- The reverse proxy is configured in `/etc/httpd/conf/httpd.conf` using:
  ```apache
  ProxyPass "/" "http://127.0.0.1:9200/"
  ProxyPassReverse "/" "http://127.0.0.1:9200/"
  ```
- The backend is a **Java** application (visible via `ps aux` as a running `java -jar ...` process), managed by its own `systemd` service and listening only on `localhost:9200`. It is not reachable directly from outside — all external traffic must go through Apache's reverse proxy on port 80/443.
- The service's `.jar` file and its `systemd` unit file (`/etc/systemd/system/<service>.service`) are the files needed to run it.

(img/Grep-Proxy.png) – ProxyPass/ProxyPassReverse configuration
(img/HTP.png) – backend process/port identification (`ps aux`, `ss -tlnp`)
(img/WorkingHTTP.png) – application reachable over plain HTTP through the reverse proxy, before any TLS is configured

## 2. Certificate Authority (CA) Setup

A private CA is created from scratch using OpenSSL (no `certbot`/public CA allowed).

```bash
mkdir ~/ca && cd ~/ca

# CA private key
openssl genrsa -out ca.key 4096

# Self-signed CA root certificate (10 years)
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=BE/ST=OostVlaanderen/L=Gent/O=CyberSecurityAdvancedLab/CN=CSA-Lab-RootCA" \
  -out ca.crt

# View the certificate
openssl x509 -in ca.crt -noout -text
```

**Answers:**
- The CA **does** use its own private key (`ca.key`) and its own certificate (`ca.crt`, self-signed).
- Every file's role is clear once you know OpenSSL's naming convention: `.key` = private key, `.csr` = signing request (unsigned), `.crt` = signed certificate.
- A certificate (CA or server) is inspected with `openssl x509 -in <file> -noout -text`.

(img/SS-CA.png) – CA key/certificate generation and verification output

## 3. Server Certificate (CSR + SAN)

The web server also needs its own private key and a certificate, signed by the CA — reusing the CA's key directly on the server would defeat the purpose of a CA.

SAN config file (`web-san.cnf`), required because modern browsers ignore the legacy `CN` field and only trust `subjectAltName`:

```ini
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C = BE
ST = OostVlaanderen
L = Gent
O = CyberSecurityAdvancedLab
CN = www.cybersec.internal

[v3_req]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = www.cybersec.internal
DNS.2 = web.cybersec.internal
DNS.3 = web
DNS.4 = www
IP.1 = 172.30.20.10
```

```bash
# Server private key + CSR
openssl genrsa -out web.key 2048
openssl req -new -key web.key -out web.csr -config web-san.cnf

# Sign the CSR with our CA
openssl x509 -req -in web.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out web.crt -days 825 -sha256 \
  -extfile web-san.cnf -extensions v3_req

# Verify
openssl x509 -in web.crt -noout -text | grep -A1 "Subject Alternative Name"
```

**Answers:**
- The webserver **does** use its own private key (`web.key`) and its own certificate (`web.crt`), separate from the CA's.
- A **CSR (Certificate Signing Request)** is an unsigned request containing the public key and identity info (CN, SAN, org...) that is sent to a CA to be signed into a trusted certificate.
- A **wildcard certificate** (e.g. `*.cybersec.internal`) is valid for every subdomain of a domain. It is convenient, but less safe: a single private key covers all subdomains, so if one service is compromised, the attacker can impersonate *every* subdomain with the same key — it also makes certificate-transparency-based reconnaissance easier for an attacker.

(img/SanCert-CaOndertekend.png) – CSR generation, CA signing and SAN verification

## 4. Apache HTTPS Configuration (mod_ssl)

```bash
sudo dnf install -y mod_ssl

# Install cert/key with correct SELinux context
sudo mkdir -p /etc/pki/tls/certs/csa /etc/pki/tls/private/csa
sudo mv web.crt /etc/pki/tls/certs/csa/
sudo mv web.key /etc/pki/tls/private/csa/
sudo restorecon -Rv /etc/pki/tls/certs/csa/ /etc/pki/tls/private/csa/
```

`/etc/httpd/conf.d/ssl.conf`:

```apache
SSLCertificateFile    /etc/pki/tls/certs/csa/web.crt
SSLCertificateKeyFile /etc/pki/tls/private/csa/web.key
```

```bash
sudo apachectl configtest
sudo systemctl restart httpd
```

**Answer:** yes — the webserver needs `mod_ssl` installed and a `VirtualHost` on port 443 pointing to the certificate/key, in addition to the existing reverse-proxy config (which keeps working unchanged behind HTTPS).

## 5. Trusting the CA in the Browser

Because the CA is private/self-signed, browsers reject it by default until it is explicitly trusted.

**Before trust** (fresh browser profile, CA not yet imported):

(img/WebsiteRisk.png) – "Warning: Potential Security Risk Ahead"

**Importing the CA** (Firefox): `about:preferences#privacy` → *Certificates* → *View Certificates* → *Authorities* → *Import* → select `ca.crt` → check *Trust this CA to identify websites*.

**After trust:**

(img/SecureWebsite.png) – valid padlock, page loads over HTTPS

**Answers:**
- Only **one file** needs to be added to the browser/client: the CA's public certificate, `ca.crt` (never the CA's private key).
- Yes, the certificate can be retrieved again at any time afterwards: Firefox → Authorities list → *View* → *Export...* saves the `.crt` back out. Importing it does not "hide" or protect the file in any special way.

## 6. Link to the OpenVPN Lab

The same PKI building blocks reappear in the OpenVPN lab: a CA key/certificate (`ca.key`/`ca.crt`), a server key/certificate pair, and this time also individual client key/certificate pairs — all signed by the same CA. The difference is that OpenVPN uses this CA for **mutual TLS**: both the server and the client authenticate each other with a certificate, instead of only the server authenticating itself to the client as in this lab.

## 7. HTTPS with TLS 1.2 (static RSA key exchange)

`/etc/httpd/conf.d/ssl.conf`:

```apache
SSLProtocol -all +TLSv1.2
SSLCipherSuite kRSA:!aNULL:!eNULL
```

```bash
sudo systemctl restart httpd

# Verify negotiated protocol/cipher
openssl s_client -connect localhost:443 -servername www.cybersec.internal -tls1_2 </dev/null | grep -E "Protocol|Cipher"
```

Result: `TLSv1.2` with `AES256-GCM-SHA384` (no `ECDHE-` prefix → confirms static RSA key exchange, no forward secrecy).

**Wireshark decryption:** Edit → Preferences → Protocols → TLS → "RSA keys list" → add entry `172.30.20.10 : 443 : http : web.key`.

**Answers:**
- Wireshark needs the server's **RSA private key** (`web.key`) to decrypt TLS 1.2 traffic that uses static RSA key exchange.
- Yes — after loading the key, the `Application Data` packets are decrypted and shown as plain `HTTP` (visible GET requests / 200 OK responses).

(img/WiresharkCapture.png) – TLS 1.2 traffic decrypted via RSA private key

## 8. HTTPS with TLS 1.3 (separate keypair)

A **separate** key/certificate pair is generated (same CA, same SAN config, different filenames: `web-tls13.key`/`web-tls13.crt`) so each TLS version is demonstrated with its own dedicated material.

```apache
SSLCertificateFile    /etc/pki/tls/certs/csa/web-tls13.crt
SSLCertificateKeyFile /etc/pki/tls/private/csa/web-tls13.key
SSLProtocol -all +TLSv1.3
```

```bash
sudo systemctl restart httpd
openssl s_client -connect localhost:443 -servername www.cybersec.internal -tls1_3 </dev/null | grep -E "Protocol|Cipher"
```

Result: `TLSv1.3` with `TLS_AES_256_GCM_SHA384`.

Loading the **same RSA private key** (`web-tls13.key`, or even the old `web.key`) into Wireshark's RSA keys list and repeating the capture:

(img/Undecrypted-TLS1.3.png) – traffic stays as encrypted `Application Data`, decryption fails

**Answer (theory):** TLS 1.3 removed static RSA key exchange from the protocol entirely. Every TLS 1.3 handshake uses an ephemeral (Elliptic-Curve) Diffie-Hellman key exchange, which provides **Perfect Forward Secrecy**: the session keys are derived from a fresh, random key pair generated for that single session and are never transmitted on the wire. The server's long-term RSA private key is only used to *sign* the handshake for authentication, not to encrypt/derive the session keys — so having it gives Wireshark no way to reconstruct the session keys, regardless of key length or algorithm strength.

## 9. HTTPS with TLS 1.3 — Decryption via SSLKEYLOGFILE

Since the private key can no longer be used, the session keys must be captured directly from the client at the moment they are generated, via the `SSLKEYLOGFILE` mechanism (standardized by NSS/OpenSSL). This is done **client-side**, because it models an attacker who has access to the victim's own machine, not the server.

```bash
# Firefox-ESR on this distro silently ignores SSLKEYLOGFILE (hardened NSS build) -> use Chromium
pkill -9 firefox chromium
SSLKEYLOGFILE=$HOME/sslkeylog-browser.log chromium --new-window "https://www.cybersec.internal/cmd" &
```

Browse / refresh the page, then make a copy of the (still-growing) log file before loading it into Wireshark:

```bash
cp $HOME/sslkeylog-browser.log $HOME/sslkeylog-browser-capture1.log
```

Wireshark: Edit → Preferences → Protocols → TLS → "(Pre)-Master-Secret log filename" → `/home/vagrant/sslkeylog-browser-capture1.log`.

(img/Decrypted-TLS1.3.png) – TLS 1.3 traffic decrypted via client-side SSLKEYLOGFILE, HTTP `GET /cmd` / `200 OK` visible

This method works for **any** TLS version and key-exchange mechanism, because it logs the actual derived traffic secrets instead of relying on the server's private key.

## 10. Discussion – Private CA & Deep Packet Inspection

A corporate-owned device can be configured (as done in this lab) to trust a privately-issued root CA. A corporate proxy holding that CA's private key can then transparently intercept, decrypt, inspect and re-encrypt all HTTPS traffic from that device — including traffic to banking or payment sites — without a browser warning, since the forged certificate chains back to a CA the device already trusts.

This is a legitimate security control (malware/data-exfiltration inspection, DLP) on company-owned hardware with a clear usage policy, but it is also a full man-in-the-middle position: the organization can read payment details, personal messages, and any other "secure" traffic in plaintext. It is therefore standard practice (and often a legal requirement) for organizations to clearly disclose this interception to employees, and it is a good reason never to do personal banking or healthcare logins on a corporate device whose CA is unknown to you — it's worth periodically checking a padlock's certificate issuer, especially on a managed device, to see whether it is a public CA or an internal corporate one.