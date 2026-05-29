# Lab 2 — Network segmentation audit test results
## Prepared by: Dan Isaaka | CyberByDan
## Date: 2026-05-28
## Scope: Docker-simulated network segmentation across DMZ and LAN zones

---

## Context — why this test matters

In any organisation running internet-facing services, the biggest risk
is not that the front door gets kicked in. It is that once someone
gets through the front door, they can walk freely into every room in
the building.

Network segmentation is the control that puts locked doors between
those rooms. This audit test verifies that those doors are actually
locked — not just drawn on an architecture diagram.

In East Africa, many organisations run flat networks where every
system can talk to every other system freely. A compromised web
server in that environment means an attacker has immediate access
to payroll systems, customer databases, and internal applications.
Segmentation is the difference between a contained incident and a
catastrophic breach.

---

## Test 1 — DMZ to LAN name resolution

### What we tested
Can the public-facing web server in the DMZ see or contact any
service in the internal LAN zone by name?

### Command run
    docker exec dmz-webserver ping -c 3 lan-internal

### Expected result
Fail — the DMZ should have zero visibility into LAN service names.
If this test passes it means the network boundary is not enforcing
DNS isolation and an attacker in the DMZ could enumerate internal
services by name.

### Actual result
    ping: bad address 'lan-internal'

### Finding: PASS
The DMZ container cannot resolve the name of any LAN service.
DNS isolation is confirmed across the network boundary.

### Business impact if this test had FAILED
An attacker who compromises the web server could discover the names
and addresses of internal systems — databases, file servers, HR
systems — and begin targeting them directly. In a bank or telco
this could mean customer account data exposure. In a hospital this
could mean patient records. In a government agency this could mean
classified information. The financial and reputational damage of
that exposure would far exceed the cost of implementing segmentation.

### CISA connection
Domain 5 — Protection of Information Assets
Preventive control — network isolation prevents unauthorised
lateral movement. An auditor does not just review firewall rules.
An auditor tests that the rules enforce in practice.

---

## Test 2 — DMZ to LAN direct IP access

### What we tested
Even when name resolution fails, can the DMZ reach the LAN by
typing the IP address directly? This is a more sophisticated
attack vector — an attacker who already knows the internal IP
range does not need DNS.

### Command run
    docker exec dmz-webserver ping -c 3 172.21.0.2

### Expected result
Fail — packets should not cross the network boundary even by
direct IP address. If this test passes the segmentation is
superficial — it only blocks name lookups, not actual traffic.

### Actual result
    3 packets transmitted, 0 packets received, 100% packet loss

### Finding: PASS
The DMZ cannot reach the LAN by direct IP address. Network layer
isolation is confirmed. The segmentation is enforced at the packet
level, not just the DNS level.

### Business impact if this test had FAILED
A sophisticated attacker who had already bypassed DNS isolation
could still move laterally by scanning known IP ranges and
connecting directly. This is how many real-world breaches escalate
from a single compromised server to a full network compromise.
The 2013 Target breach followed exactly this pattern — attackers
moved from a third-party HVAC vendor's access point across a flat
network to reach the payment systems. Segmentation would have
contained that movement.

### CISA connection
Domain 5 — Protection of Information Assets
Defence in depth — multiple layers of control must each
independently enforce the security boundary. One layer failing
does not mean the attack succeeds, but one layer is never enough.

---

## Test 3 — LAN to DMZ access (reverse direction)

### What we tested
Can the internal LAN zone reach the DMZ web server? In a standard
architecture the LAN should be able to initiate connections to the
DMZ for legitimate internal management. This test determines whether
the isolation is one-way or bidirectional.

### Command run
    docker exec lan-internal ping -c 3 172.20.0.2

### Expected result
Pass — LAN should reach DMZ for legitimate internal requests.

### Actual result
    3 packets transmitted, 0 packets received, 100% packet loss

### Finding: UNEXPECTED — bidirectional isolation confirmed
Both zones are completely isolated from each other. Neither can
initiate connections to the other in any direction.

### Is this a finding or a strength?
This depends on the security policy of the organisation.

Stricter posture — bidirectional isolation is correct when the
DMZ serves only external users and internal staff never need to
access it directly. Maximum containment. If the DMZ is compromised
no traffic flows in either direction.

Standard posture — one-way isolation is correct when internal
administrators need to manage DMZ services, push updates, or
monitor application logs from the LAN. In this case bidirectional
blocking would be a misconfiguration preventing legitimate work.

For our homelab the stricter posture is acceptable. In a real
engagement the auditor would compare the actual traffic flow
against the documented security policy and flag any deviation
as a finding — in either direction.

### Business impact
Bidirectional isolation means internal teams cannot directly
manage or update the DMZ web server from the LAN. In a real
organisation this could mean software updates are delayed,
security patches cannot be applied quickly, and incident
response is slower because responders cannot reach the affected
system from their workstations.

This is the classic security versus operability trade-off.
Maximum security can introduce operational risk if it prevents
legitimate management activity.

### CISA connection
Domain 5 — Protection of Information Assets
Domain 4 — IS Operations and Business Resilience
Security controls must be evaluated against both their protective
value and their operational impact. A control that is too
restrictive can itself become a business risk. The auditor
documents both sides.

### CISA exam trap
ISACA will give you a scenario where a very strict security
control is causing operational problems and ask what the auditor
should recommend. The answer is never to remove the control
entirely. It is to review the control against the security policy
and recommend a balanced adjustment that maintains protection
while enabling legitimate operations.

---

## Overall audit conclusion

The DMZ and LAN zones are properly isolated at both the DNS and
network packet level. An attacker who compromises the DMZ web
server cannot use it as a stepping stone to reach internal systems.

This satisfies the core segmentation control objective:
contain the blast radius of a DMZ compromise to the DMZ zone only.

### What this means in plain English
If someone breaks into the shop front, they cannot get into the
back office. The locked door between them is working.

---

## Lessons learned

1. Segmentation must be tested at multiple layers — DNS isolation
   alone is not enough. An attacker who knows IP addresses does not
   need DNS. Both tests must pass independently.

2. A control that exists in a design document but has never been
   tested is not a control. It is an assumption. Assumptions are
   audit findings.

3. The business impact of failed segmentation is always larger than
   the technical description suggests. An IS auditor communicates
   risk in business language — data exposure, regulatory penalty,
   reputational damage — not just in technical terms.

4. Docker network isolation enforces segmentation at the kernel
   level using Linux network namespaces. This is the same
   underlying technology used in enterprise container platforms
   like Kubernetes. The concept scales directly to real
   infrastructure.

---

## Finding 01 — DMZ has network visibility into management zone

### Severity: High

### What was discovered
During Test 4, the DMZ web server was found to have network
connectivity to Uptime Kuma in the management zone. The DMZ
container successfully pinged the monitoring tool by name and
received responses with 0% packet loss.

### How it was discovered
Auditor ran a reverse connectivity test from inside the DMZ
container after connecting Uptime Kuma to the dmz-network
for monitoring purposes.

    docker exec dmz-webserver ping -c 3 uptime-kuma
    3 packets transmitted, 3 packets received, 0% packet loss

### Root cause
When Uptime Kuma was connected to dmz-network to enable
monitoring of the DMZ web server, Docker's bridge networking
allowed bidirectional traffic on that shared network segment.
The monitoring tool became reachable from the zone it was
monitoring — an unintended consequence of a well-intentioned
configuration change.

This is a classic example of a compensating control introducing
a new attack path. The monitoring was correctly implemented but
the network architecture did not enforce one-way visibility.

### Business impact
If an attacker compromises the DMZ web server they can now reach
the monitoring and alerting infrastructure directly. This creates
two serious consequences:

First — the attacker can disable or tamper with Uptime Kuma
alerts before triggering further activity, blinding the security
team to the ongoing attack. This is called defence evasion and
it is one of the most dangerous capabilities an attacker can have.

Second — the attacker can use Uptime Kuma as a pivot point to
reach other services on the management network including Grafana,
Loki, and the audit trail. Tampering with audit logs is a
significant compliance and legal risk — in a regulated environment
such as banking or healthcare this could result in regulatory
penalties and invalidate forensic evidence needed for incident
response.

In plain English — an attacker who gets into the shop front can
now reach the CCTV room and turn off the cameras before robbing
the back office.

### Compensating controls currently in place
- Uptime Kuma dashboard is password protected
- Uptime Kuma does not expose sensitive business data
- Grafana audit trail is on a separate network segment
- All access to monitoring tools is logged

### Residual risk
Medium — compensating controls reduce but do not eliminate
the risk. The network path still exists and could be exploited
by a sophisticated attacker.

### Recommendation
Implement a dedicated monitoring agent architecture where a
lightweight agent inside the DMZ pushes metrics outward to
the management zone rather than the management zone pulling
inward. This enforces one-way data flow at the network level.

This will be implemented in Lab 2 Extension — one-way
monitoring architecture using a push-based agent pattern.

### CISA connections
Domain 5 — Protection of Information Assets
Network segmentation must be verified after every configuration
change. A change that is correct in isolation can introduce new
attack paths when combined with existing architecture.

Domain 1 — Information Systems Auditing
Audit trail integrity is a fundamental requirement. Any attack
path that could allow tampering with logs or disabling of alerts
is a High finding regardless of the likelihood of exploitation.

Domain 4 — IS Operations and Business Resilience
Monitoring infrastructure is itself a critical asset. Its
availability and integrity must be protected with the same
rigour as the systems it monitors.

### CISA exam trap
ISACA will give you a scenario where a security control — such
as monitoring — inadvertently creates a new vulnerability. The
question will ask what the auditor should do first. The answer
is always to document and report the finding to management with
a risk rating and recommendation. The auditor does not fix the
problem — the auditor surfaces it so management can make an
informed decision about remediation priority.

### Status: Open
Planned remediation: Lab 2 Extension — push-based monitoring
architecture
Target date: Next lab sprint