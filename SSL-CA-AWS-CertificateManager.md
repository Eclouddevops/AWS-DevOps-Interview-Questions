# SSL Certificates, Certificate Authorities & AWS Certificate Manager — Deep-Dive Interview Q&A

## Table of Contents
1. [SSL/TLS Fundamentals](#ssltls-fundamentals)
2. [Certificate Authorities (CA)](#certificate-authorities-ca)
3. [Certificate Types & Formats](#certificate-types--formats)
4. [TLS Handshake & Protocol Deep Dive](#tls-handshake--protocol-deep-dive)
5. [AWS Certificate Manager (ACM)](#aws-certificate-manager-acm)
6. [ACM Private CA](#acm-private-ca)
7. [Certificate Implementation Scenarios](#certificate-implementation-scenarios)
8. [Troubleshooting & Tricky Scenarios](#troubleshooting--tricky-scenarios)

---

## SSL/TLS Fundamentals



**Q1: What is the difference between SSL and TLS? Why do people still say "SSL"?**

**A:**

| Feature | SSL (Secure Sockets Layer) | TLS (Transport Layer Security) |
|---------|---------------------------|-------------------------------|
| Version History | SSL 1.0, 2.0, 3.0 (all deprecated) | TLS 1.0, 1.1 (deprecated), 1.2, 1.3 (current) |
| Status | Deprecated since 2015 | Active standard |
| Security | Known vulnerabilities (POODLE, BEAST) | Stronger cipher suites, forward secrecy |
| Handshake | More round trips, slower | Faster (1-RTT in TLS 1.2, 0-RTT in TLS 1.3) |
| Maintained by | Netscape (original) | IETF (Internet Engineering Task Force) |

**Why people still say "SSL":**
- Brand recognition — "SSL certificate" is the common term
- TLS is technically the successor (TLS 1.0 = SSL 3.1 internally)
- When someone says "SSL certificate," they mean a certificate used for TLS connections

**Key point**: All modern HTTPS uses TLS. If an interviewer says "SSL," they almost certainly mean TLS 1.2 or 1.3.

---


**Q2: Explain how SSL/TLS provides confidentiality, integrity, and authentication.**

**A:**

```
Three pillars of TLS:

1. CONFIDENTIALITY (Encryption)
   ├── Asymmetric encryption (RSA/ECDSA) — Key exchange phase
   └── Symmetric encryption (AES-256-GCM) — Data transfer phase
   Result: Eavesdroppers cannot read the traffic

2. INTEGRITY (Hashing)
   ├── HMAC (Hash-based Message Authentication Code)
   └── SHA-256/SHA-384 used in TLS 1.2+
   Result: Tampered data is detected and rejected

3. AUTHENTICATION (Certificates)
   ├── Server presents certificate signed by trusted CA
   ├── Client verifies certificate chain up to Root CA
   └── Optional: Client certificates (mTLS) for mutual auth
   Result: You're talking to who you think you're talking to
```

**Tricky**: Encryption WITHOUT authentication is vulnerable to Man-in-the-Middle (MITM). An attacker could present their own certificate, encrypt traffic to/from you, and decrypt everything in between. That's why CA trust is essential.

---


**Q3: What is the difference between symmetric and asymmetric encryption in the context of TLS?**

**A:**

| Feature | Symmetric | Asymmetric |
|---------|-----------|------------|
| Keys | Same key encrypts and decrypts | Public key encrypts, private key decrypts |
| Speed | Fast (AES: hardware-accelerated) | Slow (RSA: 1000x slower than AES) |
| Use in TLS | Bulk data encryption after handshake | Key exchange during handshake only |
| Examples | AES-128-GCM, AES-256-GCM, ChaCha20 | RSA-2048, ECDHE, X25519 |
| Key distribution | Problem: how to share the key securely? | Solved: public key can be shared openly |

**How they work together in TLS:**
```
Client                                    Server
  |                                          |
  |-- ClientHello (supported ciphers) ------>|
  |<-- ServerHello + Certificate (public key)|
  |                                          |
  |   [Asymmetric: Exchange pre-master secret using server's public key]
  |                                          |
  |   Both derive same symmetric session key |
  |                                          |
  |== Symmetric encryption (AES-GCM) ======>|  ← All application data
  |<= Symmetric encryption (AES-GCM) =======|  ← Fast, efficient
```

**Why not use asymmetric for everything?** Performance. RSA-2048 encryption is ~1000x slower than AES-256. TLS uses asymmetric only to securely agree on a symmetric key.

---


**Q4: What is Forward Secrecy (Perfect Forward Secrecy)? Why is it critical?**

**A:**

**Without Forward Secrecy (RSA key exchange):**
```
- Server uses same RSA private key for years
- Attacker records ALL encrypted traffic (passive collection)
- Later, attacker obtains private key (breach, legal order, quantum computing)
- Attacker decrypts ALL previously recorded sessions!
```

**With Forward Secrecy (ECDHE key exchange):**
```
- Each session generates EPHEMERAL (temporary) key pair
- Session key is derived from Diffie-Hellman exchange
- After session ends, ephemeral keys are destroyed
- Compromising the server's long-term key CANNOT decrypt past sessions

Key exchange flow (ECDHE):
Client generates: ephemeral private key (a), sends public (g^a mod p)
Server generates: ephemeral private key (b), sends public (g^b mod p)
Both compute:     shared secret = g^(ab) mod p
After session:    Delete a and b → past sessions remain secure
```

**Tricky**: TLS 1.3 REQUIRES forward secrecy. RSA key exchange is removed entirely. Only ephemeral Diffie-Hellman (DHE/ECDHE) is allowed. This is a major security improvement.

**Cipher suite example:**
- `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` — Has forward secrecy (ECDHE)
- `TLS_RSA_WITH_AES_256_GCM_SHA384` — No forward secrecy (static RSA)

---

## Certificate Authorities (CA)


**Q5: What is a Certificate Authority (CA)? Explain the chain of trust.**

**A:**

```
Certificate Chain of Trust:

Root CA (self-signed, stored in OS/browser trust store)
    │
    ├── Signs → Intermediate CA 1 (cross-certified)
    │               │
    │               └── Signs → Your Server Certificate (leaf/end-entity)
    │
    └── Signs → Intermediate CA 2
                    │
                    └── Signs → Another Server Certificate

Trust validation (client-side):
1. Server presents: [Leaf cert] + [Intermediate cert(s)]
2. Client checks: Is leaf cert signed by intermediate? ✓
3. Client checks: Is intermediate signed by a trusted root? ✓
4. Client checks: Is root in my trust store? ✓
5. Client checks: Are all certs within validity period? ✓
6. Client checks: Are any certs revoked (CRL/OCSP)? ✗
7. Result: TRUSTED ✓
```

**Why intermediate CAs exist:**
- Root CA private key is kept OFFLINE (air-gapped HSM, vault)
- If intermediate CA is compromised, only revoke that intermediate — root remains trusted
- Reduces blast radius of a key compromise

**Major public CAs:** DigiCert, Let's Encrypt, Sectigo (Comodo), GlobalSign, GoDaddy

**Tricky**: If you only send the leaf certificate without the intermediate, some clients will fail to validate (incomplete chain). Always send the full chain: leaf + intermediate(s). Never send the root — clients already have it.

---


**Q6: How does certificate revocation work? Compare CRL vs OCSP vs OCSP Stapling.**

**A:**

| Method | How It Works | Pros | Cons |
|--------|-------------|------|------|
| CRL (Certificate Revocation List) | CA publishes list of revoked cert serial numbers | Simple, well-supported | Large files, cached (stale), slow downloads |
| OCSP (Online Certificate Status Protocol) | Client queries CA's OCSP responder in real-time | Real-time status, small response | Privacy issue (CA knows which sites you visit), latency |
| OCSP Stapling | Server fetches OCSP response, "staples" it to TLS handshake | Fast, private, no extra client request | Server must refresh stapled response regularly |
| OCSP Must-Staple | Certificate extension requiring stapled response | Prevents downgrade attacks | Not widely deployed yet |

```
CRL flow:
Client → Downloads CRL from CA → Checks if cert serial number is listed → Large file!

OCSP flow:
Client → Sends cert serial to CA's OCSP responder → Gets "Good/Revoked/Unknown"
Problem: CA sees every site you visit (privacy concern)

OCSP Stapling flow:
Server → Periodically fetches OCSP response from CA → Caches it
Server → Includes (staples) signed OCSP response in TLS handshake
Client → Verifies stapled response (signed by CA) → No privacy leak!
```

**Tricky**: Most browsers use "soft-fail" for OCSP — if the OCSP responder is down, the certificate is assumed valid. This means an attacker who can block OCSP traffic can use a revoked certificate! CRLite (Firefox) and proprietary solutions (Chrome CRLsets) address this.

---


**Q7: What is Certificate Transparency (CT)? Why was it introduced?**

**A:**

**Problem it solves:**
- A rogue or compromised CA could issue a certificate for `google.com` without Google knowing
- The fake cert would pass standard validation (signed by trusted CA)
- Happened in real life: DigiNotar (2011), Symantec incidents

**How CT works:**
```
Certificate Issuance with CT:

1. CA issues certificate for example.com
2. CA submits cert to multiple public CT Logs (append-only, Merkle tree)
3. CT Log returns SCT (Signed Certificate Timestamp) — proof of submission
4. SCT is embedded in certificate (or delivered via TLS extension/OCSP)
5. Browser verifies SCT is present before trusting the cert
6. Domain owners can MONITOR CT logs for unauthorized certificates

CT Log structure:
├── Append-only (cannot delete entries)
├── Cryptographically verifiable (Merkle tree proofs)
├── Publicly auditable (anyone can verify)
└── Multiple independent logs (no single point of trust)
```

**Monitoring tools:**
- Google Certificate Transparency search (crt.sh)
- Facebook CT monitoring
- AWS Config rule for monitoring ACM certificates

**Tricky**: Since April 2018, Chrome requires all publicly-trusted certificates to have CT (SCTs). A certificate without CT will show a warning even if it's signed by a valid CA.

---

## Certificate Types & Formats


**Q8: Explain the different types of SSL certificates (DV, OV, EV). When would you use each?**

**A:**

| Type | Validation Level | What CA Verifies | Time to Issue | Cost | Use Case |
|------|-----------------|------------------|---------------|------|----------|
| DV (Domain Validation) | Low | Domain ownership only (DNS/HTTP challenge) | Minutes | Free–$50 | Blogs, internal apps, dev environments |
| OV (Organization Validation) | Medium | Domain + organization exists (business docs) | 1-3 days | $50–$200 | Business websites, SaaS platforms |
| EV (Extended Validation) | High | Domain + org + legal entity + physical address | 1-2 weeks | $200–$1000 | Banks, e-commerce, government |

**Additional types by scope:**

| Type | Coverage | Example |
|------|----------|---------|
| Single domain | One FQDN | `www.example.com` |
| Wildcard | All subdomains (one level) | `*.example.com` (covers `api.example.com` but NOT `sub.api.example.com`) |
| SAN/Multi-domain | Multiple specific domains | `example.com`, `example.org`, `api.example.com` |
| Multi-domain Wildcard | Multiple wildcards | `*.example.com`, `*.example.org` |

**Tricky**: Wildcard certs (`*.example.com`) do NOT cover the bare domain (`example.com`). You need the bare domain as a SAN entry. Also, wildcards only cover ONE level — `*.example.com` does NOT match `sub.api.example.com`.

**Tricky**: EV certificates no longer show the "green bar" with company name in modern browsers (Chrome removed it in 2019). The practical security difference between DV and EV is debated — phishing sites with DV certs are common.

---


**Q9: Explain certificate file formats. What are PEM, DER, PFX/PKCS12, and CSR?**

**A:**

| Format | Encoding | Extension | Contains | Used By |
|--------|----------|-----------|----------|---------|
| PEM | Base64 (ASCII) | .pem, .crt, .cer, .key | Cert, key, or chain | Linux/Apache/Nginx, AWS |
| DER | Binary | .der, .cer | Single cert (binary) | Java, Windows |
| PFX/PKCS#12 | Binary | .pfx, .p12 | Cert + private key + chain (bundled) | Windows/IIS, Java keystore import |
| PKCS#7 | Base64 or Binary | .p7b, .p7c | Cert chain (NO private key) | Windows, Java |
| CSR | Base64 | .csr | Certificate Signing Request | Sent to CA for signing |

**PEM format example:**
```
-----BEGIN CERTIFICATE-----
MIIFjTCCA3WgAwIBAgIRANOxciY0IzLc9AUoUSrsnGowDQYJKoZIhvcNAQEL
BQAwTzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5
... (Base64 encoded DER data)
-----END CERTIFICATE-----
```

**Common conversion commands:**
```bash
# PEM to DER
openssl x509 -in cert.pem -outform DER -out cert.der

# DER to PEM
openssl x509 -in cert.der -inform DER -outform PEM -out cert.pem

# PEM to PFX (bundle cert + key + chain)
openssl pkcs12 -export -out cert.pfx -inkey private.key -in cert.pem -certfile chain.pem

# PFX to PEM (extract)
openssl pkcs12 -in cert.pfx -out all.pem -nodes

# Generate CSR + private key
openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr

# View certificate details
openssl x509 -in cert.pem -text -noout

# Verify certificate chain
openssl verify -CAfile chain.pem cert.pem
```

---


**Q10: What is a CSR (Certificate Signing Request)? What information does it contain?**

**A:**

```
CSR Generation Flow:

1. Generate RSA/ECDSA key pair on YOUR server
2. Create CSR containing:
   ├── Public key (from your key pair)
   ├── Common Name (CN): www.example.com (deprecated for domain)
   ├── Subject Alternative Names (SAN): example.com, www.example.com
   ├── Organization (O): Example Corp
   ├── Organizational Unit (OU): Engineering
   ├── City/Locality (L): Seattle
   ├── State (ST): Washington
   ├── Country (C): US
   └── Signature (signed with YOUR private key — proves you own the key pair)
3. Send CSR to CA
4. CA validates identity + domain ownership
5. CA signs your public key + metadata → Returns certificate
6. You install certificate + your private key on server
```

**Generate CSR with SANs (modern way):**
```bash
# openssl.cnf
[req]
distinguished_name = req_dn
req_extensions = v3_req
prompt = no

[req_dn]
CN = example.com
O = Example Corp
C = US

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = example.com
DNS.2 = www.example.com
DNS.3 = api.example.com

# Generate
openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr -config openssl.cnf
```

**Tricky**: The private key NEVER leaves your server. The CSR only contains the public key. If a CA ever asks for your private key, it's a red flag.

**Tricky**: Modern browsers ignore the CN (Common Name) field and only check SANs. A certificate with CN=example.com but no SAN entries will show a warning in Chrome/Firefox.

---

## TLS Handshake & Protocol Deep Dive


**Q11: Walk through the complete TLS 1.2 handshake step by step.**

**A:**

```
TLS 1.2 Full Handshake (2-RTT):

Client                                           Server
  |                                                 |
  |--- ClientHello -------------------------------->|  RTT 1 start
  |    ├── TLS version (1.2)                        |
  |    ├── Client random (32 bytes)                 |
  |    ├── Session ID (empty or previous)           |
  |    ├── Cipher suites (ordered preference)       |
  |    ├── Compression methods                      |
  |    └── Extensions (SNI, ALPN, etc.)             |
  |                                                 |
  |<-- ServerHello ---------------------------------|  RTT 1 end
  |    ├── Selected TLS version                     |
  |    ├── Server random (32 bytes)                 |
  |    ├── Session ID                               |
  |    └── Selected cipher suite                    |
  |<-- Certificate (server's cert chain) -----------|
  |<-- ServerKeyExchange (ECDHE params) ------------|
  |<-- ServerHelloDone -----------------------------|
  |                                                 |
  |--- ClientKeyExchange (ECDHE public) ----------->|  RTT 2 start
  |--- ChangeCipherSpec (switching to encrypted) --->|
  |--- Finished (encrypted verify) ---------------->|
  |                                                 |
  |<-- ChangeCipherSpec ----------------------------|  RTT 2 end
  |<-- Finished (encrypted verify) -----------------|
  |                                                 |
  |=== Application Data (HTTP request) ============>|  Encrypted!
  |<== Application Data (HTTP response) ============|
```

**Key derivation:**
```
Pre-master secret (from ECDHE exchange)
    + Client random
    + Server random
    → Master secret (PRF/HKDF)
    → Session keys (client write key, server write key, client MAC, server MAC)
```

---


**Q12: How does TLS 1.3 differ from TLS 1.2? What security improvements were made?**

**A:**

| Feature | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| Handshake | 2-RTT | 1-RTT (0-RTT for resumption) |
| Key exchange | RSA or ECDHE | ECDHE only (forward secrecy mandatory) |
| Cipher suites | Many (some insecure) | Only 5 strong AEAD ciphers |
| Compression | Supported | Removed (prevents CRIME attack) |
| Renegotiation | Supported | Removed (prevents attacks) |
| Static RSA | Supported | Removed (no forward secrecy) |
| RC4, 3DES, SHA-1 | Available | Removed |
| Server Hello encrypted | No | Yes (after ServerHello) |
| 0-RTT resumption | No | Yes (with replay risk) |

**TLS 1.3 Handshake (1-RTT):**
```
Client                                    Server
  |                                          |
  |--- ClientHello + KeyShare ------------->|  RTT 1 (everything in one flight)
  |    ├── Supported groups (curves)        |
  |    └── Key share (ECDHE public value)   |
  |                                          |
  |<-- ServerHello + KeyShare --------------|
  |<-- {EncryptedExtensions} ---------------|  ← Encrypted from here!
  |<-- {Certificate} -----------------------|
  |<-- {CertificateVerify} -----------------|
  |<-- {Finished} --------------------------|
  |                                          |
  |--- {Finished} ------------------------->|  Handshake complete
  |=== Application Data ===================>|
```

**Security improvements in TLS 1.3:**
1. Forward secrecy is MANDATORY (no RSA key exchange)
2. Handshake is encrypted (hides certificate from passive observers)
3. Removed legacy algorithms (no more CBC, RC4, SHA-1, static RSA)
4. Simplified — fewer moving parts = fewer attack surfaces
5. Downgrade protection built into the protocol

**Tricky**: 0-RTT in TLS 1.3 is vulnerable to replay attacks. A MITM can capture and replay the 0-RTT data. Use 0-RTT only for idempotent requests (GET, not POST). AWS ALB supports TLS 1.3 but you should understand the replay risk.

---


**Q13: What is SNI (Server Name Indication)? Why is it important for shared hosting?**

**A:**

**Problem without SNI:**
```
One IP address → One SSL certificate

Why? TLS handshake happens BEFORE HTTP request.
The server doesn't know which hostname the client wants until
after encryption is established. So it can only serve one cert per IP.
```

**Solution with SNI:**
```
Client includes hostname in ClientHello (plaintext extension)
Server selects correct certificate based on requested hostname

One IP address → Multiple SSL certificates (virtual hosting)

ClientHello:
├── TLS version: 1.3
├── Cipher suites: [...]
├── SNI extension: "api.example.com"   ← Server uses this to pick cert
└── Key share: [...]
```

**AWS context:**
- ALB uses SNI to serve multiple HTTPS sites on one load balancer
- CloudFront uses SNI by default (free) vs dedicated IP ($600/month)
- ALB supports up to 25 certificates via SNI
- NLB supports SNI as well

**Tricky**: SNI sends the hostname in PLAINTEXT in the ClientHello. This leaks which site you're visiting (even though the data is encrypted). TLS 1.3 introduced Encrypted Client Hello (ECH) to fix this, but adoption is still limited.

**Tricky**: Very old clients (Windows XP, Android < 4.4, IE on XP) don't support SNI. If you must support them, you need a dedicated IP per certificate.

---


**Q14: What is mTLS (Mutual TLS)? When and how would you implement it?**

**A:**

**Standard TLS vs mTLS:**
```
Standard TLS (one-way):
Client verifies server's identity ← Server presents certificate
Server does NOT verify client     ← Client presents nothing

mTLS (two-way):
Client verifies server's identity ← Server presents certificate
Server verifies client's identity ← Client presents certificate
```

**mTLS use cases:**
- Service-to-service communication (zero-trust architecture)
- API authentication (more secure than API keys)
- IoT device authentication
- Kubernetes pod-to-pod communication (Istio service mesh)
- Internal microservices communication

**Implementation in AWS:**
```
Option 1: API Gateway + mTLS
- Upload truststore (CA cert) to S3
- Configure custom domain with mutual TLS
- Clients must present cert signed by your CA

Option 2: ALB (not natively supported)
- Terminate mTLS at the application level
- Or use NLB (TCP passthrough) + app handles mTLS

Option 3: AWS App Mesh / Istio
- Service mesh handles mTLS between services
- Automatic cert rotation via SPIFFE/SPIRE
```

**Tricky**: mTLS on API Gateway requires a custom domain name. You cannot use the default execute-api endpoint with mTLS. Also, the truststore in S3 must be a PEM bundle of trusted CA certificates.

---

## AWS Certificate Manager (ACM)


**Q15: What is AWS Certificate Manager (ACM)? What are its key features and limitations?**

**A:**

```
AWS Certificate Manager (ACM):
├── FREE public SSL/TLS certificates
├── Automatic renewal (no manual intervention)
├── Integrated with AWS services (ALB, CloudFront, API GW, etc.)
├── Managed certificate lifecycle
└── Domain validation (DV) certificates only (no OV/EV)
```

| Feature | ACM Public Certs | ACM Private CA | Imported Certs |
|---------|-----------------|----------------|----------------|
| Cost | Free | $400/month per CA + per-cert fee | Free (import) |
| Renewal | Automatic | Automatic | Manual (no auto-renewal!) |
| Validation | DNS or Email | None (you control CA) | N/A |
| Certificate type | DV only | Any type | Any type |
| Private key access | No (AWS manages) | Yes (export option) | You manage |
| Use on EC2 directly | No | Yes (export) | Yes |
| Wildcard | Yes | Yes | Yes |

**Key limitations:**
1. **Cannot export private key** for ACM public certs — usable only with integrated services
2. **Cannot use on EC2** directly (need ALB/NLB/CloudFront in front)
3. **DV only** — no OV or EV certificates
4. **Regional** — cert in us-east-1 cannot be used by ALB in eu-west-1
   - Exception: CloudFront requires cert in us-east-1 (global)
5. **No support** for: EC2, ECS task directly, on-premises servers
6. **Rate limits**: 2,500 ACM certs per account (soft limit)

---


**Q16: Explain the ACM certificate validation methods (DNS vs Email). Which should you use?**

**A:**

**DNS Validation (recommended):**
```
1. Request certificate for example.com
2. ACM provides CNAME record:
   _acme-challenge.example.com → _validation-token.acm-validations.aws
3. You add CNAME to DNS (Route 53 or external DNS)
4. ACM checks DNS record → Issues certificate
5. Record stays forever → Enables automatic renewal

Advantages:
├── Automatic renewal (as long as CNAME exists)
├── No human intervention needed
├── Works with Route 53 one-click validation
├── Can validate domains you don't receive email for
└── Infrastructure-as-Code friendly (Terraform/CloudFormation)
```

**Email Validation:**
```
1. Request certificate for example.com
2. ACM sends validation email to:
   ├── admin@example.com
   ├── administrator@example.com
   ├── postmaster@example.com
   ├── hostmaster@example.com
   ├── webmaster@example.com
   └── WHOIS contact email
3. Someone clicks approval link in email
4. Certificate issued

Disadvantages:
├── Requires human intervention for renewal
├── Email must be configured and monitored
├── Cannot automate
├── WHOIS privacy can hide contact email
└── Not IaC-friendly
```

**Terraform example (DNS validation with Route 53):**
```hcl
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["*.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id = aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

**Tricky**: DNS validation CNAME records don't need to be removed after issuance. Keep them — ACM uses them for automatic renewal. If you delete them, renewal fails ~60 days before expiry.

---


**Q17: How does ACM automatic renewal work? What can go wrong?**

**A:**

```
ACM Renewal Timeline:

Day 0: Certificate issued (valid for 13 months / 395 days)
  ...
Day 335: ACM begins automatic renewal attempt (60 days before expiry)
  │
  ├── DNS-validated cert: ACM checks CNAME record still exists
  │   ├── CNAME found → Auto-renewed ✓
  │   └── CNAME missing → Renewal FAILS ✗
  │
  ├── Email-validated cert: ACM sends renewal email
  │   ├── Someone approves → Renewed ✓
  │   └── No one approves → Renewal FAILS ✗
  │
  └── Imported cert: NO auto-renewal (you manage it)
      └── You must import new cert before expiry

Day 365-395: Certificate expires if not renewed!
```

**Common renewal failure causes:**
1. DNS validation CNAME was deleted
2. Domain transferred to different DNS provider
3. Route 53 hosted zone was deleted/recreated
4. Email validation — no one approved the email
5. Domain WHOIS contact changed
6. Imported cert — manual process forgotten

**Monitoring renewal:**
```json
// EventBridge rule for ACM expiration alerts
{
  "source": ["aws.acm"],
  "detail-type": ["ACM Certificate Approaching Expiration"],
  "detail": {
    "DaysToExpiry": [45, 30, 15, 7, 3, 1]
  }
}
```

**AWS Config rule:**
```
acm-certificate-expiration-check (configurable days threshold)
```

**Tricky**: ACM certificates have a maximum validity of 395 days (13 months) per CA/Browser Forum rules. ACM renews at 60 days before expiry. If renewal fails silently, you might not notice until your site goes down.

---


**Q18: Which AWS services integrate with ACM? How do you attach certificates?**

**A:**

| AWS Service | ACM Integration | Certificate Region Requirement |
|-------------|----------------|-------------------------------|
| Elastic Load Balancer (ALB/NLB) | Direct integration | Same region as LB |
| CloudFront | Direct integration | Must be us-east-1 |
| API Gateway (Custom domains) | Direct integration | Same region (Regional) or us-east-1 (Edge) |
| Elastic Beanstalk | Via ALB integration | Same region |
| CloudFormation | Resource reference | Same region |
| AWS Amplify | Direct integration | us-east-1 |
| App Runner | Direct integration | Same region |
| VPN (Client VPN) | Direct integration | Same region |

**Services that do NOT integrate with ACM:**
| Service | What to Use Instead |
|---------|-------------------|
| EC2 instances | Import cert manually, or use ALB in front |
| ECS (container directly) | Use ALB/NLB in front, or mount cert from Secrets Manager |
| On-premises servers | Export from ACM Private CA, or use third-party cert |
| S3 static website | Put CloudFront in front with ACM cert |
| Lightsail | Has its own free cert provisioning |

**ALB with ACM example (CloudFormation):**
```yaml
Resources:
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 443
      Protocol: HTTPS
      Certificates:
        - CertificateArn: !Ref ACMCertificate
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      SslPolicy: ELBSecurityPolicy-TLS13-1-2-2021-06

  # Additional certs via SNI
  AdditionalCert:
    Type: AWS::ElasticLoadBalancingV2::ListenerCertificate
    Properties:
      ListenerArn: !Ref ALBListener
      Certificates:
        - CertificateArn: !Ref SecondCert
```

**Tricky**: For CloudFront, the ACM certificate MUST be in `us-east-1` regardless of where your origin is. This is a common mistake — people create the cert in their application region and wonder why CloudFront can't see it.

---


**Q19: How do you import a third-party certificate into ACM?**

**A:**

**When to import (vs using ACM-issued cert):**
- Need OV or EV certificate (ACM only issues DV)
- Certificate for use on EC2 directly
- Organization policy requires specific CA
- Existing certificates from DigiCert, Sectigo, etc.

**What you need to import:**
```
1. Certificate body (PEM)         — Your server certificate
2. Certificate private key (PEM)  — Your private key (NEVER share this!)
3. Certificate chain (PEM)        — Intermediate CA cert(s)
```

**AWS CLI import:**
```bash
aws acm import-certificate \
  --certificate fileb://cert.pem \
  --private-key fileb://private-key.pem \
  --certificate-chain fileb://chain.pem \
  --region us-east-1
```

**Important considerations:**
```
Imported certificates:
├── NO automatic renewal (you must reimport before expiry)
├── Must reimport to update (same ARN preserved)
├── Private key stored encrypted in ACM
├── Same integration capability as ACM-issued certs
├── Supported key algorithms: RSA (1024-4096), ECDSA (P-256, P-384)
└── Must be valid (not expired) at time of import
```

**Reimport for renewal (preserves ARN and associations):**
```bash
aws acm import-certificate \
  --certificate-arn arn:aws:acm:us-east-1:123456:certificate/abc-123 \
  --certificate fileb://new-cert.pem \
  --private-key fileb://new-key.pem \
  --certificate-chain fileb://new-chain.pem
```

**Tricky**: When reimporting, the ARN stays the same and all existing associations (ALB, CloudFront) automatically use the new cert. No need to update listeners or distributions.

**Tricky**: ACM imported certs generate CloudWatch events for expiration (45, 30, 15, 7, 3, 1 days out). Set up alerts because there's no auto-renewal!

---


## ACM Private CA

**Q20: What is ACM Private CA? When would you use it instead of public ACM?**

**A:**

```
ACM Private CA:
├── You operate your own Certificate Authority within AWS
├── Issue certificates for INTERNAL use (not publicly trusted)
├── Full control over certificate content and policies
├── Can issue ANY type of cert (server, client, code signing, etc.)
├── Supports certificate export (unlike public ACM)
└── Cost: $400/month per CA + $0.75 per cert (tiered pricing)
```

**Public ACM vs Private CA comparison:**

| Feature | ACM Public | ACM Private CA |
|---------|-----------|----------------|
| Trust | Publicly trusted by all browsers | Only trusted by systems you configure |
| Cost | Free | $400/month + per-cert |
| Use case | Public-facing websites | Internal services, mTLS, IoT |
| Cert types | DV server certs only | Server, client, code signing, etc. |
| Export key | No | Yes |
| Custom extensions | No | Yes |
| Validity period | Max 13 months | Configurable (up to 10 years for private) |
| Hierarchy | Amazon's CA | Your CA hierarchy |

**Use cases for Private CA:**
1. **Internal microservices** — mTLS between services
2. **IoT devices** — Device identity certificates
3. **VPN authentication** — Client certificates for VPN access
4. **Code signing** — Sign internal packages/artifacts
5. **Kubernetes** — Custom certificates for ingress/service mesh
6. **Compliance** — Regulatory requirement for own CA
7. **Wi-Fi (802.1X)** — Certificate-based network authentication

**Private CA hierarchy example:**
```
Root CA (offline, long-lived: 10 years)
├── Issuing CA for Servers (3 years)
│   └── Internal server certs (1 year)
├── Issuing CA for Clients (3 years)
│   └── Client auth certs (90 days)
└── Issuing CA for IoT (5 years)
    └── Device identity certs (2 years)
```

---


**Q21: How do you set up ACM Private CA with a proper hierarchy?**

**A:**

**Best practice: Two-tier hierarchy:**
```
                    ┌──────────────┐
                    │   Root CA    │ ← Inactive after signing subordinates
                    │ (10yr valid) │    Used ONLY to sign subordinate CAs
                    └──────┬───────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
     ┌──────┴──────┐ ┌────┴────┐ ┌──────┴──────┐
     │ Server CA   │ │Client CA│ │  IoT CA     │
     │(3yr valid)  │ │(3yr)    │ │ (5yr valid) │
     └──────┬──────┘ └────┬────┘ └──────┬──────┘
            │              │              │
     End-entity certs  Client certs   Device certs
     (1 year)          (90 days)      (2 years)
```

**Terraform setup:**
```hcl
# Root CA
resource "aws_acmpca_certificate_authority" "root" {
  type = "ROOT"

  certificate_authority_configuration {
    key_algorithm     = "RSA_4096"
    signing_algorithm = "SHA512WITHRSA"

    subject {
      common_name  = "Example Corp Root CA"
      organization = "Example Corp"
      country      = "US"
    }
  }

  revocation_configuration {
    crl_configuration {
      enabled            = true
      expiration_in_days = 7
      s3_bucket_name     = aws_s3_bucket.crl.id
    }
  }
}

# Subordinate CA
resource "aws_acmpca_certificate_authority" "subordinate" {
  type = "SUBORDINATE"

  certificate_authority_configuration {
    key_algorithm     = "RSA_2048"
    signing_algorithm = "SHA256WITHRSA"

    subject {
      common_name  = "Example Corp Server CA"
      organization = "Example Corp"
      country      = "US"
    }
  }
}

# Issue certificate from subordinate CA
resource "aws_acm_certificate" "internal_service" {
  domain_name               = "api.internal.example.com"
  certificate_authority_arn = aws_acmpca_certificate_authority.subordinate.arn

  lifecycle {
    create_before_destroy = true
  }
}
```

**Tricky**: After creating a Root CA in ACM Private CA, you must INSTALL its self-signed certificate before it can issue anything. The Root CA starts in `PENDING_CERTIFICATE` state. Use `aws acm-pca get-certificate-authority-csr` then sign and install.

---


## Certificate Implementation Scenarios

**Q22: Design end-to-end HTTPS architecture for a multi-tier web application on AWS.**

**A:**

```
Architecture:

Internet Users
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│ CloudFront (ACM cert: *.example.com — us-east-1)               │
│ ├── TLS 1.3 termination at edge                                │
│ ├── Security policy: TLSv1.2_2021                              │
│ ├── HSTS header: Strict-Transport-Security: max-age=63072000   │
│ └── Origin protocol: HTTPS only                                │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTPS (TLS 1.2+)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ ALB (ACM cert: internal.example.com — same region)             │
│ ├── SSL policy: ELBSecurityPolicy-TLS13-1-2-2021-06            │
│ ├── HTTP → HTTPS redirect (port 80 → 443)                     │
│ └── Access logs enabled                                         │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP or HTTPS (internal)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ ECS/EC2 Application Tier                                        │
│ ├── If HTTPS: Self-signed or Private CA cert                   │
│ └── Security Group: only allow traffic from ALB SG             │
└──────────────────────────────┬──────────────────────────────────┘
                               │ TLS (require SSL)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│ RDS (enforce ssl: rds.force_ssl = 1)                           │
│ ├── Uses RDS CA certificate (rds-ca-rsa2048-g1)               │
│ ├── App connects with: sslmode=verify-full                     │
│ └── Download RDS CA bundle for client verification             │
└─────────────────────────────────────────────────────────────────┘
```

**TLS configuration at each layer:**
```
Layer          | Cert Source    | TLS Version | Notes
CloudFront     | ACM (public)   | TLS 1.2+   | Must be us-east-1
ALB            | ACM (public)   | TLS 1.2+   | Same region as ALB
App → RDS      | RDS CA (AWS)   | TLS 1.2+   | Force via parameter group
App → Redis    | In-transit enc | TLS 1.2+   | ElastiCache encryption
App → S3       | AWS managed    | TLS 1.2+   | HTTPS endpoint automatic
```

---


**Q23: How do you enforce HTTPS everywhere and redirect HTTP to HTTPS on AWS?**

**A:**

**Layer 1: CloudFront — Viewer Protocol Policy**
```json
{
  "ViewerProtocolPolicy": "redirect-to-https"
  // Options: "allow-all", "https-only", "redirect-to-https"
}
```

**Layer 2: ALB — HTTP to HTTPS redirect rule**
```yaml
# ALB Listener on port 80
Resources:
  HTTPListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref ALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: redirect
          RedirectConfig:
            Protocol: HTTPS
            Port: "443"
            StatusCode: HTTP_301  # Permanent redirect
```

**Layer 3: Application — HSTS Header**
```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```
- `max-age=63072000` — Browser remembers to use HTTPS for 2 years
- `includeSubDomains` — Applies to all subdomains
- `preload` — Submit to browser preload list (Chrome, Firefox, etc.)

**Layer 4: S3 bucket policy — Deny non-SSL requests**
```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"],
  "Condition": {
    "Bool": {"aws:SecureTransport": "false"}
  }
}
```

**Layer 5: RDS — Force SSL connections**
```sql
-- PostgreSQL parameter group
rds.force_ssl = 1

-- MySQL parameter group  
require_secure_transport = ON

-- Application connection string
postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=rds-ca.pem
```

**Tricky**: `aws:SecureTransport` in S3 bucket policies also blocks non-SSL CLI/SDK calls. Make sure all your applications and scripts use HTTPS endpoints for S3.

---


**Q24: How do you configure SSL/TLS security policies on ALB and CloudFront? What cipher suites should you use?**

**A:**

**ALB Security Policies (predefined by AWS):**

| Policy Name | Min TLS | Use Case |
|-------------|---------|----------|
| ELBSecurityPolicy-TLS13-1-2-2021-06 | TLS 1.2 | Recommended (TLS 1.3 + 1.2 fallback) |
| ELBSecurityPolicy-TLS13-1-3-2021-06 | TLS 1.3 | TLS 1.3 only (strictest) |
| ELBSecurityPolicy-TLS-1-2-2017-01 | TLS 1.2 | TLS 1.2 only (no 1.3) |
| ELBSecurityPolicy-2016-08 | TLS 1.0 | Legacy (avoid!) |
| ELBSecurityPolicy-FS-1-2-2019-08 | TLS 1.2 | Forward secrecy ciphers only |

**CloudFront Security Policies:**

| Policy | Min TLS | Notes |
|--------|---------|-------|
| TLSv1.2_2021 | TLS 1.2 | Recommended |
| TLSv1.2_2019 | TLS 1.2 | Broader cipher support |
| TLSv1.2_2018 | TLS 1.2 | Older, more compatible |
| TLSv1.1_2016 | TLS 1.1 | Legacy (avoid) |
| TLSv1_2016 | TLS 1.0 | Legacy (avoid!) |

**Recommended cipher suite order (TLS 1.2):**
```
1. TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384  (best: ECDSA + ECDHE + AEAD)
2. TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384    (RSA cert + ECDHE + AEAD)
3. TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
4. TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
5. TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305   (mobile-friendly)
```

**TLS 1.3 cipher suites (only 3 in practice):**
```
TLS_AES_256_GCM_SHA384
TLS_AES_128_GCM_SHA256
TLS_CHACHA20_POLY1305_SHA256
```

**Tricky**: You CANNOT customize cipher suites on ALB — you can only select a predefined security policy. If you need custom cipher control, use NLB (TCP passthrough) and terminate TLS at your application.

**Tricky**: CloudFront between viewer and edge uses the CloudFront security policy. But CloudFront to origin uses a DIFFERENT negotiation. Make sure your origin also supports strong TLS.

---


**Q25: How do you implement SSL/TLS for a multi-region application with Route 53 failover?**

**A:**

```
Multi-Region SSL Architecture:

                     ┌─────────────────────┐
                     │    Route 53          │
                     │ (latency/failover)   │
                     └──────┬──────┬────────┘
                            │      │
              ┌─────────────┘      └─────────────┐
              ▼                                   ▼
┌─────────────────────────┐       ┌─────────────────────────┐
│ us-east-1               │       │ eu-west-1               │
│ ├── ACM cert (regional) │       │ ├── ACM cert (regional) │
│ ├── ALB + ASG           │       │ ├── ALB + ASG           │
│ └── RDS (primary)       │       │ └── RDS (read replica)  │
└─────────────────────────┘       └─────────────────────────┘
```

**Key considerations:**
1. **ACM certs are regional** — You need a separate cert in EACH region
2. **Same domain on multiple certs** — ACM allows this (DNS validation works for both)
3. **Route 53 health checks** — Use HTTPS health checks (validates TLS too)
4. **Certificate management** — Both certs auto-renew independently

**Terraform for multi-region certs:**
```hcl
# Provider for each region
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# Cert in us-east-1
resource "aws_acm_certificate" "us" {
  provider          = aws.us_east_1
  domain_name       = "app.example.com"
  validation_method = "DNS"
}

# Cert in eu-west-1
resource "aws_acm_certificate" "eu" {
  provider          = aws.eu_west_1
  domain_name       = "app.example.com"
  validation_method = "DNS"
}

# SAME DNS validation record works for both!
# (ACM uses the same CNAME regardless of region)
```

**Tricky**: The DNS validation CNAME record is the SAME for the same domain across regions. You only need one CNAME record in Route 53 to validate the certificate in multiple regions.

---


**Q26: How do you handle SSL certificate rotation with zero downtime?**

**A:**

**ACM-managed certificates (automatic, zero-downtime):**
```
ACM auto-renewal process:
1. ACM issues new certificate ~60 days before expiry
2. New cert has same ARN as old cert
3. AWS atomically replaces cert on ALB/CloudFront/API GW
4. No configuration change needed
5. Zero downtime — seamless transition

Timeline:
├── T-60 days: ACM starts renewal attempt
├── T-45 days: If failed, ACM retries + sends EventBridge notification
├── T-30 days: More notifications
├── T-0: Certificate expires (if all attempts failed)
```

**Imported certificates (manual rotation):**
```bash
# Zero-downtime rotation for imported certs:

# 1. Generate new cert from CA (before old one expires)
# 2. Reimport to SAME ARN:
aws acm import-certificate \
  --certificate-arn arn:aws:acm:us-east-1:123456:certificate/existing-id \
  --certificate fileb://new-cert.pem \
  --private-key fileb://new-key.pem \
  --certificate-chain fileb://new-chain.pem

# ALB/CloudFront automatically uses new cert (same ARN)
# Zero config change, zero downtime
```

**On EC2/ECS (application-managed TLS):**
```bash
# Strategy 1: Blue-Green deployment
# Deploy new instances with new cert → shift traffic → terminate old

# Strategy 2: Graceful reload (Nginx)
# Replace cert files on disk → reload without dropping connections
cp new-cert.pem /etc/nginx/ssl/cert.pem
cp new-key.pem /etc/nginx/ssl/key.pem
nginx -s reload  # Graceful reload, no dropped connections

# Strategy 3: Use Secrets Manager with rotation Lambda
# Application watches for secret changes → reloads cert
```

**Tricky**: When you reimport a certificate to the same ARN, there's a brief moment (usually < 1 second) during propagation. In practice, this is imperceptible, but for ultra-sensitive applications, consider deploying a second cert ARN first and switching listeners.

---


## Troubleshooting & Tricky Scenarios

**Q27: Your website shows "NET::ERR_CERT_AUTHORITY_INVALID" — walk through the troubleshooting steps.**

**A:**

```
Common causes and diagnostic steps:

1. INCOMPLETE CERTIFICATE CHAIN
   ├── Server sends leaf cert but NOT intermediate CA
   ├── Browser can't build trust chain to root
   ├── Fix: Include intermediate cert(s) in server config
   └── Test: openssl s_client -connect host:443 -showcerts
             (should show 2-3 certs, not just 1)

2. SELF-SIGNED CERTIFICATE
   ├── Cert signed by untrusted CA (or self-signed)
   ├── Common in dev environments
   └── Fix: Use ACM or public CA cert for production

3. EXPIRED INTERMEDIATE/ROOT CA
   ├── CA cert expired even if leaf cert is valid
   ├── Example: Let's Encrypt DST Root CA X3 expiry (Sep 2021)
   └── Fix: Update certificate chain, use current intermediates

4. WRONG CERTIFICATE SERVED
   ├── Server has multiple certs, serving wrong one
   ├── SNI misconfiguration
   └── Fix: Check SNI config, verify correct cert per domain

5. CLOCK SKEW ON CLIENT
   ├── Client system time is wrong
   ├── Cert appears "not yet valid" or "expired"
   └── Fix: Sync system clock (NTP)
```

**Diagnostic commands:**
```bash
# Check full certificate chain
openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | \
  openssl x509 -text -noout

# Verify chain completeness
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>&1 | \
  grep -A2 "Certificate chain"

# Check certificate expiration
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Check which cert is returned for specific SNI
openssl s_client -connect example.com:443 -servername api.example.com

# Test with SSL Labs (comprehensive)
# https://www.ssllabs.com/ssltest/
```

---


**Q28: ACM certificate is stuck in "Pending validation" — how do you troubleshoot?**

**A:**

```
Troubleshooting ACM Pending Validation:

DNS Validation Issues:
├── CNAME record not created
│   └── Fix: Check ACM console for exact CNAME name/value, create it
├── CNAME created in wrong hosted zone
│   └── Fix: Must match the domain's authoritative DNS
├── CNAME propagation delay
│   └── Check: dig _validation.example.com CNAME (or nslookup)
├── CNAME has wrong value (typo)
│   └── Fix: Copy exact value from ACM (including trailing dot)
├── DNS provider doesn't support underscore in CNAME name
│   └── Fix: Use email validation or switch DNS provider
└── Wildcard + bare domain need SAME validation record
    └── Note: *.example.com and example.com use one CNAME

Email Validation Issues:
├── Email not received
│   ├── Check spam folder
│   ├── Verify WHOIS email is correct
│   └── Ensure admin@/postmaster@/etc. mailbox exists
├── Approval link expired (72 hours)
│   └── Fix: Resend validation email from ACM console
└── Domain privacy hiding WHOIS email
    └── Fix: Use DNS validation instead
```

**Debugging commands:**
```bash
# Check if DNS validation record exists
dig _acme-challenge.example.com CNAME +short

# Verify the authoritative nameservers
dig example.com NS +short

# Check if record is visible from AWS perspective
aws acm describe-certificate --certificate-arn arn:aws:acm:... \
  --query 'Certificate.DomainValidationOptions'

# Force DNS validation check (delete and recreate the record)
# Sometimes helps with caching issues
```

**Tricky**: ACM validation can take up to 72 hours in worst cases due to DNS propagation. If using an external DNS provider (not Route 53), propagation is often slower. Route 53 integration validates in minutes.

**Tricky**: For wildcard certs (`*.example.com`), ACM generates ONE validation record for the apex domain. If you also include `example.com` as a SAN, it uses the SAME validation record — you only need one CNAME.

---


**Q29: You need to migrate SSL certificates from on-premises to AWS. What's the process?**

**A:**

**Migration strategy:**
```
Assessment:
├── Inventory all existing certificates
│   ├── Where are they installed? (web servers, load balancers, APIs)
│   ├── What domains do they cover?
│   ├── When do they expire?
│   ├── What type? (DV, OV, EV, wildcard, SAN)
│   └── Who issued them? (DigiCert, Sectigo, Let's Encrypt, internal CA)
│
├── Decision matrix:
│   ├── Public-facing + DV → Use ACM free public cert (no migration needed)
│   ├── Public-facing + OV/EV → Import existing cert into ACM
│   ├── Internal services → ACM Private CA or import
│   └── EC2 direct TLS → Import to ACM + use ALB, OR install on EC2 directly
│
└── Migration approach per category:
    Option A: Replace with ACM cert (recommended for public)
    Option B: Import existing cert (for OV/EV or remaining validity)
    Option C: ACM Private CA (for internal PKI migration)
```

**Step-by-step for importing existing cert:**
```bash
# 1. Export cert from on-premises (example: from Apache/Nginx)
# Cert: /etc/ssl/certs/server.crt
# Key:  /etc/ssl/private/server.key  
# Chain: /etc/ssl/certs/chain.crt

# 2. Verify the certificate
openssl x509 -in server.crt -text -noout
openssl verify -CAfile chain.crt server.crt

# 3. Verify key matches certificate
openssl x509 -noout -modulus -in server.crt | md5sum
openssl rsa -noout -modulus -in server.key | md5sum
# Both MD5 hashes must match!

# 4. Import to ACM
aws acm import-certificate \
  --certificate fileb://server.crt \
  --private-key fileb://server.key \
  --certificate-chain fileb://chain.crt \
  --region us-east-1 \
  --tags Key=Environment,Value=Production Key=MigratedFrom,Value=OnPrem

# 5. Attach to ALB/CloudFront
# 6. Update DNS to point to AWS resources
# 7. Test thoroughly before decommissioning on-prem
```

**For internal PKI migration to ACM Private CA:**
```
On-premises Root CA → Cross-certify with ACM Private CA
OR
Create new ACM Private CA hierarchy → Gradually reissue internal certs
```

**Tricky**: When importing, ensure the private key is in PKCS#8 or traditional PEM format. If it's encrypted (has a passphrase), decrypt it first: `openssl rsa -in encrypted.key -out decrypted.key`

---


**Q30: Explain common SSL/TLS attacks and how to prevent them on AWS.**

**A:**

| Attack | How It Works | Prevention on AWS |
|--------|-------------|-------------------|
| MITM (Man-in-the-Middle) | Attacker intercepts traffic between client and server | Proper cert validation, HSTS, certificate pinning |
| SSL Stripping | Downgrade HTTPS to HTTP transparently | HSTS header, HTTP→HTTPS redirect, preload list |
| POODLE | Exploits SSL 3.0 CBC padding | Disable SSL 3.0 (use TLS 1.2+ security policy) |
| BEAST | Exploits TLS 1.0 CBC ciphers | Use TLS 1.2+, prefer GCM ciphers |
| Heartbleed | OpenSSL memory leak (CVE-2014-0160) | Patch OpenSSL, use managed services (ALB/CloudFront) |
| CRIME/BREACH | Exploits TLS compression | TLS 1.3 removes compression; ALB doesn't use it |
| Downgrade attack | Force use of weaker TLS version | Minimum TLS 1.2 security policy, TLS_FALLBACK_SCSV |
| Renegotiation attack | Exploit TLS renegotiation | TLS 1.3 removes renegotiation entirely |
| Certificate spoofing | Fake cert for target domain | Certificate Transparency, HSTS, CAA records |

**AWS-specific protections:**
```
1. ALB Security Policy: ELBSecurityPolicy-TLS13-1-2-2021-06
   └── Disables TLS 1.0, 1.1, weak ciphers

2. CloudFront Security Policy: TLSv1.2_2021
   └── Edge-level TLS enforcement

3. CAA DNS Records (Certificate Authority Authorization):
   example.com. CAA 0 issue "amazon.com"
   example.com. CAA 0 issue "letsencrypt.org"
   example.com. CAA 0 iodef "mailto:security@example.com"
   └── Only listed CAs can issue certs for your domain

4. HSTS via CloudFront Response Headers Policy:
   Strict-Transport-Security: max-age=63072000; includeSubDomains; preload

5. Security Hub + Config Rules:
   ├── alb-http-drop-invalid-header-enabled
   ├── elb-tls-https-listeners-only
   └── cloudfront-viewer-policy-https
```

**Tricky**: CAA records prevent unauthorized CAs from issuing certificates for your domain, but they DON'T prevent someone from using a previously issued rogue certificate. You need Certificate Transparency monitoring for that.

---


**Q31: Your ALB shows "502 Bad Gateway" after enabling HTTPS. What's wrong?**

**A:**

```
Debugging 502 with HTTPS on ALB:

Common causes:
├── 1. Backend (target) health check failing
│   ├── Health check uses HTTPS but target only supports HTTP
│   ├── Health check path returns non-200
│   └── Fix: Match health check protocol to target protocol
│
├── 2. SSL/TLS mismatch between ALB and target
│   ├── ALB configured for HTTPS to target, but target doesn't support TLS
│   ├── Target has self-signed cert that ALB rejects
│   └── Fix: ALB trusts ALL certs to targets (doesn't validate!)
│       Actually: ALB does NOT validate backend certs
│       The issue is the TARGET not supporting TLS at all
│
├── 3. Target security group blocking ALB
│   ├── SG on target doesn't allow traffic from ALB
│   └── Fix: Allow inbound from ALB security group
│
├── 4. Target returning response > 1MB (large response)
│   └── ALB has response size limits
│
└── 5. Target taking too long to respond (timeout)
    ├── ALB idle timeout (default 60s)
    └── Fix: Increase ALB idle timeout or fix target performance

Diagnostic steps:
1. Check target group health: aws elbv2 describe-target-health
2. Check ALB access logs (5XX errors with target details)
3. Test target directly: curl -k https://target-ip:port/health
4. Check security groups and NACLs
5. Verify target is listening on configured port
```

**Tricky**: ALB does NOT validate SSL certificates on backend targets. If you configure HTTPS to targets, ALB will accept self-signed certs. The 502 is NOT from cert validation failure — it's because the target isn't responding correctly to TLS.

**Tricky**: If you change target group protocol from HTTP to HTTPS, you must ensure your application is actually listening on HTTPS. The port number doesn't change the protocol — port 443 with HTTP protocol means ALB sends HTTP to port 443.

---


**Q32: How do you monitor SSL certificate expiration across your entire AWS infrastructure?**

**A:**

**Multi-layered monitoring approach:**

```
Layer 1: AWS Config Rules
├── acm-certificate-expiration-check
│   ├── Configurable threshold (e.g., 30 days)
│   ├── Checks ALL ACM certs in the account
│   └── Marks non-compliant if expiring soon
└── Deploy via StackSets across all accounts

Layer 2: EventBridge + ACM Native Events
├── ACM emits events for approaching expiration
├── "ACM Certificate Approaching Expiration"
└── Route to SNS → PagerDuty/Slack/Email

Layer 3: CloudWatch Alarms
├── Metric: DaysToExpiry (ACM certificates)
├── Alarm at 45, 30, 15, 7 days
└── Action: SNS notification

Layer 4: External monitoring (for non-ACM certs)
├── Lambda function that probes endpoints externally
├── Checks actual TLS cert expiration via socket connection
└── Catches certs not managed by ACM (EC2, on-prem)
```

**Lambda for external certificate monitoring:**
```python
import ssl
import socket
import json
from datetime import datetime, timezone
import boto3

sns = boto3.client('sns')

def check_cert_expiry(hostname, port=443):
    context = ssl.create_default_context()
    with socket.create_connection((hostname, port), timeout=10) as sock:
        with context.wrap_socket(sock, server_hostname=hostname) as ssock:
            cert = ssock.getpeercert()
            expires = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
            expires = expires.replace(tzinfo=timezone.utc)
            days_left = (expires - datetime.now(timezone.utc)).days
            return {
                'hostname': hostname,
                'expires': cert['notAfter'],
                'days_left': days_left,
                'issuer': dict(x[0] for x in cert['issuer'])
            }

def lambda_handler(event, context):
    endpoints = [
        'www.example.com',
        'api.example.com',
        'admin.example.com'
    ]

    alerts = []
    for endpoint in endpoints:
        result = check_cert_expiry(endpoint)
        if result['days_left'] < 30:
            alerts.append(result)

    if alerts:
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456:cert-alerts',
            Subject='SSL Certificate Expiration Warning',
            Message=json.dumps(alerts, indent=2)
        )
```

**Tricky**: ACM auto-renewal can silently fail. Don't assume "we use ACM so certs never expire." Always have monitoring as a safety net. Common failure: someone deleted the DNS validation CNAME record during a DNS cleanup.

---


**Q33: Explain the difference between TLS termination, TLS passthrough, and TLS re-encryption. When would you use each?**

**A:**

```
1. TLS TERMINATION (most common)
   Client ──HTTPS──> ALB ──HTTP──> Target
   ├── ALB decrypts traffic, inspects it, routes it
   ├── Certificate on ALB (ACM)
   ├── Target sees plain HTTP (X-Forwarded-Proto: https)
   ├── ALB can do content-based routing, WAF inspection
   └── Use when: You need L7 features (path routing, host routing, WAF)

2. TLS PASSTHROUGH
   Client ──HTTPS──> NLB ──HTTPS──> Target (same TLS session)
   ├── NLB does NOT decrypt — passes encrypted bytes through
   ├── Certificate on TARGET (not NLB)
   ├── Client TLS session extends to target directly
   ├── NLB cannot inspect content, no L7 routing
   ├── Preserves client IP (no X-Forwarded-For needed)
   └── Use when: End-to-end encryption required, mTLS, compliance

3. TLS RE-ENCRYPTION (bridge/re-encrypt)
   Client ──HTTPS──> ALB ──HTTPS──> Target
   ├── ALB decrypts, inspects, re-encrypts to target
   ├── TWO certificates: one on ALB (ACM), one on target
   ├── ALB gets L7 features AND backend is encrypted
   ├── Extra CPU for double encrypt/decrypt
   └── Use when: Need L7 routing + encrypted backends (compliance)
```

**AWS service mapping:**

| Pattern | AWS Service | Certificate Location |
|---------|-------------|---------------------|
| TLS Termination | ALB (HTTPS listener) | ALB (ACM) |
| TLS Passthrough | NLB (TCP listener) | Target EC2/ECS |
| TLS Re-encryption | ALB (HTTPS listener + HTTPS target group) | ALB + Target |
| TLS Termination at Edge | CloudFront | CloudFront (ACM us-east-1) |

**Tricky**: With TLS termination, traffic between ALB and targets is unencrypted (HTTP). Within a VPC this is generally acceptable, but for PCI-DSS or HIPAA compliance, you might need re-encryption (HTTPS to targets). ALB does NOT validate target certificates — you can use self-signed certs on targets.

---


**Q34: How do you implement certificate pinning? Is it recommended?**

**A:**

**What is certificate pinning:**
```
Normal TLS: Client trusts ANY certificate signed by ANY CA in trust store (~150 CAs)
Pinning:    Client trusts ONLY specific certificate/public key for a given domain

Types:
├── Certificate pinning: Pin entire cert (breaks on renewal)
├── Public key pinning: Pin the public key (survives cert renewal if key unchanged)
└── CA pinning: Pin to specific CA (less restrictive)
```

**Implementation methods:**
```
1. HTTP Public Key Pinning (HPKP) — DEPRECATED!
   Public-Key-Pins: pin-sha256="base64=="; max-age=5184000; includeSubDomains
   └── Removed from browsers (too risky — can brick your site if key lost)

2. Mobile app certificate pinning (still used):
   ├── iOS: ATS (App Transport Security) settings
   ├── Android: network_security_config.xml
   └── Embed expected cert/key hash in app code

3. Backend service pinning:
   ├── Custom TLS verification in code
   └── Trust specific CA only (not full system trust store)
```

**Should you use it?**
```
Recommended:
├── Mobile apps talking to YOUR API (you control both ends)
├── IoT devices (embedded trust)
└── Service-to-service with Private CA (pin your own CA)

NOT Recommended:
├── Public websites (breaks if you change CA)
├── Websites in general (HPKP is dead for good reason)
└── When using CDNs (they may change certificates)
```

**AWS context:**
- API Gateway + mTLS is a better alternative for service auth
- ACM Private CA + trust store is preferred over pinning
- CloudFront doesn't support client-side pinning enforcement

**Tricky**: If you pin a certificate and lose access to that key (key compromise, CA change), your app is BRICKED until users update. HPKP was deprecated because it enabled "ransom attacks" — an attacker who gains temporary site control could set an HPKP header pinning to their key, permanently locking out the real owner.

---


**Q35: You have 100+ microservices on ECS. How do you manage TLS certificates for internal service-to-service communication?**

**A:**

**Option 1: Service Mesh (AWS App Mesh + Envoy) — Recommended**
```
┌─────────────────────────────────────────────┐
│ ECS Task                                     │
│ ├── App container (no TLS awareness needed) │
│ └── Envoy sidecar (handles all mTLS)        │
│     ├── Gets cert from ACM Private CA       │
│     ├── Auto-rotates certificates           │
│     ├── Enforces mTLS between services      │
│     └── App talks to localhost (plain HTTP)  │
└─────────────────────────────────────────────┘

Benefits:
├── Applications don't manage certs
├── Automatic rotation (short-lived certs: hours/days)
├── Zero-trust networking
├── Traffic encryption + mutual authentication
└── Observability (mTLS metadata in metrics)
```

**Option 2: ALB-based (TLS termination per service)**
```
Service A → ALB (ACM cert) → Service B
├── Each service behind internal ALB
├── ACM cert for internal domain (*.internal.example.com)
├── Simple but adds latency (extra hop)
└── No mutual authentication (one-way TLS only)
```

**Option 3: ACM Private CA + Secrets Manager**
```python
# Lambda rotates certs from Private CA into Secrets Manager
# ECS tasks pull cert from Secrets Manager at startup

def rotate_cert():
    # Issue new cert from Private CA
    cert_arn = acmpca.issue_certificate(
        CertificateAuthorityArn=CA_ARN,
        Csr=generate_csr(service_name),
        SigningAlgorithm='SHA256WITHRSA',
        Validity={'Value': 7, 'Type': 'DAYS'}
    )

    # Store in Secrets Manager
    secrets.put_secret_value(
        SecretId=f'{service_name}-tls-cert',
        SecretString=json.dumps({
            'cert': cert_pem,
            'key': key_pem,
            'chain': chain_pem
        })
    )
```

**Option 4: SPIFFE/SPIRE (advanced)**
```
├── SPIRE server issues short-lived SVIDs (SPIFFE Verifiable Identity Document)
├── Automatic mTLS between workloads
├── Works across Kubernetes, ECS, EC2, on-prem
└── Integration with AWS IAM roles for trust
```

**Tricky**: For 100+ microservices, manual cert management is a non-starter. The service mesh approach (App Mesh or Istio) is the most operationally sound because it removes certificate management from application code entirely.

---


**Q36: What are CAA records? How do they complement ACM?**

**A:**

**CAA (Certificate Authority Authorization) DNS records:**
```
Purpose: Specify which CAs are allowed to issue certificates for your domain
Type: DNS CAA record
Enforcement: CAs MUST check CAA before issuance (mandatory since Sep 2017)

Record format:
domain.com.  CAA  <flags> <tag> <value>

Tags:
├── issue       — Authorize CA to issue standard certs
├── issuewild   — Authorize CA to issue wildcard certs
├── iodef       — URL/email for violation reporting
└── contactemail — Contact email (newer extension)
```

**Example CAA configuration for AWS:**
```dns
; Allow only Amazon CA and Let's Encrypt to issue certs
example.com.  CAA  0 issue "amazon.com"
example.com.  CAA  0 issue "letsencrypt.org"

; Allow only Amazon for wildcard certs
example.com.  CAA  0 issuewild "amazon.com"

; Report violations
example.com.  CAA  0 iodef "mailto:security@example.com"

; Block all other CAs (empty issue prevents issuance)
; If ANY issue records exist, unlisted CAs are denied
```

**Route 53 Terraform:**
```hcl
resource "aws_route53_record" "caa" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "CAA"
  ttl     = 300

  records = [
    "0 issue \"amazon.com\"",
    "0 issue \"letsencrypt.org\"",
    "0 issuewild \"amazon.com\"",
    "0 iodef \"mailto:security@example.com\""
  ]
}
```

**Tricky**: CAA records are checked hierarchically. If `api.example.com` has no CAA record, the CA checks `example.com`. If neither has CAA, any CA can issue. CAA is PREVENTIVE only — it doesn't revoke already-issued rogue certificates.

**Tricky**: For ACM, the CAA value must be `"amazon.com"` (not `"amazonaws.com"` or `"aws.amazon.com"`). This is a common misconfiguration that causes ACM issuance to fail.

---


**Q37: Compare Let's Encrypt vs ACM. When would you use each?**

**A:**

| Feature | Let's Encrypt | AWS ACM |
|---------|--------------|---------|
| Cost | Free | Free (public certs) |
| Validity | 90 days | 13 months (395 days) |
| Renewal | Automatic via ACME client (certbot) | Automatic (AWS managed) |
| Validation | DV (HTTP-01, DNS-01) | DV (DNS, Email) |
| Wildcard | Yes (DNS-01 only) | Yes |
| Rate limits | 50 certs/domain/week | 2,500 certs/account |
| Private key | You have full access | Cannot export (ACM public) |
| Use on EC2 | Yes (install directly) | No (need ALB/CloudFront) |
| Works outside AWS | Yes (any server) | No (AWS services only) |
| Cert types | DV only | DV only |
| OCSP Stapling | Supported | AWS manages |
| Client support | Very broad | Very broad |
| Best for | EC2 direct, non-AWS, hybrid | ALB, CloudFront, API GW |

**When to use Let's Encrypt:**
- EC2 instance serving HTTPS directly (no load balancer)
- On-premises or hybrid environments
- Non-AWS cloud providers
- Development/testing environments
- When you need private key access

**When to use ACM:**
- Any AWS service that integrates with ACM
- Production workloads behind ALB/CloudFront
- When you want zero certificate management overhead
- Multi-region deployments with automatic renewal

**Certbot automation on EC2:**
```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Obtain certificate
sudo certbot --nginx -d example.com -d www.example.com

# Auto-renewal (cron/systemd timer)
sudo certbot renew --dry-run

# Certbot auto-installs a systemd timer for renewal
# Renews at 60 days (30 days before expiry)
```

**Tricky**: Let's Encrypt has strict rate limits. If you accidentally revoke and reissue many times during testing, you can hit the limit. Use the staging environment for testing: `--staging` flag with certbot.

---


**Q38: How do you implement end-to-end mTLS on AWS API Gateway?**

**A:**

**Architecture:**
```
Client (with client cert)
    │
    ▼ TLS + Client Certificate
┌─────────────────────────────────────┐
│ API Gateway (Custom Domain + mTLS)  │
│ ├── Truststore in S3 (CA cert)     │
│ ├── Validates client certificate    │
│ ├── Rejects clients without valid cert │
│ └── Optional: Cert info in headers  │
└──────────────────┬──────────────────┘
                   │
                   ▼
          Lambda / Backend
```

**Setup steps:**
```bash
# 1. Create truststore (bundle of trusted CA certs in PEM)
cat root-ca.pem intermediate-ca.pem > truststore.pem

# 2. Upload to S3
aws s3 cp truststore.pem s3://my-truststore-bucket/truststore.pem

# 3. Create custom domain with mTLS
aws apigatewayv2 create-domain-name \
  --domain-name api.example.com \
  --domain-name-configurations \
    CertificateArn=arn:aws:acm:us-east-1:123:certificate/xxx \
  --mutual-tls-authentication \
    TruststoreUri=s3://my-truststore-bucket/truststore.pem
```

**Terraform:**
```hcl
resource "aws_apigatewayv2_domain_name" "api" {
  domain_name = "api.example.com"

  domain_name_configuration {
    certificate_arn = aws_acm_certificate.api.arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }

  mutual_tls_authentication {
    truststore_uri = "s3://${aws_s3_bucket.truststore.id}/truststore.pem"
  }
}
```

**Client-side call:**
```bash
curl --cert client-cert.pem --key client-key.pem \
  https://api.example.com/resource
```

**Accessing client cert info in Lambda:**
```python
def handler(event, context):
    # Client cert details available in request context
    client_cert = event['requestContext']['authentication']['clientCert']
    subject = client_cert['subjectDN']      # "CN=service-a,O=Example"
    issuer = client_cert['issuerDN']        # "CN=Internal CA,O=Example"
    serial = client_cert['serialNumber']
    validity = client_cert['validity']       # notBefore, notAfter
```

**Tricky**: mTLS on API Gateway requires a CUSTOM DOMAIN NAME. You cannot use the default `execute-api.amazonaws.com` endpoint. Also, the truststore S3 object must have versioning enabled for proper update handling.

---


**Q39: What happens when a Root CA expires? How does this affect your infrastructure?**

**A:**

**Real-world example: Let's Encrypt DST Root CA X3 (expired Sep 30, 2021)**
```
Impact:
├── Older clients (Android < 7.1, old OpenSSL) stopped trusting LE certs
├── Millions of IoT devices couldn't connect to HTTPS endpoints
├── Automated systems with outdated CA bundles failed
└── Services dependent on LE certs saw partial outages

Why it happened:
├── Original: DST Root CA X3 → Intermediate → Leaf certs
├── DST Root CA X3 expired (old root)
├── New chain: ISRG Root X1 → Intermediate → Leaf certs
├── Older devices didn't have ISRG Root X1 in trust store
└── Cross-signing bridge partially mitigated the issue
```

**Impact on AWS infrastructure:**
```
ACM certificates (public):
├── Amazon Root CA 1/2/3/4 are trusted by major browsers
├── AWS updates roots as needed
├── Your responsibility: NOTHING (AWS manages chain)
└── Risk: Very low for ACM public certs

Imported certificates:
├── You manage the chain
├── If your CA's root expires, you need new certs
└── Monitor: CA vendor communications about root transitions

ACM Private CA:
├── YOUR root CA has an expiry you set (e.g., 10 years)
├── Plan root rotation WELL before expiry (years ahead)
├── Cross-certification can extend trust during transition
└── All certs issued by expired CA become untrusted!

On EC2 (OS trust store):
├── Old AMIs may have outdated CA bundles
├── Fix: Update ca-certificates package regularly
│   sudo yum update ca-certificates   (Amazon Linux)
│   sudo apt update && sudo apt install ca-certificates  (Ubuntu)
└── Automate via SSM patch baseline or user data
```

**Preventive measures:**
1. Keep OS CA bundles updated across fleet (SSM Patch Manager)
2. Monitor CA vendor announcements for root transitions
3. Set Private CA root validity appropriately (10+ years)
4. Plan subordinate CA rotation (3-5 years)
5. Test with older clients if supporting legacy devices

**Tricky**: When a root CA expires, the ENTIRE chain under it becomes untrusted — even if your leaf certificate hasn't expired yet. The chain is only as strong as its weakest (or oldest) link.

---


**Q40: Design a complete certificate lifecycle management strategy for an enterprise on AWS.**

**A:**

```
Certificate Lifecycle Management Strategy:

┌─────────────────────────────────────────────────────────────────┐
│                    CERTIFICATE INVENTORY                         │
├─────────────────────────────────────────────────────────────────┤
│ Public-facing (ACM):                                            │
│ ├── *.example.com (ALB, CloudFront)                             │
│ ├── api.example.com (API Gateway)                               │
│ └── Auto-renewed by ACM                                         │
│                                                                  │
│ Internal (ACM Private CA):                                       │
│ ├── Service mesh certs (auto-rotated: 24h validity)             │
│ ├── Internal APIs (auto-rotated: 7-day validity)                │
│ └── Client auth certs (90-day validity)                         │
│                                                                  │
│ Imported/Third-party:                                            │
│ ├── EV cert for payment page (DigiCert, 1-year)                │
│ └── Legacy system certs (manual tracking)                       │
└─────────────────────────────────────────────────────────────────┘

Lifecycle Phases:
1. PROVISIONING
   ├── Public → ACM (DNS validation, Terraform/CloudFormation)
   ├── Internal → ACM Private CA (automated via Lambda/service mesh)
   └── Third-party → CSR generation → CA → Import to ACM

2. DEPLOYMENT
   ├── Infrastructure-as-Code (Terraform manages cert associations)
   ├── Blue-green for cert changes on EC2
   └── ACM handles deployment to ALB/CloudFront automatically

3. MONITORING
   ├── AWS Config: acm-certificate-expiration-check
   ├── EventBridge: ACM expiration events → SNS/PagerDuty
   ├── Lambda: External endpoint certificate checking
   ├── Security Hub: Central compliance dashboard
   └── Custom CloudWatch dashboard for cert inventory

4. ROTATION/RENEWAL
   ├── ACM public: Automatic (verify DNS CNAME exists)
   ├── ACM Private CA: Automatic or Lambda-driven
   ├── Imported: Automated pipeline (reimport before expiry)
   └── Let's Encrypt: Certbot auto-renewal with monitoring

5. REVOCATION
   ├── ACM Private CA: CRL distribution via S3 + CloudFront
   ├── OCSP responder configuration
   ├── Incident response playbook for key compromise
   └── Automation: Revoke + reissue + redeploy in minutes

6. DECOMMISSIONING
   ├── Delete unused certificates
   ├── Audit: No resources referencing deleted cert
   └── Clean up DNS validation records (optional)
```

**Automation pipeline (GitOps approach):**
```yaml
# cert-rotation-pipeline.yml
name: Certificate Rotation Pipeline
on:
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6 AM

jobs:
  check-certs:
    steps:
      - name: Check all certificate expirations
        run: |
          aws acm list-certificates --query 'CertificateSummaryList[?Status==`ISSUED`]' | \
          jq -r '.[] | select(.NotAfter | fromdate < (now + 2592000))' # Expiring in 30 days

      - name: Rotate imported certs if needed
        run: ./scripts/rotate-imported-certs.sh

      - name: Verify all endpoints
        run: ./scripts/check-endpoint-tls.sh

      - name: Alert on failures
        if: failure()
        run: ./scripts/send-pagerduty-alert.sh
```

**Tricky**: The biggest risk in certificate management is the "unknown unknowns" — certificates that nobody is tracking. Shadow IT, developer test environments, IoT devices with embedded certs. Start with a comprehensive discovery/inventory phase before building automation.
