# Homelab 02 — Network Segmentation and DMZ Architecture
### A hands-on CISA study lab by @CyberByDan

---

## Who this is for

If you want to understand what network segmentation actually looks
like in practice — not just in a textbook diagram — this lab is
for you.

I built this on the same single laptop as Homelab 01, using Docker
networks to simulate a three-zone architecture: DMZ, LAN, and a
management zone. No VMs, no cloud spend, no special hardware.

The most valuable outcome was not what worked. It was what went
wrong — a well-intentioned monitoring configuration accidentally
created a security gap that a real attacker could exploit.

That finding is fully documented here.

---

## What I built

A simulated enterprise network architecture using Docker networks
to enforce zone isolation between public-facing services, internal
systems, and monitoring infrastructure.

| Zone | Purpose | Services | CISA Domain |
|---|---|---|---|
| DMZ | Public facing | Nginx web server | Domain 5 |
| LAN | Internal network | Internal apps, databases | Domain 5 |
| Management | Monitoring and audit | Grafana, Loki, Uptime Kuma | Domain 1, 4 |

---

## CISA domains covered

Domain 5 — Protection of Information Assets
Designed and tested network segmentation controls. Verified
isolation between DMZ and LAN at both DNS and IP packet level.
Documented a real security gap created by monitoring configuration.

Domain 1 — Information Systems Auditing
Conducted formal audit tests with documented evidence, findings,
risk ratings, and recommendations. Applied the auditor mindset —
test the control, not just review the design.

Domain 4 — IS Operations and Business Resilience
Evaluated the operational impact of strict segmentation controls
and the risk of monitoring infrastructure being reachable from
a compromised zone.

---

## Prerequisites

Homelab 01 completed — Loki, Grafana, Promtail, and Uptime Kuma
already running. If you have not done Homelab 01 yet, start there.

    https://github.com/cyberbydan/homelab-01-audit-trail

Docker and Docker Compose installed on Ubuntu or Debian Linux.

---

## Network architecture

Three Docker networks simulating enterprise network zones:

    dmz-network      Public facing zone
    lan-network      Trusted internal zone
    mgmt-network     Monitoring and audit zone

The DMZ serves external traffic. The LAN holds internal systems.
The management zone watches both but should be reachable by neither.

---

## Audit tests conducted

Test 1 — DMZ to LAN name resolution
Result: PASS — DMZ cannot resolve LAN service names
Evidence: ping: bad address 'lan-internal'

Test 2 — DMZ to LAN direct IP access
Result: PASS — 100% packet loss across network boundary
Evidence: 3 packets transmitted, 0 received

Test 3 — LAN to DMZ access
Result: UNEXPECTED — bidirectional isolation confirmed
Evidence: 100% packet loss in both directions
Note: Stricter than intended but acceptable for this lab

Test 4 — DMZ to management zone
Result: FINDING — DMZ can reach monitoring infrastructure
Evidence: 0% packet loss to Uptime Kuma
Severity: High

Full test results and evidence in docs/audit-test-results.md

---

## Key finding — monitoring created an attack path

When Uptime Kuma was connected to the DMZ network to monitor
the web server, Docker bridge networking allowed the DMZ to
reach back into the management zone.

A well-intentioned configuration change introduced a new
attack path. This is one of the most common real-world audit
findings — a compensating control that creates a new vulnerability.

In plain English: we opened a small window so the security
camera could see the shop front. The shop front could now
also see through the window into the camera room.

Remediation is planned for Homelab 02 Extension using a
push-based monitoring agent architecture.

---

## Lessons learned

The most important lesson from this lab:

Always test after every configuration change, not just at the
end of the project. Security architecture is a continuous review
process, not a one-time activity.

A control that is correct in isolation can introduce new attack
paths when combined with existing architecture. The auditor's
job is to find those combinations before an attacker does.

Full lessons learned in docs/lessons-learned.md

---

## What is next

Homelab 02 Extension — push-based monitoring architecture
Homelab 03 — Identity and access management with Authentik SSO
Homelab 04 — Vulnerability assessment using OpenVAS

---

## About CyberByDan

IT audit and cybersecurity professional based in Nairobi, Kenya,
preparing for the CISA certification. Building these labs to make
IS audit concepts accessible for East African professionals
breaking into the field.

Follow the journey:
TikTok: @CyberByDan
Instagram: @CyberByDan
LinkedIn: @CyberByDan

---

Built with curiosity, documented with intent.
