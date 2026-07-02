# **The Developer's Guide to DevSecOps**

> **Our Main Goal:** We are building a secure, automated pipeline for our on-premise Kubernetes data platform. It costs zero dollars in licensing fees and follows industry gold standards like NIST, OWASP, and CIS.

---

## **1. Team Rules & Security Culture (NIST SSDF)**

Great tools don't matter if we don't have the right habits. This pipeline is built around the **NIST Secure Software Development Framework (SSDF)**, which focuses on four key areas: preparing our team, protecting our code, producing secure software, and responding to vulnerabilities.

To make this work, we all need to follow a few basic ground rules:

* **Security Training:** Every engineer must complete an annual secure coding training (like the OWASP Top 10) before getting CI/CD pipeline access.
* **Lock Every Account:** MFA is mandatory on every SCM (GitHub/GitLab) account. Stolen or reused passwords are still the #1 way real breaches start, and this one setting blocks most of them before any tool below ever gets a chance to fire.
* **Don't Skip the Gates:** Bypassing automated checks (like Checkov, Gitleaks, or Kyverno) is strictly forbidden unless you have a documented exception signed by a Security Lead.
* **Treat Security Rules Like Code:** If you want to change a security rule (like a Kyverno policy or Falco alert), it goes through the exact same Pull Request (PR) process as application code. It requires two reviewers and a signed commit.
* **Make It Safe to Report Bugs:** We publish exactly how to reach us privately — SECURITY.md in every repo, security.txt on every live app — so a researcher who finds an issue tells us quietly instead of posting it publicly.
* **Security Champions:** Each service has a designated security champion who:
  * Reviews security-related PRs.
  * Maintains threat models and ASVS checklists.
  * Acts as the first escalation point for security questions.
* **Exception-Management Template:** Every documented exception must include:
  * A clear description.
  * Mitigations in place.
  * A time-bound expiry (e.g., 30 days).
  * An automatic SecObserve ticket.
* **Annual Tabletop Exercise:** Once per year, run a simulated incident (e.g., leaked secret, CVE exploit) and:
  * Validate SecObserve workflows.
  * Verify MFA, backup restore, and Falco alerting.
  * Log lessons learned.

---

Here is the fully revised and integrated **Chapter 2** for your DevSecOps guide.

This version replaces all references to traditional, resource-heavy observability tools with the unified, Rust-based **OpenObserve** framework, and updates the lifecycle to show exactly how **Tetragon** acts as your final runtime containment engine.

---

## **2. The Game Plan: "Shift Left"**

Our core strategy is simple: **catch bugs early**. Instead of waiting for a security audit right before launch, our tools check for issues while you're designing, coding, and building.

We use free, community-driven tools and invest our time in setting them up well, rather than paying for expensive commercial software. We base our cluster rules on the **OWASP Kubernetes Security Testing Guide** and our app rules on the **OWASP ASVS**.

```mermaid
flowchart TD
    classDef uniformSize font-size:28px;

    subgraph Row1 [" "]
        direction LR
        1[1. REQUIREMENTS<br/>OWASP ASVS] --> 2[2. DESIGN<br/>Threat Dragon <br/><br/>] --> 3[3. CODE<br/>SonarLint / Bearer] --> 4[4. PRE-COMMIT<br/>Checkov + Gitleaks] --> 5[5. BUILD<br/>dep-scan --deep] --> 6[6. VALIDATE<br/>OWASP ZAP - DAST]
    end

    Row1 ~~~ Row2

    subgraph Row2 [" "]
        direction LR
        7[7. GOVERN<br/>SecObserve <br/><br/>] --> 8[8. DEPLOY<br/>Kyverno + Cosign] --> 9[9. RUNTIME<br/>Falco + Tetragon] --> 10[10. AUDIT<br/>Kube-bench <br/><br/>] --> 11[11. STREAM HUB<br/>OpenObserve]
    end

    style Row1 fill:none,stroke:none
    style Row2 fill:none,stroke:none

    class 1,2,3,4,5,6,7,8,9,10,11 uniformSize;

    style 1 fill:#1E293B,stroke:#0F172A,stroke-width:2px,color:#fff
    style 2 fill:#4B5563,stroke:#1F2937,stroke-width:2px,color:#fff
    style 3 fill:#3B82F6,stroke:#1D4ED8,stroke-width:2px,color:#fff
    style 4 fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff
    style 5 fill:#F97316,stroke:#C2410C,stroke-width:2px,color:#fff
    style 6 fill:#84CC16,stroke:#4D7C0F,stroke-width:2px,color:#fff
    style 7 fill:#6366F1,stroke:#4338CA,stroke-width:2px,color:#fff
    style 8 fill:#10B981,stroke:#059669,stroke-width:2px,color:#fff
    style 9 fill:#EF4444,stroke:#DC2626,stroke-width:2px,color:#fff
    style 10 fill:#8B5CF6,stroke:#7C3AED,stroke-width:2px,color:#fff
    style 11 fill:#06B6D4,stroke:#0891B2,stroke-width:2px,color:#fff

```

### **The Lifecycle, Step-by-Step**

| **Phase** | **Tool** | **What it Does** | **Why it Matters** |
| --- | --- | --- | --- |
| **1. Requirements** | OWASP ASVS | Creates security checklists. | Sets clear security goals before we even write code. |
| **2. Design** | Threat Dragon | Helps us map out potential threats. | Flushes out bad architectural ideas early. |
| **3. Code** | SonarLint & Bearer CLI | Scans your code right in your IDE. | Acts like a spellchecker for bugs and data leaks. |
| **4. Pre-Commit** | Checkov + Gitleaks | Checks for bad infrastructure configs and accidentally pasted passwords. | Stops silly mistakes before they reach the main codebase. |
| **5. Build** | OWASP dep-scan (`--deep`) | Scans our app libraries AND the container's operating system. | Gives us a single report of all the ingredients we use and any known vulnerabilities. |
| **6. Validate** | OWASP ZAP | Tests the running app in staging (Dynamic Testing). | Catches things code-scanners miss, like broken login flows. |
| **7. Govern** | SecObserve | A central dashboard for all static security alerts. | Keeps build-time and dependency bugs organized so they actually get fixed. |
| **8. Deploy** | Kyverno + Cosign | Checks our Kubernetes setup and verifies image signatures. | Acts as a bouncer, rejecting deployment manifests that break our cluster security rules. |
| **9. Runtime** | **Falco + Tetragon** | Watches system call paths and actively enforces kernel boundaries. | **Falco** handles broad macro-behavior alerts; **Tetragon** provides our final line of defense by forcefully killing malicious process threads in the kernel. |
| **10. Audit** | Kube-bench | Checks our Kubernetes settings against CIS standards. | Makes sure our core platform architecture is locked down tight. |
| **11. Stream Hub** | **OpenObserve** | Consolidated, low-resource database for logs and metrics. | Collects live streams from FluentBit and OTel collectors into cheap Parquet storage using 10x less RAM than traditional setups. |

### **Pipeline Controls**

* **SBOM Generation:** Every CI build must generate a CycloneDX or SPDX SBOM artifact, stored alongside the image.
* **Branch-Protection Rules:** Use Terraform or similar to codify:
* Requires signed commits.
* Requires 2 approvals.
* Requires a match between CODEOWNERS.
* Blocks force-pushes and deletion of protected branches.


* **Pre-Commit Hooks:** Pre-commit hooks are mandatory and enforced via:
* A shared `.pre-commit-config.yaml` in the repo.
* CI validating that hooks ran (e.g., `pre-commit run --all-files`).

---

## **3. Starting Safe: Secure Container Images**

Standard container images (like the ones you pull by default from public hubs) are bloated with extra tools like `curl` or `bash`. Hackers love these tools. We use stripped-down images instead:

* **dhi.io (Free Tier):** Our standard choice for most apps. It has no shells or package managers, and it updates automatically every week.
* **gcr.io/distroless:** Used for compiled languages like Go or Rust that just need bare minimum libraries.

### **Image Comparison Matrix**

| **Feature** | **Standard Public Images** | **Our Standard (dhi.io Free Tier)** | **Future Upgrade (dhi.io Enterprise)** |
| --- | --- | --- | --- |
| **Updates** | Occasional / quarterly | Weekly automated cycles | Daily / on-demand |
| **Vulnerability Footprint** | Vulnerable between releases | Near-zero CVEs | Near-zero + SLAs |
| **Hacker Friendly?** | Highly (ships with `sh`, `apt`, etc.) | Zero (no shell, no utilities) | Zero + FIPS crypto |

### **Proving Our Images are Ours (Signing)**

We don't just trust images; we verify them.

1. **Build:** Cosign digitally signs every image we make.
2. **Deploy:** Kyverno checks the signature. No signature? The code doesn't deploy.
3. **Keys:** We store our signing keys safely in HashiCorp Vault.

### **Additional Image Controls**

* **Pinned Digests:** All image references in manifests must use a digest, not a tag:
  * `image: dhi.io/app@sha256:...`
  * CI fails if a tag is used instead.
* **Base-Image Update Automation:** Automated weekly PRs update base images (dhi.io/distroless) using Renovate or Dependabot.
* **Cosign Key Rotation:** Cosign keys are rotated annually; old keys are revoked in Kyverno policy.

---

## **4. Developer Tools (Catching Bugs Before Commit)**

The cheapest time to fix a bug is while you're writing the code. We have two layers of protection:

1. **On your laptop:** Fast tools that warn you instantly. You can bypass them locally if you're testing, but they exist to save you a headache later.
2. **In CI/CD:** The unskippable automated gate. If the CI/CD pipeline finds a hardcoded password or a critical flaw, it *will* block the merge.

**Repo Rules:** You need signed commits (so we know who wrote the code), a CODEOWNERS file so the right reviewer gets pulled in automatically, a shared .gitignore baseline so secrets and build junk never get committed in the first place, and a second reviewer to approve your PR.

```mermaid
flowchart TD
    A[Bearer CLI] -->|Privacy & data leak scan| E[Developer's IDE]
    B[SonarLint] -->|Code quality & common bugs| E
    C[ESLint + Security Rules] -->|JS / React vulnerability scan| E
    D[FindSecBugs] -->|Java bytecode vulnerability scan| E
    H[Gitleaks] -->|Secret & credential scan| E

    E -->|Fixed before commit| G[Commit-Ready Code]

    style A fill:#3B82F6,stroke:#1D4ED8,stroke-width:2px,color:#fff
    style B fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff
    style C fill:#0ea5e9,stroke:#0284c7,stroke-width:2px,color:#fff
    style D fill:#64748b,stroke:#475569,stroke-width:2px,color:#fff
    style H fill:#F43F5E,stroke:#BE123C,stroke-width:2px,color:#fff
    style E fill:#8B5CF6,stroke:#7C3AED,stroke-width:2px,color:#fff
    style G fill:#10B981,stroke:#059669,stroke-width:2px,color:#fff

```

### **What These Tools Do For You**

| **Tool** | **What It Catches** | **Where It Runs** |
| --- | --- | --- |
| **Gitleaks** | Accidentally pasted API keys and passwords. | Pre-commit on your laptop, and again in CI. |
| **Bearer CLI** | Personal user data (PII) leaking into logs. | On your laptop. |
| **SonarLint** | Common bugs and bad code patterns. | Right in your code editor. |
| **ESLint** | Security bugs specific to JavaScript/React. | IDE and build phase. |
| **FindSecBugs** | Security bugs specific to compiled Java code. | IDE and build phase. |

### **Developer Environment Controls**

* **IDE Config-as-Code:** Share IDE security configs (SonarLint, ESLint rules) as code in `.vscode/` or `.idea/` directories.
* **Secret-Detection in CI as Mandatory:** CI fails if Gitleaks finds secrets; remediation requires:
  * Rotating the secret.
  * Adding it to Vault.
  * Updating the code.

---

## **5. Our Security Toolkit & Where Everything Lives**

Here's how our tools map out across the work environment:

* **Planning Zone:** We use OWASP ASVS and Threat Dragon to figure out what needs to be secured before we start typing.
* **Workstation Zone (Your Laptop):** SonarLint, Bearer CLI, and Gitleaks keep your local environment safe.
* **Pipeline Zone (CI/CD):** Checkov scans our Terraform/Helm files, Gitleaks double-checks for passwords, and OWASP dep-scan (`--deep`) checks our libraries and base image operating system for known flaws.
* **Staging Zone:** OWASP ZAP attacks our staging app to find runtime holes.
* **Edge Zone (Ingress / Load Balancer):** Before traffic ever reaches the cluster, we enforce HTTPS redirect, HSTS, CSP, and TLS 1.3 here, and publish security.txt so anyone who finds a live issue knows how to reach us. This is baseline config, not a scan — set once per app, and it closes the exact holes ZAP would otherwise keep flagging in Staging.
* **Management Hub:** SecObserve acts as our central dashboard for all alerts.
* **Live Kubernetes Zone:** Kyverno enforces rules at the door. Falco watches for weird behavior inside. Cilium/Calico segments our network.
* **Compliance:** OpenSCAP makes sure the underlying servers are up to code.

*Note: Automation bots might suggest fixes on your PRs, but only human engineers can approve them.*

### **Infrastructure Controls**

* **Network Segmentation Diagram:** Add a simple diagram showing:
  * Ingress → DMZ → App tier → Data tier.
  * Where Cilium/Calico policies sit.
* **Explicit Logging of Policy Decisions:** All Kyverno policy changes are:
  * Logged to SecObserve.
  * Reviewed annually for drift against CIS/ASVS.

---

## **6. Proving It: Audit & Evidence**

It's not enough to say we're secure; we have to prove it to auditors and ourselves. Every tool we use generates evidence, and almost all of it feeds directly into our central dashboard, **SecObserve**.

Here is exactly how each tool proves we are doing our jobs:

| **Tool** | **What it Protects** | **The Proof it Creates (Evidence)** |
| --- | --- | --- |
| **OWASP ASVS & Threat Dragon** | Planning & Design | Security checklists and threat models tracked as project tickets before coding begins. |
| **SonarLint, Bearer CLI, ESLint, FindSecBugs** | Local Code Safety | Local scan histories proving that bugs and data leaks were fixed *before* the code was committed. |
| **Kube-bench, Checkov, OpenSCAP** | Configuration & Compliance | Audit-ready reports proving our app configs, Kubernetes settings, and Host OS align with industry standards. |
| **Gitleaks** | Secrets | Pipeline logs and remediation records proving any exposed keys were caught and rotated. |
| **OWASP dep-scan (`--deep`)** | Supply Chain | Software Bill of Materials (SBOMs) and vulnerability reports for both our app libraries and the base operating system. |
| **OWASP ZAP** | Staging / Dynamic Testing | Baseline scan reports proving our live application was attacked and tested before going to production. |
| **Kyverno + Cosign** | Deployment Policies | Logs of rejected deployments (e.g., stopping a container that runs as root or lacks a verified Cosign signature). |
| **Falco + Falcosidekick** | Live Environment | Real-time routed alerts tied to a specific incident ticket and an on-call engineer. |
| **Velero & etcd Snapshots** | Disaster Recovery | Logs of successful "restore drills" logged into SecObserve as Proof of Protection (an untested backup doesn't count as proof). |

**SecObserve** is locked down to protect this evidence. Most of the team can update a finding, but only a couple of trusted leads can delete one — so **no one can quietly make a finding disappear**. Every change needs a short written reason, saved automatically next to the finding, and for sensitive cases we can require a second person to sign off, just like our code reviews. (Logs are even stricter — see Chapter 7 — once written, they can't be changed by anyone, not even an administrator.) The database itself is backed up nightly to independent, tamper-proof storage so we never lose this record.

### **Evidence Management**

* **Automated Evidence Collection:** CI/CD automatically attaches the following to each release artifact in SecObserve:
  * SBOM.
  * dep-scan report.
  * Checkov report.
  * Gitleaks report.
  * ZAP baseline scan.
* **Retention Policy:** SecObserve retains:
  * Critical findings: 7 years.
  * High/Medium: 2 years.
  * Low: 1 year.
* **Read-Only Auditor Role:** Auditors get read-only access to SecObserve with no delete/modify permissions.

### **Our Immediate Action Plan**

Even with these tools, we have a few gaps to close. Here is the roadmap for the platform:

| **Risk** | **The Gap** | **How We're Fixing It** |
| --- | --- | --- |
| 🔴 **Critical** | Secrets stored in plain text. | Rolling out HashiCorp Vault. |
| 🔴 **Critical** | Unverified container images. | Enforcing Cosign image signing + Kyverno verification. |
| 🔴 **Critical** | Flat network (easy for hackers to spread). | Rolling out Cilium/Calico Network Policies to contain breaches. |
| 🟠 **High** | SecObserve needs a backup plan. | Implementing nightly DB backups and tested restore drills. |
| 🟡 **Medium** | Need longer log storage. | Centralizing logs in Loki/ELK for long-term audit history. |

---

## **7. Bouncing Back (Resilience)**

Disaster recovery is a core part of security. Even with every prevention tool in this guide running, things can still go wrong — a bad deploy, a hardware failure, or a real attack. When that happens, the only thing that matters is whether we can get back the data and systems we lost. Here's what we back up, how often, and — just as important — how we actually prove each backup works before we ever need it for real.

| **What** | **Tool** | **How Often** | **How We Prove It Works** |
| --- | --- | --- | --- |
| **Cluster State (etcd)** — the master record of everything running in Kubernetes | Native etcd snapshotting | Daily, stored away from the cluster itself | Every three months, we restore it onto a separate test cluster to confirm it actually works |
| **App Volumes** — the files and storage our applications use | Velero | Daily | Same quarterly restore test, on the same test cluster |
| **Application Data (Databases)** — the actual records our apps create, like customer or transaction data | Native database backup tools (e.g. `pg_basebackup` for Postgres) | Continuous, plus one full backup daily | Every three months, we restore the database to a specific point in time and confirm the data is correct |
| **SecObserve (our security record-keeping)** | Database export to storage outside the cluster | Nightly | See Chapter 6 — restore tests are logged back into SecObserve itself as proof |
| **Network Containment** — stopping an attacker from spreading if they get in | Cilium/Calico Network Policies | Always running | We review network traffic logs after any real incident to confirm the containment held |

**One rule applies to everything above:** our backups are stored somewhere that even our own cluster administrators can't delete or change. If an attacker — or a member of our own team, by mistake — gets full admin access to the cluster, they should still not be able to touch yesterday's backup. That's the difference between a bad day and a permanent loss.

In practice, this means turning on a setting called **WORM (Write Once, Read Many)**, sometimes labeled "object lock," on whatever storage holds the backups. Once switched on for a file, this setting makes it physically impossible for anyone — including someone with full admin rights — to delete or overwrite it until a set time period has passed. It's enforced by the storage system itself, not by Kubernetes permissions, which is exactly why it still protects us even if our Kubernetes access is fully compromised.

### **Resilience Controls**

* **Backup Test Automation:** Quarterly restore tests are:
  * Automated via scripts.
  * Logged in SecObserve with pass/fail status.
* **RPO/RSL Definitions:** Define Recovery Point Objective (RPO) and Recovery Service Level (RSL) per workload:
  * Critical: RPO < 15 minutes, RSL < 4 hours.
  * Non-critical: RPO < 24 hours, RSL < 24 hours.

---

## **8. AI Assistants (Helping, Not Taking Over)**

We use AI agents to help us move faster, but **humans are always in control of the big decisions**. 

This section is the floor, not the ceiling. We anchor it to the **OWASP Top 10 for Agentic Applications (ASI Top 10)** — the OWASP framework for autonomous-agent risk — and its two guiding ideas: **Least-Agency** (give an agent only the autonomy a task actually needs) and **Strong Observability** (always know what it did, why, and under which identity). What follows is enough to start applying agents safely; as our usage grows, it's on the team to keep building on these foundations.

### **The Agent Responsibility Model**

| **Stage** | **Agent Security Role** | **Human Security Role** | **Guardrail** |
| --- | --- | --- | --- |
| **Plan** | Draft security requirements and threat-model checklists. | Approve scope and risk ratings. | Agents cannot accept risk or change severity. |
| **Design** | Suggest security patterns and mitigations. | Own data flows and trust boundaries. | Agents cannot approve architecture or reclassify data. |
| **Code** | Propose secure refactors and fixes for findings in branches. | Own intent and merge decisions. | Agents never push to protected branches or mark checks as passed. |
| **Test / Review** | Generate candidate security tests and first-pass PR comments. | Curate tests and perform final review. | Agents cannot merge tests or PRs; human approval is required. |
| **Deploy** | Enrich deploy checks with risk context. | Decide rollout and rollback. | Agents cannot trigger deploys or rollbacks. |
| **Runtime / Operate** | Correlate alerts and draft runbook steps. | Own incident severity, RCA, and containment. | Agents cannot close incidents or change on-call routing. |
| **Maintain** | Open PRs for low-risk dependency and policy updates. | Approve changes for critical systems. | Auto-fix limited to well-tested repos; humans approve merges. |

Agents are **never** allowed to initiate design changes, alter deployment pipelines, perform incident RCA, or trigger production deploys or rollbacks — those stay human-only, as defined in the Agent Responsibility Model above.

### **Working With Agents Day to Day**

* When tools like Bearer or SonarLint flag an issue, the agent explains it in context and suggests remediation options.
* The agent drafts candidate fixes and targeted tests; humans review, edit, and decide what actually lands in the branch.
* Security-rule changes the agent drafts (Kyverno, Falco, Checkov) still go through the **same two-reviewer, signed-commit process** as any other security rule change (Chapter 1). Agent-authored is not a shortcut.

### **The Autonomous Loop (Low-Blast-Radius Work)**

For background maintenance, a constrained agent loop can handle routine tasks on its own:

```mermaid
flowchart LR
    1[1. FIND<br/>Scanner Flags Issue] --> 2[2. FIX<br/>Agent Applies Change]
    2 --> 3[3. VERIFY<br/>Tests & Policies]
    3 --> 4[4. APPROVE<br/>Human Reviews & Merges]

    style 1 fill:#EF4444,stroke:#DC2626,stroke-width:2px,color:#fff
    style 2 fill:#EC4899,stroke:#BE185D,stroke-width:2px,color:#fff
    style 3 fill:#3B82F6,stroke:#1D4ED8,stroke-width:2px,color:#fff
    style 4 fill:#10B981,stroke:#059669,stroke-width:2px,color:#fff
```

This loop is limited to low-blast-radius tasks: dependency bumps and static-analysis-driven refactors are good fits. Policy synthesis (below) can use this loop too, but because a wrong policy can fail *silently* — blocking good deploys, or worse, leaving a quiet gap — it never skips the two-reviewer rule above.

### **Where Agents Plug Into the Pipeline**

| **Pillar** | **What It Does** | **On-Premise Implementation** |
| --- | --- | --- |
| **Reachability Triage** | Decides whether a CVE that dep-scan found is actually reachable in our code, and produces a VEX verdict. | Agent walks the call graph, checks compatibility, updates the dependency, re-runs tests, and opens a PR with the VEX result attached. |
| **Policy Synthesis** | Turns plain-text NIST/CIS rules into Kyverno or Checkov policy. | Agent drafts validated YAML, links it to test cases, and proposes it via PR — held to the two-reviewer rule above. |
| **Alert Correlation** | Groups related Falco alerts into one picture and drafts a first-pass runbook. | Agent clusters alerts by host/workload and attaches relevant logs, giving the on-call engineer a starting point — it never closes the incident. |

### **Agent Controls**

* **Agent Identity and Audit Trail:** All agent actions:
  * Use a dedicated service account.
  * Are logged with a `service=ai-agent` label.
  * Require human approval for any production change.
* **Agent-Generated Code Review Checklist:** Agents generate a lightweight checklist per PR:
  * Secrets scanned.
  * CVEs triaged.
  * Policy changes reviewed.
  * Humans check off before merge.

---

## **9. Step-by-Step Implementation Checklist**

Here is the exact order we are rolling this out:

* **Phase 1 (Basics):** Enable MFA org-wide and branch protections (2 reviewers, signed commits, CODEOWNERS). Add a shared .gitignore baseline and SECURITY.md. Install SonarLint, Bearer, and Gitleaks locally. Enforce pre-commit hooks and shared IDE configs.
* **Phase 2 (CI/CD):** Make Checkov and Gitleaks required steps in the pipeline. Set up dep-scan and SecObserve. Require SBOM generation in CI.
* **Phase 3 (Testing):** Deploy OWASP ZAP in staging. Start doing Threat Dragon design reviews. Automate ZAP baseline scans on each staging deploy.
* **Phase 4 (Live Cluster):** Turn on Kyverno, Falco, and NetworkPolicies (Cilium/Calico). Configure ingress/load balancer for HTTPS redirect, HSTS, CSP, TLS 1.3, and security.txt. Run OpenSCAP. Document network segmentation and policy review cadence.
* **Phase 5 (Images):** Set up HashiCorp Vault. Enable Cosign image signing. Enforce image digests and key rotation.
* **Phase 6 (Backups):** Finalize and test etcd, Velero, and SecObserve backups. Automate backup restore tests and define RPO/RSL.
* **Phase 7 (AI):** Connect our AI agents for triage and auto-drafting PRs. Add agent identity logging and PR checklists.

---

## **10. Cheat Sheet (Acronyms)**

| Term | What it Means | Context in our Pipeline |
| --- | --- | --- |
| **API** | Application Programming Interface | Where Kyverno intercepts code heading into Kubernetes. |
| **ASVS** | Application Security Verification Standard | Our security checklist for planning. |
| **CI/CD** | Continuous Integration / Deployment | Our automated build/test pipeline. |
| **CIS** | Center for Internet Security | The group that writes the rules Kube-bench checks for. |
| **CSP** | Content Security Policy | An ingress/edge rule that blocks malicious scripts from loading in the browser. |
| **CVE** | Common Vulnerabilities and Exposures | A public list of known hacker exploits. |
| **DAST** | Dynamic Application Security Testing | Attacking a live staging app (done by ZAP). |
| **HSTS** | HTTP Strict Transport Security | An ingress/edge rule forcing browsers to always use HTTPS for our apps. |
| **IaC** | Infrastructure as Code | Config files (Helm/Terraform) scanned by Checkov. |
| **LLM** | Large Language Model | The local AI brains powering our SRE assistants. |
| **MFA** | Multi-Factor Authentication | Required on every SCM (GitHub/GitLab) account. |
| **PR** | Pull Request | The gate where humans approve code and security changes. |
| **RBAC** | Role-Based Access Control | Decides who has admin powers in the cluster. |
| **RPO** | Recovery Point Objective | How much data loss is acceptable in a disaster. |
| **RSL** | Recovery Service Level | How quickly services must be restored after a disaster. |
| **SAST** | Static Application Security Testing | Scanners looking at raw code without running it. |
| **SCA** | Software Composition Analysis | Checking our third-party libraries for flaws. |
| **SBOM** | Software Bill of Materials | The "ingredients list" of our apps. |
| **VEX** | Vulnerability Exploitability eXchange | Tells us if a known bug is *actually* dangerous to us. |
