# The CRA Guide: What It Is, and How to Actually Build for It
*A combined guide synthesizing "Aligning Software Security Practices with the EU CRA Requirements" (Tripwire) and "From SBOMs to Trust Boundaries" (OpenNovations) — deduplicated, structured for a tech lead / security champion.*

---

# Part 1 — What the CRA Is

## 1.1 The basics

The **EU Cyber Resilience Act (CRA)** is an EU regulation that makes cybersecurity a **legal precondition for selling products with digital elements** into the EU — covering software and hardware, on-prem, embedded, and SaaS-adjacent products alike.

It's a **risk-based** regulation, not a checklist of specific technologies. It doesn't tell you *which tool* to use — it tells you what *outcomes* you must be able to prove: your product is secure by design, known vulnerabilities are eliminated, you can trace what's inside it, and you can respond to new threats within a defined window, for the entire support period — not just at launch.

## 1.2 Key dates

| Date | What happens |
|---|---|
| **11 September 2026** | Vulnerability & incident reporting obligations begin (Article 14) — 24h early warning, 72h full notification to ENISA for actively exploited vulnerabilities |
| **11 December 2027** | Full CRA obligations apply — SBOM, conformity assessment, CE marking, technical documentation |

*(Verify against the official EU Digital Strategy page — implementing acts can adjust specifics.)*

**Practical implication:** treat September 2026 as your real deadline. The reporting pipeline (SBOM → vulnerability correlation → decision → notification) takes months to build properly; you can't assemble it retroactively in a week once the clock is already running.

## 1.3 The structure that matters to engineering

| CRA element | What it requires | Why it matters to you |
|---|---|---|
| **Annex I, Part 1** (14 requirements) | Secure defaults, access control, encryption, data minimisation, attack-surface reduction, logging, secure erasure, etc. | Your **architecture checklist** |
| **Annex I, Part 2** (8 requirements) | SBOM, timely remediation, coordinated vulnerability disclosure (CVD), security contact point, secure patch distribution | Your **process checklist** |
| **Article 10** | Ongoing, documented, per-product risk assessment | Not a one-off — a recurring review |
| **Article 13 / Annex VII** | Technical documentation, producible on demand for market surveillance authorities | Your evidence must exist *before* someone asks for it |
| **Article 14** | Mandatory vulnerability/incident reporting to ENISA (24h/72h) | Requires a live, working pipeline — not a manual process |

## 1.4 The mindset shift

Both source articles converge on the same reframe, worth internalising and repeating to stakeholders:

> Regulators, customers, and auditors will increasingly ask: *"Can you prove — quickly and consistently — what's in your product, how you built it, and how you handle vulnerabilities across the support period?"*

That proof is not a PDF written once a year. **It's an operating capability** — evidence generated automatically as a byproduct of how you already build and ship software, not a separate compliance project bolted on top.

Three pillars fall out of this, and the rest of this guide is organised around them:

1. **Risk-based, secure-by-design engineering** (what you build)
2. **Supply chain visibility** — SBOM and beyond (what you can see)
3. **Continuous vulnerability handling and reporting readiness** (what you can act on and prove)

---

# Part 2 — The Three Pillars in Practice

## Pillar 1: Risk-based, secure-by-design engineering

**What Article 10 and Annex I Part 1 require:** a documented, regularly-updated risk assessment per product, and architecture that reflects secure-by-design principles (least privilege, minimised attack surface, encryption in transit/at rest, secure defaults).

**How to actually do it:**
- Build a lightweight, repeatable **per-product risk assessment template**: threat landscape → applicable Annex I controls → vulnerability management approach. Re-run it at every major release, not once at launch.
- Push security requirements *into* the PR/CI gate rather than treating them as a separate audit — e.g., require a short security design note on any PR touching auth, crypto, update mechanisms, or trust boundaries.
- Train developers in secure coding and keep records of it — auditors will ask for evidence of training, not just policy documents.

**Tools:**
- Threat modeling: **OWASP Threat Dragon**, **Microsoft Threat Modeling Tool**, or **IriusRisk** (more automation, ties into Jira/backlogs)
- Secure coding: **Secure Code Warrior**, or the free **OWASP Secure Coding Practices Checklist** as a PR template
- Policy-as-code gating: **OPA (Open Policy Agent)** or GitHub required status checks

**A concrete architectural exercise (the "trust boundary" method):** rather than a vague risk statement, draw a single data-flow diagram (DFD) of your system: components, external entities, data stores, entry points, data flows. Annotate where data/identity/execution crosses from trusted to untrusted zones, and flag your **highest-privilege operations** specifically — firmware/OTA updates, remote admin, key provisioning, identity issuance. These get the most regulatory scrutiny because they're capable of causing systemic harm. Run a lightweight **STRIDE** pass at each boundary and prioritise 5–10 abuse cases (not an exhaustive list) with a simple High/Med/Low scheme.

---

## Pillar 2: Supply chain visibility — SBOM and beyond

**What Annex I Part 2 and Article 13 require:** you must be able to trace, audit, and verify every component (proprietary and third-party) in your software, and produce that documentation on request.

### The SBOM is an input, not the outcome
The most important reframe from these articles: **generating an SBOM is not the finish line.** An SBOM that sits in a file share, disconnected from vulnerability data and never revisited, is shelfware. The outcome you actually need is the ability to run your product with **controlled, provable supply chain risk** — SBOM generation is just the first link in that chain.

### Building the pipeline, end to end

**Step 1 — Generate SBOMs automatically, on every build, not just at release.**
Treat the SBOM as a build artifact, like the binary itself.
- **Syft** — best for container images
- **cdxgen** — broadest multi-ecosystem coverage (npm, Python, Java, Go); good default for polyglot stacks
- **ScanCode Toolkit** — deeper license/provenance detection when the above aren't enough
- Ecosystem-specific tools (**cyclonedx-rust-cargo**, **pypispdx**) if you have long-tail language stacks
- **SCANOSS** — catches vendored/copy-pasted code that manifest-based tools miss
- **ESSTRA** — embeds provenance metadata directly into binaries at build time (useful for embedded/compiled products)

Format choice: **CycloneDX 1.6** currently has broader practical tool support than SPDX 3.0 — use it as your default unless you have a specific reason (e.g., a customer requiring SPDX) to generate both natively. Don't convert between formats after the fact; you lose information.

**Step 2 — Validate SBOM quality. Don't skip this.**
SBOMs vary hugely in completeness and correctness. A low-quality SBOM fed into a vulnerability platform means you either miss real issues or drown in noise.
- **OpenChain Telco SBOM Validator** — profile-based pass/fail against a defined quality bar
- **sbomqs** — scores SBOM completeness; set a minimum score as a CI gate

Treat SBOM quality like test coverage: measure it, enforce a minimum, keep improving it.

**Step 3 — Fix identifier chaos before it becomes correlation chaos.**
If component identifiers don't match across tools, your vulnerability correlation is unreliable — you have noise, not visibility. Standardise on **PURLs** (Package URLs) as your canonical identifier.
- **PurlDB** — internalise/serve package metadata keyed by PURL
- **VulnerableCode** — models vulnerability data in a package-first (PURL-based) way
- **ClearlyDefined / OSSelot / LicenseLynx** — pre-curated license/provenance data, avoids re-deriving it yourself
- **FederatedCode** — decentralised metadata sharing, useful for sovereignty/air-gapped contexts

**Step 4 — Correlate SBOMs against vulnerability data, continuously.**
This is where an SBOM becomes an actual security tool rather than an inventory list — connect it to a live vulnerability feed so exposure is assessed automatically and in real time as new CVEs are disclosed, not rediscovered manually per incident.
- **Dependency-Track** — the highest-value single tool here: ingests SBOMs, continuously maps them against CVEs, tracks policy and remediation state per project
- **dep-scan** — good for offline/air-gapped patterns
- **SecObserve** — operational triage cockpit on top of the correlation data

**Step 5 — Record obligations and inventory in a system of record**, not a spreadsheet, so the process is repeatable across products and teams.
- **Eclipse SW360** or **DejaCode** — component/release cataloguing, obligations workflows
- **FOSSology** — license/copyright scanning and clearing workflow

**Step 6 — Store evidence immutably.**
Everything shipped to production must be traceable back to an immutable release record and its evidence pack. If evidence can be silently edited, it isn't evidence — it's a story. Use object storage with versioning/object-lock (e.g., S3 Object Lock) as your evidence vault, separate from your day-to-day systems.

---

## Pillar 3: Continuous vulnerability handling and reporting readiness

**What Annex I Part 2 and Article 14 require:** eliminate known vulnerabilities before shipping, remediate new ones on a defined timeline, run a coordinated vulnerability disclosure (CVD) process, and — for actively exploited vulnerabilities — report to ENISA within 24h (early warning) and 72h (full notification).

### Eliminating known vulnerabilities before release
- Two scanning layers, minimum: **SCA** (your dependencies) and **SAST** (your own code)
- SCA: **Dependency-Track**, **Snyk**, **OWASP Dependency-Check**, GitHub **Dependabot**
- SAST: **Semgrep**, **CodeQL**, **SonarQube**
- Set and document a patch SLA per severity (e.g., critical = 48h triage) — CRA cares about an evidenced process, not just the tooling existing
- Stand up a public **CVD policy** and `security.txt` contact point (`/.well-known/security.txt`) — this is explicitly required, and cheap to do now

### Handling vulnerabilities discovered after shipping
New vulnerabilities *will* appear in components already deployed — the process for handling them needs to be operational, not a periodic scramble.
- **Dependency-Track** continuous monitoring flags newly-disclosed CVEs against your deployed SBOMs automatically
- **SSVC** (Stakeholder-Specific Vulnerability Categorization) — a decision-tree framework to prioritise fixes by actual exploitability and context, rather than raw CVSS score alone
- Log every decision (fix / mitigate / accept) with an **owner and an expiry date** — risk acceptances that never expire become permanent blind spots

### Communicating impact — VEX
Not every disclosed CVE in a component you use actually affects your product. **VEX (Vulnerability Exploitability eXchange)** statements let you say so, formally and machine-readably, rather than customers assuming the worst about every CVE mention.
- Start with **OpenVEX** (`vexctl`, or the **generate-vex** GitHub Action) — lightweight, JSON-LD, PURL-based
- Escalate to **CSAF** as your program matures — it aligns explicitly with Article 14's reporting format, and is the standard to use once you're actually filing reports
- **VEX Generation Toolset** / **vexy** — automate VEX creation at scale once you have many products/fleets, rather than writing them by hand

### Reporting readiness — the part everyone underestimates
Even if you never file a report, build the *discipline* now:
- Run a tabletop exercise: *"A critical CVE just got disclosed in a component we use. Can we identify affected products, generate VEX, and draft a customer/regulator notice within hours?"* If the honest answer is no, that's your real gap — not a missing tool.
- **SSVC** justifies your prioritisation decision in the report; **SecObserve** (or an equivalent case-management layer) links the decision to evidence and a timeline, so the report writes itself from data you already have rather than being reconstructed from memory under time pressure.

---

# Part 3 — Reference Architecture: Putting It All Together

The three pillars above are *what* to do. This section is *how* to structure it as a system, so evidence flows automatically rather than being assembled by hand at each audit.

## 3.1 Five zones (trust boundaries)

Model your system as five zones with explicit trust boundaries between them. Draw this once, for your highest-risk product, with real component names.

| Zone | Trust level | Contains | Rule |
|---|---|---|---|
| **External ecosystem** | Untrusted | Public registries, upstream repos, public CVE feeds | Assume compromise/error until verified |
| **Developer zone** | Semi-trusted | Source control, PRs, local checks | Fast, cheap checks only — don't slow devs down |
| **Build & analysis zone** | Trusted build boundary | CI, SBOM generation, scanning | Evidence gets *created* here |
| **Governance & evidence zone** | High-integrity | Systems of record, evidence vault, audit logs | Immutable — no silent edits, ever |
| **Production zone** | Runtime boundary | Deployment, monitoring, patching | Only evidence-backed artifacts get in |

## 3.2 The gates (enforcement points)

A common failure mode: adopting security tools as a shopping list with no enforcement. The fix is **gates** — pass/fail checkpoints, not best-effort guidance.

| Gate | Purpose | Key tools |
|---|---|---|
| **0 — Dependency onboarding** | Stop bad dependencies before they enter the codebase | Hipcheck, ClearlyDefined, OSSelot, LicenseLynx, SW360/DejaCode |
| **1 — PR/Merge** | Cheap, high-leverage checks at code landing | REUSE, ORT |
| **2 — Build** | Every build produces SBOM + evidence; no SBOM, no promotion | Syft, cdxgen, ScanCode, sbomqs, OpenChain Validator |
| **3 — Release** | Assemble a defensible evidence pack, not just a git tag | ORT, FOSSology, SW360/DejaCode, Dependency-Track, Ampel |
| **4 — Deployment** | Only evidence-backed, approved artifacts reach production | Ampel, Dependency-Track/SecObserve |
| **5 — Continuous risk** | Handle newly-disclosed CVEs in what's already deployed | Dependency-Track, SecObserve, SSVC |
| **6 — Reporting readiness** | Be ready to communicate to customers/regulators (Article 14) | VEX tooling, SSVC, SecObserve |

## 3.3 Attestation and provenance — making gates enforceable
This is what turns "we have a policy" into "unapproved artifacts physically cannot reach production":
- **Ampel** — a policy engine that verifies attestations/provenance and enforces "no evidence, no promotion" at the deployment gate
- Aligns with **in-toto/SLSA**-style provenance — proving the build pipeline itself wasn't tampered with, not just checking the SBOM

## 3.4 A note on "digital sovereignty"
Beyond compliance, there's a strategic reason to favour open standards/tooling over a single closed platform here: you can **self-host, audit the logic behind risk decisions, and retain evidence for years** without depending on a vendor's business continuity or pricing changes. For European vendors specifically, this is increasingly a customer-facing differentiator, not just an internal engineering preference.

## 3.5 CRA-specific evidence overlays (watch, don't build around yet)
Projects like **OCCTET**, **OSCRAT**, **CONFIRMATE**, **CRACoWi**, and **CURIUM** (often EU-funded) aim to sit *above* this engineering toolchain — providing guidance, templates, and evidence automation mapped directly to CRA requirements. They don't replace SBOM generation, vulnerability ops, or governance systems described above, but can reduce the documentation burden once mature. Worth monitoring, especially if you're an SME without dedicated compliance staff — but treat them as an accelerant, not your core pipeline, since they're still early-stage.

---

# Part 4 — Rollout Roadmap

Don't build all seven gates at once. This sequence delivers value fastest and mirrors what both source articles converge on:

1. **Weeks 1–2:** Automate SBOM generation on every build (Syft/cdxgen in CI). Just get it flowing — don't gate on it yet.
2. **Weeks 3–4:** Add SBOM quality scoring (sbomqs) as a visible metric, then a soft gate.
3. **Month 2:** Stand up Dependency-Track, feed it your SBOMs, get continuous CVE correlation live. (This single step delivers the majority of near-term risk reduction.)
4. **Month 2–3:** Pick a system of record (SW360 or DejaCode) and integrate FOSSology for license/obligations clearing.
5. **Month 3–4:** Add SSVC-based triage and SecObserve as your operational cockpit for decisions.
6. **Month 4–5:** Introduce Ampel-style attestation gating at deploy time — this is the point policy becomes enforcement, not advisory.
7. **Month 5–6:** Automate VEX generation; run your first incident-response tabletop against Gate 6.
8. **Ongoing:** Re-run risk assessments per release (Article 10); watch CRA evidence overlays as they mature; target **September 2026** as the date your reporting pipeline must be genuinely operational.

---

# Part 5 — Common Pitfalls

1. **Treating SBOM generation as the finish line.** It's an input; without correlation and governance it's shelfware.
2. **Building a dashboard instead of a capability.** The real test: can you answer "what's affected, who decided, what evidence backs it" in minutes?
3. **Editable evidence.** If it can be silently changed, it isn't evidence.
4. **Exceptions without expiry.** Every risk acceptance needs an owner and a date.
5. **Correlation chaos.** Inconsistent component identifiers (fix with PURLs) make vulnerability mapping unreliable.
6. **No deployment gate.** If nothing can physically block an unapproved artifact from shipping, your "quality control" is advisory, not controlling.
7. **Waiting until 2027.** The reporting pipeline (SBOM → correlation → decision → notification) needs to be live well before the Sept 2026 Article 14 deadline — it can't be assembled retroactively.

---

# Part 6 — First Two Weeks: Concrete Commands

```bash
# 1. Generate an SBOM for a container image
syft <your-image>:<tag> -o cyclonedx-json=sbom.json

# 2. Generate an SBOM for a polyglot repo
cdxgen -o sbom.json .

# 3. Score SBOM quality
sbomqs score sbom.json

# 4. Stand up Dependency-Track (docker) and upload the SBOM
docker compose -f dtrack-docker-compose.yml up -d
# then POST sbom.json to the Dependency-Track API for your project

# 5. Check licensing hygiene
reuse lint
```

Get these five running against one real product this week. Everything else in this guide builds on top of that loop — SBOM generation and Dependency-Track correlation are the foundation the rest of the architecture sits on.

---

---

# Part 7 — Full Tool Reference Table (LinkedIn/OpenNovations article)

Every tool named in the OpenNovations article, in one place: what it does, and where it fits in the gate/zone model from Part 3. Use this as a lookup when deciding what to adopt next in the roadmap (Part 4).

## Dependency onboarding & metadata (Gate 0)

| Tool | What it's about | Where to use it |
|---|---|---|
| **Hipcheck** | Analyses a repo/package for risk signals — maintainer activity, release hygiene, security practices | Run before approving a new direct dependency, at Gate 0 |
| **ClearlyDefined** | Community-curated database of license/provenance/security metadata for open-source packages | Query instead of re-deriving license/provenance data yourself during onboarding |
| **OSSelot** | Curated metadata service, similar accelerant role to ClearlyDefined | Same use case — onboarding-time metadata lookup |
| **LicenseLynx** | Normalises messy, non-standard license identifiers into consistent SPDX-style identifiers | Run on raw scan output before it enters your system of record |

## SBOM generation & component discovery (Gate 2)

| Tool | What it's about | Where to use it |
|---|---|---|
| **Syft** | SBOM generator, strongest for container images (CycloneDX/SPDX output) | CI build step for containerised services |
| **cdxgen** | CycloneDX generator with the broadest multi-ecosystem coverage (npm, Python, Java, Go, etc.) | Default SBOM generator for polyglot repos |
| **ScanCode Toolkit** | Deep license, copyright, and provenance detection engine | When Syft/cdxgen aren't precise enough on licensing/provenance |
| **cyclonedx-rust-cargo** | CycloneDX SBOM generator specific to Rust/Cargo projects | Rust-based services or components |
| **pypispdx** | SPDX SBOM generator for Python/PyPI packages | Python-heavy stacks needing SPDX specifically |
| **SCANOSS** | Component intelligence engine that detects code beyond declared manifests (vendored/copy-pasted code) | Complementary scan alongside manifest-based SBOM tools |
| **ESSTRA** | Embeds provenance metadata directly into binaries at build time | Embedded/compiled products where binary-level provenance matters |

## SBOM quality & validation (Gate 2/3)

| Tool | What it's about | Where to use it |
|---|---|---|
| **OpenChain Telco SBOM Validator** | Validates an SBOM against a defined quality profile (pass/fail) | CI gate immediately after SBOM generation |
| **sbomqs** ("SBOM-QA" in the article) | Scores SBOM completeness/quality; lets you compare generator outputs | Set a minimum score threshold as a CI gate |

## Compliance inventory & obligations management (Gate 3, Governance zone)

| Tool | What it's about | Where to use it |
|---|---|---|
| **Eclipse SW360** | System of record for components, releases, obligations, and approval workflows | Central inventory — record onboarding decisions (Gate 0) and release obligations (Gate 3) |
| **DejaCode** | Alternative system of record (AboutCode ecosystem), strong compliance/SBOM management focus | Same role as SW360 — pick one, not both |
| **FOSSology** | License and copyright scanning with a clearing workflow (legal sign-off) | Release gate — clearing step before shipping |

## Vulnerability & risk operations (Gates 5/6, Production zone)

| Tool | What it's about | Where to use it |
|---|---|---|
| **OWASP dep-scan** | SCA scanner producing vulnerability reports; supports offline/air-gapped use | Continuous scanning, especially in air-gapped environments |
| **BLint** | Binary linter/security checker (from the same project family as dep-scan) | Analysing compiled binaries for security issues |
| **Dependency-Track** | Ingests SBOMs, continuously correlates them against CVE feeds, tracks policy/remediation state per project | Core of Gate 5 — continuous post-deploy monitoring |
| **SecObserve** | Operational triage cockpit — connects ingestion, assessment, reporting, VEX-aware workflows | Sits on top of Dependency-Track for day-to-day triage and decision tracking |

## Licensing hygiene & PR-time policy gates (Gate 1)

| Tool | What it's about | Where to use it |
|---|---|---|
| **REUSE** (tool + spec) | Enforces machine-readable per-file licensing metadata | Required CI check on every PR |
| **ORT** (OSS Review Toolkit) | Orchestrates dependency analysis, policy evaluation, and compliance report generation | PR-time policy checks (Gate 1) and release-time report generation (Gate 3) |

## Identity & metadata commons (cross-cutting — fixes correlation chaos)

| Tool | What it's about | Where to use it |
|---|---|---|
| **PurlDB** | Serves package metadata keyed by PURL (Package URL) | Backing store so all your tools reference components consistently |
| **VulnerableCode** | Models vulnerability data in a package-first (PURL-based) way | Feed into Dependency-Track/SecObserve for reliable CVE-to-component mapping |
| **FederatedCode** | Decentralised sharing of curated software metadata | Sovereignty- or air-gap-sensitive environments where centralised metadata services aren't viable |

## Attestations, provenance & policy gating (Gate 4)

| Tool | What it's about | Where to use it |
|---|---|---|
| **Ampel** | Policy engine that verifies attestations/provenance and enforces "no evidence, no promotion" | Deployment gate (Gate 4) — the point where policy becomes enforcement, not advisory |

## Prioritisation & VEX (Gate 5/6)

| Tool | What it's about | Where to use it |
|---|---|---|
| **SSVC** | Decision-tree framework for prioritising vulnerability response based on stakeholder context, not just CVSS | Triage step in Gate 5, and to justify decisions in Gate 6 reports |
| **VEX Generation Toolset** | Toolset for generating VEX statements at scale across many products/fleets | Automating VEX once you have more than a handful of products |
| **vexy** | VEX generator library/CLI | Lightweight/programmatic VEX generation |
| **OpenVEX generate-vex** | GitHub Action for generating OpenVEX statements | CI-integrated VEX generation, good starting point before CSAF |

## CRA-oriented enabling projects (overlay layer — watch, don't build around yet)

| Project | What it's about | Where to use it |
|---|---|---|
| **OCCTET** | Eclipse Foundation project building CRA compliance tooling/guidance | Monitor for evidence-mapping templates as it matures |
| **OSCRAT** | EU-funded CRA compliance tooling initiative | Same — overlay, not core pipeline |
| **CONFIRMATE** | Fraunhofer AISEC project for CRA compliance testing | Potential future testing/certification support |
| **CRACoWi** (CRA Compliance Wizard) | Digital Europe project guiding CRA compliance steps | Useful for SMEs without dedicated compliance staff |
| **CURIUM** | EU project supporting CRA-related tooling/standards work | Monitor for standards alignment |
| **SECURE** | "Strengthening EU SMEs Cyber Resilience" program | Support program for smaller vendors navigating CRA |
| **CYBERSTAND** | EU initiative on cybersecurity standardisation | Track for evolving standards CRA will reference |

---

## Quick reference: CRA requirement → where it's satisfied

| CRA requirement | Where covered |
|---|---|
| Risk assessment (Article 10) | Part 2, Pillar 1 |
| Secure-by-design (Annex I Part 1) | Part 2, Pillar 1 + trust-boundary exercise |
| SBOM (Annex I Part 2) | Part 2, Pillar 2 + Gate 2/3 |
| Known vulnerability elimination | Part 2, Pillar 3 (SCA/SAST) |
| Technical documentation (Annex VII) | Evidence vault, Part 3.1 |
| Vulnerability reporting (Article 14) | Part 2, Pillar 3 (VEX, SSVC) + Gate 6 |
| Coordinated disclosure / secure updates | Part 2, Pillar 3 (CVD policy) + Gates 4/5 |
