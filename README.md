# LetsDefend Challenge Writeup: HTTP Basic Auth

**Category:** DFIR (Digital Forensics & Incident Response)
**Difficulty:** Easy
**Tool used:** Wireshark, CyberChef
**Author:** Nabin Tiwari

---

## Scenario

A log entry indicated a possible attack, and analysts were asked to investigate a captured network traffic file (`webserver.em0.pcap`) to gather details about the incident. The objective was to reconstruct what happened between a client and a web server using packet-level evidence.

---

## Tools

- **Wireshark** — for opening and analyzing the `.pcap` file, applying display filters, and reconstructing HTTP conversations via *Follow > HTTP Stream*.
- **CyberChef** — for decoding the Base64-encoded credentials found in the `Authorization` header.

---

## Investigation

### Step 1 — Open the capture

The file was loaded directly into Wireshark. The capture contains 65 packets total, a mix of ICMP (ping) traffic and TCP/HTTP traffic between a client (`192.168.63.100`) and a server (`1.1.1.5`).

![PCAP overview in Wireshark](images/01-pcap-overview.png)

---

### Q1 — How many HTTP GET requests are in the pcap?

**Filter used:**
```
http.request.method == GET
```

This filter narrows the capture down to only HTTP requests using the GET method. Rather than scrolling and manually counting, the packet list status bar at the bottom right of Wireshark shows the displayed packet count after the filter is applied — a much faster way to get the number in larger captures.

![HTTP GET filter applied](images/02-http-get-filter.png)

**Answer: `5`**

---

### Q2 — What is the server operating system?

To identify the server OS, it helps to understand the basic HTTP request/response model: the client sends a **request**, and the server replies with a **response**. The response's `Server` header discloses the software (and often the OS) that generated it.

Right-clicking any HTTP packet and choosing **Follow > HTTP Stream** reconstructs the full request/response conversation in one readable view.

![Server header in HTTP stream](images/03-server-header.png)

The response header read:
```
Server: Apache/2.2.15 (FreeBSD) DAV/2 mod_ssl/2.2.15 OpenSSL/0.9.8n
```

This single line discloses three separate pieces of information: the web server software, the OS, and the SSL library version — all at once, which is exactly the kind of banner information an attacker would fingerprint for exploit targeting.

**Answer: `FreeBSD`**

---

### Q3 — What is the name and version of the web server software?

Read from the same `Server` header shown above.

**Answer: `Apache/2.2.15`**

---

### Q4 — What is the version of OpenSSL running on the server?

Also read from the same `Server` header.

**Answer: `OpenSSL/0.9.8n`**

*(Worth flagging in a real investigation: OpenSSL 0.9.8n is extremely outdated and pre-dates the Heartbleed disclosure era — a real environment running this would be a strong candidate for vulnerability scanning.)*

---

### Q5 — What is the client's user-agent information?

Where the `Server` header identifies the server-side software, the `User-Agent` header in the client's **request** identifies the software making the request. Using the same Follow HTTP Stream technique on a request packet:

![User-Agent header in HTTP stream](images/04-user-agent.png)

**Answer: `Lynx/2.8.7rel.1 libwww-FM/2.14 SSL-MM/1.4.1 OpenSSL/0.9.8n`**

Lynx is a text-based, terminal-only browser — unusual for typical end-user traffic, and often associated with scripted access, automation, or minimal-footprint reconnaissance activity.

---

### Q6 & Q7 — Basic Authentication credentials

This capture contains two distinct HTTP response codes, visible with:
```
http.response.code
```

![HTTP response codes 401 and 200](images/05-response-codes.png)

- **401 Unauthorized** — the client's request lacked valid credentials.
- **200 OK** — the server successfully authenticated and served the request.

Filtering specifically for the successful authentication:
```
http.response.code == 200
```

...and following that stream reveals the `Authorization` header sent by the client:

![Authorization header with Base64 credentials](images/06-authorization-header.png)

```
Authorization: Basic d2ViYWRtaW46VzNiNERtMW4=
```

HTTP Basic Auth sends credentials as `Base64(username:password)` — **not encryption, just encoding** — meaning anyone who can capture this traffic can trivially recover the plaintext credentials. Decoding the string in CyberChef using the **From Base64** operation:

![CyberChef Base64 decode](images/07-cyberchef-decode.png)

```
webadmin:W3b4Dm1n
```

**Answer (Q6 — Username): `webadmin`**
**Answer (Q7 — Password): `W3b4Dm1n`**

---

## Summary

This capture documents an HTTP Basic Authentication exchange over an **unencrypted HTTP/1.0 connection** to a web server running severely outdated software (Apache 2.2.15 on FreeBSD, OpenSSL 0.9.8n). The client, using the Lynx text browser, was initially rejected with a 401 Unauthorized response and subsequently authenticated successfully with valid credentials (`webadmin:W3b4Dm1n`), sent in the clear as a Base64-encoded string.

Because Basic Auth performs no encryption of its own, and this traffic was not wrapped in TLS/HTTPS, the credentials would be trivially recoverable by anyone positioned to capture the traffic — for example, via a man-in-the-middle or a compromised network segment. This is a textbook case of **credential exposure over cleartext HTTP**.

**MITRE ATT&CK mapping:**
- **T1040** — Network Sniffing (credentials recoverable from captured traffic)
- **T1552.001** — Unsecured Credentials: Credentials In Files/Traffic

**Remediation recommendations:**
- Enforce HTTPS/TLS for all authenticated endpoints; never transmit Basic Auth over plain HTTP.
- Replace Basic Auth with a token-based or session-based authentication scheme where possible.
- Patch/upgrade the web server stack — Apache 2.2.x and OpenSSL 0.9.8n are long past end-of-life and carry known vulnerabilities.
- Monitor for repeated 401 responses followed by a 200, which can indicate credential brute-forcing or guessing attempts.

---

*Lab completed on [LetsDefend](https://app.letsdefend.io/challenge/http-basic-auth) — SOC Analyst DFIR track.*
