# Homelab — lessons learned

## Day 1 — Log audit trail

### What we built
A centralised log aggregation stack using Loki, Promtail, and Grafana
running in Docker. Three dashboard panels monitoring auth events, failed
logins, and sudo privilege escalation.

### What we did
- Deployed Loki (log storage), Promtail (log collector), Grafana (dashboard)
  and Uptime Kuma (availability monitor) via docker-compose
- Connected Loki as a Grafana data source
- Built an audit trail dashboard with three LogQL queries
- Simulated failed login attempts and confirmed they appeared in the dashboard

### What we found
- SSH was not installed on the machine — discovered during failed login
  simulation. This is itself an audit finding: you cannot test a control
  that does not exist.
- After installing openssh-server, failed logins were successfully captured
  and visible in Grafana within 30 seconds.

### CISA connections
- **Domain 1:** Audit evidence must be sufficient, reliable, and relevant.
  We verified all three — the logs are timestamped, tamper-evident, and
  directly tied to access events.
- **Domain 5:** Failed login attempts are a detective control indicator.
  Repeated failures against the same account = potential brute force attack.
- **Exam watch:** CISA distinguishes between a log existing and a log being
  reviewed. A log nobody reads is not a control — it is just storage.

### Lessons learned
1. Always verify that the service you are auditing actually exists before
   testing controls around it. SSH was absent — in a real audit this would
   be a finding: "Control cannot be evidenced as service is not running."
2. Container networking uses service names not localhost — Grafana reaches
   Loki via http://loki:3100 not http://localhost:3100.
3. LogQL syntax: {job="auth"} pulls all auth logs. Adding |= "Failed password"
   filters to failed logins only. Simple but powerful for audit queries.

### Sudo capture — privilege escalation evidence

Ran `sudo ls` and confirmed the event appeared in Grafana within 30 seconds.

Log line captured:
pam_unix(sudo:session): session closed for user root

What this tells an auditor:
- Who: dan-isaaka
- What: escalated to root privileges
- When: 2026-05-27 23:19:10 EAT
- Outcome: session closed cleanly (no lingering root session)

CISA exam watch: Privileged access must be logged, monitored, and reviewed
regularly. The existence of sudo logs is a preventive control design — but
only becomes a detective control when someone actually reviews them. We just
did that review.

## Controls remediated — Day 1 evening

### Fix 1 — Grafana password strengthened
Changed Grafana admin password from weak default (homelab123) to a strong
password. Verified by logging out and back in successfully.

CISA connection: Default credentials are a critical finding in any audit.
An auditor who finds a system running on default credentials documents it
as a High risk — the control exists (authentication) but is ineffective
(trivially bypassed). Changing the password moves this from High to Low.

### Fix 2 — SSH restricted to localhost
Added ListenAddress 127.0.0.1 to sshd_config so SSH only accepts
connections from the local machine. Restarted SSH and confirmed active.

CISA connection: Attack surface reduction is a preventive control. Every
open door that is not needed is an unnecessary risk. Restricting SSH to
localhost means even if someone finds port 22, they cannot reach it from
outside the machine.

### Auditor mindset note
Both fixes took under 2 minutes. The risk was not in the complexity of
the fix — it was in not knowing the gap existed. That is exactly why
audits exist. The auditor's value is in finding the gap, not fixing it.

## Day 2 — Backup and restore drill

### What we built
A Restic backup repository storing snapshots of all homelab config files
and documentation. First snapshot taken at 11:53 EAT on 2026-05-28.

### The restore drill
- Deleted lessons-learned.md to simulate data loss
- Started RTO timer at 11:57
- Ran restore command — file recovered in under 1 minute
- Confirmed file present in ~/homelab/docs/

### Metrics documented
- RPO: 24 hours (how often we back up — we will automate this next)
- RTO: under 1 minute (how long recovery actually took)
- Backup size: 1.082 MiB compressed to 28.713 KiB
- Snapshot ID: 10e1205a

### Errors noted
Four permission denied errors on Uptime Kuma database files. These files
are locked by the running container during backup. Not a critical gap for
our lab but in a production environment this would be a finding — live
database files require a different backup strategy such as a database dump
before backup runs.

### CISA connections
- A backup that has never been tested is not a control — it is hope.
  ISACA expects auditors to verify that backups are tested regularly,
  not just scheduled.
- RTO is a business decision not a technical one. The business decides
  how long it can survive without a system. IT then builds to meet that
  target. Our RTO target was met comfortably.
- The permission denied errors are a real audit finding — documented,
  rated Low for now, added to risk register as R-11.

### Lesson learned
Always run a restore drill before declaring a backup system operational.
The backup command completing successfully does not mean recovery works.
Only a successful restore proves the backup has value.

## Day 2 — User Access Review

### What we did
Ran a full user access review (UAR) on the Ubuntu machine covering:
- Human account enumeration
- Sudo privilege review
- Never-logged-in account investigation

### What we found

**Human accounts:** 1
- dan-isaaka (UID 1000) — active, has home directory, uses bash
- Status: Justified — this is the primary operator account

**Sudo access:** 1
- dan-isaaka — sole administrator
- Status: Appropriate for single-user homelab environment

**Never logged in accounts:** 20+ system accounts
- Root, daemon, www-data, backup, and others
- Status: All are OS system accounts, not human accounts
- These are expected and correct — they exist to run services,
  not for interactive login

**Finding: CLEAN — no orphan accounts, no unexpected privileges**

### CISA exam trap learned
A list of accounts that have never logged in is NOT automatically a
finding. The auditor must distinguish between system accounts and human
accounts before concluding. Raw data without context leads to wrong
conclusions — and wrong audit findings damage the auditor's credibility.

### Key UAR concepts for exam
- Orphan account: a human account whose owner no longer works at the
  organisation — this is a real finding
- Ghost account: an account that exists but has no business justification
- Privilege creep: an account that has accumulated access beyond what
  the role requires over time
- Least privilege: every account should have only the access it needs,
  nothing more

  ## Lab 2 — Key lesson: controls can create vulnerabilities

The most important finding in Lab 2 was not planned. When we
connected Uptime Kuma to the DMZ network to enable monitoring,
we accidentally gave the DMZ visibility into the management zone.

A well-intentioned change introduced a new attack path.

This happens constantly in real organisations. A firewall rule
is opened for a legitimate business reason and nobody reviews
whether it creates unintended connectivity elsewhere. An auditor
who only reviews the original design misses this entirely.

The lesson: always test after every configuration change, not
just at the end of the project. Security architecture is not
a one-time activity — it is a continuous review process.

In CISA language this is called a compensating control gap —
a control put in place to address one risk inadvertently
increases exposure to another risk.