# Task 3 — Low-Level Network Packet Inspection & Protocol Analysis

**Prepared by:** Uzair Ahmed (Student ID: B4/963)
**Target Scope:** Loopback interface (127.0.0.1) — isolated Kali Linux VM
**Assessment Type:** Packet-level protocol analysis and clear-text credential exposure testing
**Tools Used:** Wireshark, pyftpdlib, Python Flask, VMware Kali Linux
**Assessment Window:** 21 August 2026
**Report Classification:** Internal / Lab Use — Internship Task 3

---

## 📄 Summary

This task involved a **packet-by-packet inspection** of network traffic using Wireshark to understand TCP connection behaviour and to identify **clear-text authentication exposure** in unencrypted application protocols.

Two controlled services were set up on an isolated Kali Linux VM:

| Port | Protocol | Service | Notes |
|------|----------|---------|-------|
| 21 | FTP | pyftpdlib 2.2.0 | Plain-text authentication; USER/PASS visible in capture |
| 8080 | HTTP | Flask dev server | Plain-text POST body; login form served without TLS |

All traffic was captured on the **loopback (`lo`) interface** using Wireshark.

---

## 🔍 Methodology

1. Started each service and began a live Wireshark capture on `lo` before initiating any connection.
2. Connected as a client (FTP login / HTTP form submission) to generate a fresh TCP handshake and application-layer exchange.
3. Filtered captured traffic by protocol and `tcp.port` to isolate the three-way handshake for each service.
4. Expanded protocol layers (Ethernet → IP → TCP → Application) of representative packets.
5. Used **Statistics → Conversations** to check traffic volume/endpoints for signs of unauthorized data movement.

---

## 🧩 Key Findings

### 1. TCP Three-Way Handshake
The standard `SYN → SYN,ACK → ACK` handshake was observed immediately before any application data was exchanged in **both** FTP and HTTP sessions — confirming handshake behaviour is protocol-independent.

### 2. FTP — Plain-Text Credential Exposure (HIGH)
- `USER` and `PASS` commands were transmitted in **fully readable plain ASCII text**.
- Observed request: `USER testuser` → `PASS user123`, both visible directly in the Wireshark hex/ASCII pane.
- Server-side log (pyftpdlib) confirmed the same session lifecycle: connection open → USER attempt → auth outcome → disconnect.

### 3. HTTP — Plain-Text Login Form (HIGH)
- A Flask login form masked the password field visually (dots) in the browser UI.
- However, since the form was submitted over **HTTP (not HTTPS)**, the POST body was sent unencrypted.
- Following the TCP stream revealed: `username = testuser`, `password = 1234567` — in full clear text.
- **Conclusion:** client-side field masking is a UI convenience only and provides *no real confidentiality*. Only transport-layer encryption (TLS) protects credentials in transit.

### 4. Traffic & Exfiltration Trend Analysis (LOW)
- All captured conversations were confined strictly to `127.0.0.1 ↔ 127.0.0.1`.
- Traffic volumes were small and consistent with manual test actions.
- No unauthorized or unexpected external data transfer was observed.

### 5. Bonus Observation — Anomalous Login Pattern (MEDIUM)
- Multiple failed FTP login attempts were recorded from the same loopback client across several short-lived sessions.
- This repeated failed-login pattern resembles a **brute-force / credential-stuffing** signature and would typically trigger rate-limiting or account lockout in production.

---

## 📊 Findings Summary Table

| Severity | Area | Observation | Recommendation |
|----------|------|-------------|-----------------|
| **HIGH** | FTP (port 21) | USER/PASS transmitted in fully readable plain text | Replace FTP with SFTP or FTPS for end-to-end encryption |
| **HIGH** | HTTP (port 8080) | POST body with credentials sent unencrypted despite masked password field | Serve login forms exclusively over HTTPS/TLS |
| **MEDIUM** | Authentication logic | Repeated failed FTP logins resembling brute-force pattern | Implement rate-limiting / account lockout |
| **LOW** | Traffic scope | All traffic confined to loopback; no external destinations | No action needed in lab; monitor Conversations in production |

---

## ✅ Recommendations

- Replace FTP with **SFTP/FTPS** to encrypt authentication and file transfer.
- Serve all login forms exclusively over **HTTPS/TLS** — never accept credentials over plain HTTP.
- Do **not** rely on client-side password masking as a security control (cosmetic only).
- Apply **rate-limiting/account lockout** after repeated failed login attempts.
- Regularly review **Conversations/endpoint statistics** in production to flag unexpected external destinations.

---

## 🏁 Conclusion

This exercise demonstrated — using real captured traffic rather than theoretical description — that TCP's three-way handshake is a consistent, protocol-independent foundation underlying both FTP and HTTP connections, and that both unencrypted protocols expose authentication credentials in fully readable plain text. Traffic analysis confirmed no unauthorized exfiltration occurred within this isolated lab scope. The core lesson: **confidentiality of credentials in transit depends entirely on transport-layer encryption (TLS)**, not on application-level conventions like password masking.

---

📎 **Full detailed report:** [`Network_Packet_Analysis_Report.pdf`](./Network_Packet_Analysis_Report.pdf)
