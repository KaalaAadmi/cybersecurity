# Architecting an Enterprise Cloud SOC & SIEM Lab: From Vulnerability to Detection

Coming from a software engineering background, I am used to building infrastructure and applications. However, I wanted to understand how to actively defend them. Instead of just following a cookie-cutter security tutorial, I decided to build an enterprise-grade environment, break it, and engineer the detection mechanisms from scratch.

This project documents the entire lifecycle of a cyberattack: architecting the vulnerable infrastructure, simulating the threat, engineering the log ingestion pipelines, and finally, hunting the telemetry as a Blue Team analyst.

### 🎯 Project Goal

My primary objective was to accelerate my transition into cybersecurity by bridging the gap between software development and security operations. I set out to build a fully functional Security Operations Center (SOC) and SIEM lab natively in Microsoft Azure, complete with strict network segmentation, custom data normalization, and live threat emulation.

### 🧠 Key Learnings

- **Security is Systems Engineering:** You cannot defend what you don't understand. Building the multi-region Virtual Networks and Network Security Groups (NSGs) reinforced how critical foundational cloud architecture is to an organization's security posture.
- **Raw Logs are Messy:** Threat actors don't send polite, clear-text payloads. I learned how to engineer custom ingest pipelines in Elasticsearch to transform obfuscated, URL-encoded attacker traffic into actionable Threat Intelligence.
- **Enterprise OPSEC:** I gained hands-on experience implementing a Bastion Host (Jump-Box) architecture, utilizing SSH Agent Forwarding and local port tunneling to access internal analytics dashboards without exposing them to the public internet.

### 🎥 The Highlight Reel

_If you are short on time, I highly recommend watching this 4-minute video breakdown of the architecture and the final security analysis._

> **[Insert Unlisted YouTube Link Here]**_Note: Attacker Source IPs and specific environment identifiers have been redacted from the video for operational security._

## 🏗️ The Architecture: Building the Vault

To mirror strict enterprise compliance, I couldn't just throw virtual machines onto the public internet. The infrastructure required segmentation and secure access controls.

- **Multi-Region Deployment:** To optimize resource allocation and navigate cloud compute quotas, I split the lab across two Azure regions. The target web server sits in Switzerland-North, while the SIEM and management gateways live in Norway-East.
- **The Bastion Host:** The Elastic SIEM is completely isolated from the public internet. Accessing the Kibana analytics dashboard requires an authorized analyst to establish a secure, encrypted SSH tunnel (`L 5601`) from their local terminal, routing directly through a dedicated Jump-Box.
- **Containerization:** Both the vulnerable web app (DVWA) and the Elastic Stack were containerized using Docker for absolute environmental isolation and rapid deployment.

> _**(Insert the [Draw.io](http://Draw.io) Architecture Diagram here)**_

## ⚙️ Data Engineering: When Default Tools Fail

I initially configured Filebeat to ship the Apache logs from the target web server to Elasticsearch. However, the default Filebeat parsing modules immediately crashed trying to read the Docker-mounted volume logs.

Just like debugging a complex backend issue, I had to drop down a level. I ripped out the fragile default modules and configured Filebeat as a raw `filestream` to force the data through.

Because raw logs are unreadable and URL-encoded (making attackers hard to find), I built a custom **Ingest Pipeline** directly inside Elasticsearch. Using the Kibana Dev Tools API, I deployed a URL-decoder processor to catch the raw traffic and translate obfuscated attacker payloads back into clear text _as they arrived in the database_.

## ⚔️ Threat Emulation: The Red Team Perspective

With the SIEM locked down and the custom ingest pipeline humming, it was time to generate malicious traffic.

I navigated to the Damn Vulnerable Web App (DVWA) hosted on the target VM. Moving to the SQL Injection module, I attempted to bypass the login authentication by injecting a classic Boolean payload directly into the User ID parameter:

```sql
%' or '1'='1
```

The database immediately dumped the internal user table to the frontend GUI. The vulnerability was successfully exploited.

## 🛡️ The Hunt: The Blue Team Perspective

Switching hats to the SOC Analyst role, I dropped into my SSH-tunneled Kibana dashboard to track the intrusion.

Because attackers rarely send clear-text payloads, a standard string search usually fails. The web server actually recorded the attack as `%25%27+or+%271%27%3D%271`. However, thanks to the custom URL-decoding pipeline I engineered earlier, the SIEM had already normalized the data.

I queried the Discover tab directly for `"1'='1"` and immediately surfaced the raw telemetry.

> _**(Insert Screenshot of the expanded Kibana log here, highlighting the message field)**_

### Analyzing the Raw Artifacts

By investigating the raw `message` string, I could extract the exact indicators of compromise (IoCs): `[REDACTED] - - [22/May/2026:00:58:03 0000] "GET /vulnerabilities/sqli/?id=%' or '1'='1&Submit=Submit HTTP/1.1" 200...`

- **The Payload:** The URL-decoder successfully revealed the clear-text `id=%' or '1'='1` injection in the URI.
- **The Attacker IP:** The source IP (`[REDACTED]`) is clearly identifiable at the very beginning of the string.
- **The Status Code:** The `200` following the HTTP/1.1 declaration is the smoking gun. It confirms to the analyst that the web server did not block the request; it processed the malicious payload successfully.
