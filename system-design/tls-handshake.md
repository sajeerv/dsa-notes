# TLS Handshake in a Web Browser

## Purpose

When you open an HTTPS website, your browser and the website first establish a protected connection. This setup is called the TLS handshake. After it succeeds, they send the website’s HTTP request and response through that protected connection.

This note describes a typical **full TLS 1.3 handshake** for a browser connecting over TCP. Some connections use TLS 1.2 or reuse a previous TLS session, so the exact messages can differ. HTTP/3 uses QUIC instead of TCP, with TLS 1.3 integrated into QUIC.

## What TLS provides

TLS is designed to provide:

- **Privacy:** people observing the network cannot read the protected web traffic.
- **Integrity:** changes to protected traffic can be detected.
- **Server authentication:** the browser can verify that it is talking to the server for the requested website.

TLS does not prove that a website is honest or safe to use. It protects the connection to the site whose identity the browser verified.

TLS protects the HTTP request and response contents, but it is not a VPN: the destination IP address and traffic timing/volume may still be visible to network observers.

## Before TLS starts

When you enter a website address, the browser first needs to find and reach a server:

1. It gets an IP address for the website, usually through DNS.
2. For a typical HTTPS connection, it establishes a TCP connection to that IP address.
3. It starts TLS on that connection.

DNS and TCP happen before the usual TLS handshake. HTTP/3 is a transport difference: it uses QUIC rather than TCP, while still using TLS 1.3 security mechanisms.

## The handshake, step by step

### 1. Browser says hello

The browser sends a ClientHello message. In simple terms, it says:

- “I want to use HTTPS and support these TLS versions.”
- “Here are the cryptographic methods I can use.”
- “Here is the website name I am trying to reach.”
- “Here is my temporary public key share.”

The website name helps a server hosting many sites choose the right website certificate. The temporary key information is used to create shared secret keys without sending those secrets over the network.

A key share is public information calculated from a fresh private value. The browser sends the public share, but keeps the private value to itself. The server does the same. Each side combines its private value with the other side’s public share to calculate the same shared secret.

### 2. Server chooses the settings

The server responds with a ServerHello. It selects compatible TLS settings and sends its own temporary key-exchange information.

That server information is its public key share; the server keeps its matching temporary private value secret.

Each side creates a fresh secret locally when the handshake starts. The browser’s TLS software asks the browser’s operating system for random data and uses it to create the browser’s temporary secret. This secret does not come from the website, is not in the website’s certificate, and is never sent over the network. The server independently creates its own temporary secret the same way.

From this shared secret, both sides derive keys for protecting the connection. In TLS 1.3, most of the remaining handshake messages are encrypted.

That means the server’s certificate and proof messages are generally sent inside the encrypted part of the TLS 1.3 handshake.

### Why exchange temporary key shares?

The browser and server need a shared secret to make their traffic-encryption keys. They do not send that secret directly. Instead, each creates a fresh private value, sends a public key share derived from it, and combines the other side’s public share with its own private value. Both sides arrive at the same secret; a network observer sees the public shares but cannot feasibly calculate that secret.

In a common certificate-authenticated full browser handshake, the browser and server create fresh, temporary Diffie-Hellman key shares for that connection. TLS 1.3 forbids reusing key-share values across connections. This creates fresh traffic keys and provides **forward secrecy**: if the server’s long-term certificate private key is stolen later, previously recorded sessions should still be unreadable, provided the temporary secrets were erased and the endpoints were not compromised.

### Who creates the temporary values?

The browser’s TLS software uses its temporary secret to calculate a public key share, then sends that public share to the server. The server uses its own temporary secret to calculate a different public share and sends that to the browser.

A public share is **not the secret encrypted**. It is a value computed from the secret using the key-exchange math. There is no “decrypt the share” step. Each side uses its own private secret plus the other side’s public share to calculate the shared result.

The browser does not get the server’s private value, and the server does not get the browser’s. Each combines its own private value with the other side’s public share and arrives at the same result.

Here is a deliberately tiny Diffie-Hellman example. The numbers are small enough to check by hand, so they are **not secure**:

~~~text
Public settings both sides know: p = 23, g = 5

Browser chooses private a = 6
Browser calculates public A = 5^6 mod 23 = 8

Server chooses private b = 15
Server calculates public B = 5^15 mod 23 = 19

They exchange only A = 8 and B = 19.

Browser calculates B^a mod 23 = 19^6 mod 23 = 2
Server calculates A^b mod 23 = 8^15 mod 23 = 2
~~~

Why are the results equal? The browser calculates (g^b)^a, which is g^(b × a). The server calculates (g^a)^b, which is g^(a × b). Since b × a equals a × b, both calculations produce g^(a × b), and therefore the same result modulo p.

This is exponentiation and modular arithmetic, not ordinary addition of the two shares.

Both got the same result, 2, without sending their private values or the result. A real TLS key exchange uses vastly larger values and cryptographic algorithms, so an observer cannot feasibly work backward from the public shares. TLS then feeds the shared result into a key-derivation function to produce the actual traffic keys; the shared result itself is not used directly as the page-encryption key.

The certificate private key has a different job: it signs handshake data to prove the server’s identity. It is not the shared secret and is not used to decrypt each web page.

### How does encryption and decryption work?

After deriving the shared secret, the browser and server use a key-derivation function to make separate traffic keys for each direction:

- The browser encrypts outgoing records with its browser-to-server key. The server uses the matching receive key to decrypt them.
- The server encrypts replies with its server-to-browser key. The browser uses the matching receive key to decrypt them.

TLS uses symmetric authenticated encryption for the HTTP data. The sender turns readable data into ciphertext and adds an authentication tag. The receiver uses its matching key to recover the data and verify the tag. If the data was changed or the tag is invalid, TLS rejects it. The symmetric keys make this efficient for large amounts of data; public-key cryptography is used during the handshake for key agreement and authentication.

### 3. Server presents its certificate

The server sends its certificate, often along with intermediate certificates that connect it to a trusted certificate authority (CA).

The browser checks, among other things:

- Does the certificate chain lead to a CA trusted by the browser or operating system?
- Is the certificate valid for the website name in the address bar?
- Is it within its validity period and acceptable for server authentication?
- Can the server prove that it has the private key matching the certificate?

The certificate is public. It contains the server’s public key and identity information. The matching private key stays with the server.

### 4. Server proves it owns the private key

The server sends a CertificateVerify message: a digital signature over the handshake so far. The browser checks that signature using the public key in the certificate.

This proves the server possesses the matching private key and ties the certificate to this specific handshake. The certificate does not encrypt all the website traffic.

### 5. Server says “finished”

The server sends a Finished message. It is a cryptographic check over the handshake so far. The browser verifies it to confirm that the handshake messages were not altered and that both sides derived compatible keys.

### 6. Browser says “finished”

The browser sends its own Finished message. The server verifies it. In an ordinary web visit, the browser usually does **not** send a client certificate; the server is the party authenticated by TLS.

### 7. The browser sends the HTTP request

The handshake is complete. The browser can now send its HTTP request—such as “GET the home page”—inside the encrypted connection. The server’s response travels through that same protected connection.

~~~text
Browser                                      Website server
   |                                                |
   |--- ClientHello: supported options, key share ->|
   |<-- ServerHello: chosen options, key share -----|
   |<-- EncryptedExtensions, certificate, signature |
   |<-- Finished ------------------------------------|
   |--- Finished ----------------------------------->|
   |                                                |
   |<==== Encrypted HTTP request and response ====>|
~~~

## What the certificate does—and does not do

A useful way to think about the certificate:

- **Certificate:** a public ID card that links a website name to a public key.
- **Private key:** the secret that proves the server controls that identity.
- **Temporary key exchange:** lets browser and server derive fresh shared traffic keys.
- **Symmetric encryption:** efficiently protects the actual HTTP data after the handshake.

The certificate is not a password, and the browser does not use the server’s private key to encrypt every page.

## TLS and mTLS: what is different?

**mTLS means mutual TLS.** It is the same TLS protocol and uses the same key exchange and symmetric encryption. The difference is that the server also asks the client to prove its identity with a client certificate.

| | Ordinary browser TLS | Mutual TLS (mTLS) |
| --- | --- | --- |
| Server proves its identity to the client | Yes, usually with a server certificate | Yes |
| Client proves its identity to the server during TLS | Usually no | Yes, with a client certificate |
| Common use | Public websites; users sign in later through the website | Devices, internal services, and managed enterprise connections |

In ordinary browser TLS, the browser verifies the website certificate, but the website usually does not ask the browser for a certificate. If the user signs in, that usually happens afterward through an application login, session cookie, or token.

In mTLS, the server sends a CertificateRequest during the handshake. The client sends its certificate and signs handshake data with the matching private key. The server checks that certificate and signature. The client still verifies the server’s certificate, so both ends authenticate each other.

mTLS does **not** replace authorization. A valid client certificate identifies the connecting client; the application still decides which actions that client is allowed to perform.

### How IoT devices get and store certificates

1. **Give each device a unique identity.** The fleet registry records which device is allowed to connect and what it may do.
2. **Generate a key pair on the device.** Prefer a TPM or secure element. The private key should be created and kept there so it cannot be copied off the device. The device can still use it to prove possession during TLS.
3. **Request a certificate.** The device creates a Certificate Signing Request (CSR) containing its public key and device identity, and signs the request with its private key. It sends the CSR to an enrollment service. An installer, manufacturing credential, one-time claim code, or other bootstrap identity lets the service decide whether this device is allowed to enroll.
4. **Have the CA issue it.** The enrollment service checks the device and request, then asks the fleet’s certificate authority (CA) to sign a certificate binding the device identity to the public key. The CA returns the certificate to the device. The private key is never sent to the CA.
5. **Store and use it.** The certificate is public and can be stored in the device’s filesystem. Keep the private key in the TPM or secure element. The device also needs trusted CA certificates so it can verify the backend’s server certificate. At connection time, it sends its certificate and uses the protected private key to sign the TLS handshake.
6. **Rotate or revoke.** Before expiration, the device enrolls a replacement certificate, preferably using a newly generated key pair. If a device is lost or compromised, disable its identity or revoke its certificate. Do not reuse one private key or client certificate across the fleet.

If a device has no secure hardware, its private key may be stored in an encrypted, access-restricted filesystem, but that offers weaker protection against someone with physical access. A standard enrollment protocol such as [EST (RFC 7030)](https://www.rfc-editor.org/rfc/rfc7030.html) can handle certificate enrollment and renewal; cloud IoT platforms may provide their own provisioning service.

## Why the browser may show a warning

If the certificate is expired, names a different website, chains to an untrusted CA, or otherwise fails validation, the browser cannot reliably confirm the site’s identity. It may block the connection or show a security warning.

A valid certificate means the browser verified the website identity according to its trust rules. It does not mean the site itself is reputable.

## Variations

- **TLS 1.2:** Still supported by some websites. Its handshake has different message ordering and may require additional round trips. TLS 1.3 simplified and shortened the typical full handshake.
- **Session resumption:** On a later visit, the browser and server may reuse securely established session information to reconnect faster. A resumed handshake can omit the certificate exchange.
- **HTTP/3:** Uses QUIC rather than TCP. It still uses TLS 1.3 to authenticate the server and establish traffic keys.

## FAQ: attacks and protections

| Attacker’s attempt | What TLS does | What TLS does not do |
| --- | --- | --- |
| **Listen on public Wi-Fi and read traffic** | Encrypts HTTP content, including headers, cookies, and page data, so a passive observer cannot read it without the traffic keys. | Does not hide all metadata. The destination IP and traffic timing/volume may still be visible; DNS protection is a separate matter. |
| **Pretend to be the website (man-in-the-middle)** | The browser checks the certificate chain and verifies that the certificate names the requested website. The server also signs the handshake to prove it controls the certificate’s private key. | Cannot protect the user if a trusted CA or endpoint is compromised, a malicious root certificate is installed, or the user bypasses a certificate warning. |
| **Change a request or inject data** | Authenticated encryption detects changes to protected TLS records; invalid records are rejected. | Does not stop an attacker from blocking or disrupting the connection. |
| **Force an old or weaker TLS version** | The supported versions and handshake choices are bound into the handshake. Current TLS 1.3 guidance forbids TLS 1.0 and 1.1 negotiation. | Cannot fix a server that is configured to permit obsolete protocols or weak application behavior. |
| **Steal the server certificate key later and decrypt recorded traffic** | Fresh temporary key shares in a full TLS 1.3 handshake provide forward secrecy, protecting past recorded sessions. | Does not protect a currently compromised server, browser, or device. Attackers controlling an endpoint can see data before encryption or after decryption. |
| **Replay or repeat an action** | TLS sequence tracking detects replayed or reordered records within a connection. | TLS 1.3 early data (0-RTT), when enabled, can be replayed. Applications should not use it for actions that must happen only once unless they have replay protection. |
| **Phish the user with a look-alike site** | TLS can securely connect to the exact domain shown in the address bar. | HTTPS does not prove that the site is honest. A phishing site can have a valid certificate for its own look-alike domain. Check the domain and do not treat the padlock as a trust guarantee. |

### Common questions

**What if someone in the middle presents a fake certificate?**

The browser checks that the certificate chain leads to a CA it trusts, that the certificate’s Subject Alternative Name matches the exact website domain, that it is valid for server use and within its validity period, and that the server proves it has the matching private key by signing the handshake. A self-signed certificate or one for a different domain fails these checks, so the browser shows an error or blocks the connection. The hostname check follows the service-identity rules in [RFC 9525](https://www.rfc-editor.org/rfc/rfc9525.html).

An attacker could get around this if a trusted CA wrongly issues a certificate for the site, the attacker installs a new trusted root certificate on the device, the device is compromised, or the user bypasses the browser warning. Some workplaces deliberately install a company root certificate to inspect managed-device traffic. TLS cannot distinguish that trusted inspection proxy from another trusted certificate authority.

**Why not encrypt a random key with the server’s public key?**

That is a possible design: the browser could create a random session key, encrypt it with the server’s public key, and send it to the server to decrypt with its private key. Older TLS versions had an RSA key-transport mode like this.

The weakness is what happens later: if an attacker recorded the connection and the server’s long-term private decryption key is stolen afterward, the attacker could recover the old session key from the recording and decrypt that old traffic. Current TLS 1.3 removes static RSA key transport. Its public-key key-exchange methods use fresh temporary values, which provide forward secrecy. RSA can still be used for a certificate signature; it just is not used to transport the traffic secret. ([RFC 9846](https://www.rfc-editor.org/rfc/rfc9846.html))

So the word “share” is misleading: the browser does not send the secret. It sends a public key share. The browser and server each combine the other side’s public share with their own temporary private value and independently calculate the same shared secret. They then derive the traffic keys from it. Public-key operations help establish and authenticate those keys; faster symmetric encryption protects the actual web data.

**Can an eavesdropper calculate the shared secret from the key shares?**

For the key-exchange methods used by TLS, calculating it from the exchanged public values is designed to be computationally infeasible. The attacker can see the key shares, but not the fresh private values held by the browser and server.

**What is the public key in the server’s certificate used for?**

The browser uses it to verify the server’s CertificateVerify signature. That proves the server holds the matching private key and ties the server’s identity to this handshake. The browser also checks that the certificate is trusted and names the site it requested. This certificate key authenticates the server; the temporary key shares establish the shared secret, and derived symmetric keys encrypt the web traffic.

**Does the server use its certificate’s private key to decrypt my request?**

No. The certificate private key proves the server’s identity by signing handshake data. The browser and server use derived symmetric traffic keys to encrypt and decrypt application data.

**Can someone steal my password on an open Wi-Fi network?**

A passive Wi-Fi observer cannot read a correctly established HTTPS connection. An active attacker may redirect or block traffic, but the browser’s certificate and hostname checks should reject an impostor server. Do not click through certificate warnings.

**How does symmetric encryption protect the web data, and why not use asymmetric encryption for everything?**

After the handshake, TLS derives matching traffic keys for each direction: one for browser-to-server data and another for server-to-browser data. It splits the data into records. For each record, an authenticated-encryption algorithm (such as AES-GCM or ChaCha20-Poly1305) uses the direction’s key and a unique nonce to produce ciphertext plus an authentication tag.

The receiver uses its matching key and nonce to decrypt the record and check the tag. If the record was changed or the wrong key was used, the check fails. “Symmetric” means both ends have matching secret keys for each direction; it does not mean the exact same key is used both ways.

Asymmetric operations use a public/private key pair and are much more computationally expensive, so they are not a good fit for encrypting every byte of a page. TLS uses public-key cryptography during the handshake to authenticate the server and agree on fresh secret material, then uses fast symmetric authenticated encryption for the large stream of HTTP data. The server’s certificate private key does not decrypt each request or page.

**Does HTTPS make a website safe?**

No. It protects the connection to the verified domain. It does not stop phishing, malware on your device, a compromised website, or someone who tricks you into sharing information.

**What if the server’s private key is stolen?**

An attacker may be able to impersonate the site until the certificate is revoked or expires. With a full TLS 1.3 handshake using fresh temporary key shares, the stolen certificate key alone should not decrypt previously recorded sessions. It cannot protect data handled by a server that is actively compromised.

**What is the Finished message checking?**

Each side verifies a cryptographic check over the handshake. This confirms that both derived matching secrets and that the negotiated handshake was not silently altered.

## References

- [RFC 9846 — The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc9846.html). Published July 2026; it obsoletes RFC 8446.
- [RFC 9525 — Service Identity in TLS](https://www.rfc-editor.org/rfc/rfc9525.html), which describes checking the service name against the certificate.
- [RFC 7030 — Enrollment over Secure Transport (EST)](https://www.rfc-editor.org/rfc/rfc7030.html), a standard for certificate enrollment and re-enrollment.
- [AWS IoT device provisioning guidance](https://docs.aws.amazon.com/wellarchitected/latest/iot-lens/device-provisioning.html) as one example of fleet provisioning.
- [OWASP Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html).
- [MDN — Transport Layer Security (TLS)](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Transport_Layer_Security).

