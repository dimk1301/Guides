
---

# Threat Modeling for Software Applications with OWASP Threat Dragon and PyTM

---

## Table of Contents

[Executive Summary](#executive-summary)
1. [1. Fundamentals of Threat Modeling](#1-fundamentals-of-threat-modeling)
   - [1.1 Core Goals](#11-core-goals)
   - [1.2 Common Methodologies](#12-common-methodologies)
   - [1.3 Data Flow Diagrams (DFDs) – Core Elements and Notation](#13-data-flow-diagrams-dfds--core-elements-and-notation)
2. [2. Threat Modeling Process for Microservices and Legacy Systems](#2-threat-modeling-process-for-microservices-and-legacy-systems)
   - [2.1 Define Scope and System Representation](#21-define-scope-and-system-representation)
   - [2.2 Identify Assets and Trust Boundaries](#22-identify-assets-and-trust-boundaries)
   - [2.3 Enumerate Threats](#23-enumerate-threats)
   - [2.4 Analyze Risk and Prioritize](#24-analyze-risk-and-prioritize)
   - [2.5 Define and Track Mitigations](#25-define-and-track-mitigations)
   - [2.6 Integrate Into Lifecycle and CI/CD](#26-integrate-into-lifecycle-and-cicd)
3. [3. Focused Tooling: OWASP Threat Dragon & OWASP PyTM](#3-focused-tooling-owasp-threat-dragon--owasp-pytm)
   - [3.1 OWASP Threat Dragon – Visual, Collaborative Threat Modeling](#31-owasp-threat-dragon--visual-collaborative-threat-modeling)
     - [3.1.1 Creating a DFD in Threat Dragon](#311-creating-a-dfd-in-threat-dragon)
   - [3.2 OWASP PyTM – Programmable Threat Modeling as Code](#32-owasp-pytm--programmable-threat-modeling-as-code)
     - [3.2.1 Creating a DFD in PyTM (Model as Code)](#321-creating-a-dfd-in-pytm-model-as-code)
   - [3.3 Choosing Between the Two (or Combining)](#33-choosing-between-the-two-or-combining)
4. [4. Automation, LLM, and AI‑Agent Integration](#4-automation-llm-and-ai-agent-integration)
   - [4.1 Why PyTM Excels for AI Agents and MCP Compatibility](#41-why-pytm-excels-for-ai-agents-and-mcp-compatibility)
   - [4.2 Using Threat Dragon with AI Assistance](#42-using-threat-dragon-with-ai-assistance)
   - [4.3 AI Integration Patterns for PyTM](#43-ai-integration-patterns-for-pytm)
5. [5. Practical Recommendations](#5-practical-recommendations)
   - [5.1 Recommended Workflow](#51-recommended-workflow)
   - [5.2 Tool Usage by Phase](#52-tool-usage-by-phase)
   - [5.3 AI‑Agent and MCP Integration with PyTM](#53-ai-agent-and-mcp-integration-with-pytm)
6. [6. Summary](#6-summary)

---

## Executive Summary

Threat modeling is essential for identifying and mitigating security risks early in the software lifecycle. For organisations with a mix of microservices and legacy systems, two open‑source OWASP tools stand out:

- **OWASP Threat Dragon** – a user‑friendly, diagram‑based web/desktop application that supports STRIDE methodology and collaborative threat documentation.
- **OWASP PyTM** – a Python‑centric “threat modeling as code” framework that automates threat generation, diagram production, and report creation, making it ideal for CI/CD pipelines and AI‑driven automation.

This guide provides a practical, step‑by‑step methodology for threat modeling in mixed environments and shows how to use both tools effectively. PyTM’s CLI, JSON exports, and Python scriptability make it particularly well‑suited for integration with Large Language Models (LLMs) and AI agents via Model Context Protocol (MCP) or similar frameworks, enabling generation and evolution of threat models from prose or infrastructure‑as‑code.

---

## 1. Fundamentals of Threat Modeling

### 1.1 Core Goals

Threat modeling answers four key questions (Shostack):

1. **What are we building?** – Understand architecture and data flows.
2. **What can go wrong?** – Enumerate threats against assets and interactions.
3. **What are we going to do about it?** – Define mitigations or controls.
4. **Did we do a good job?** – Validate coverage and residual risk.

For microservices and legacy systems, threat modeling also highlights where to introduce compensating controls or refactor to reduce risk.

### 1.2 Common Methodologies

| Methodology | Focus Area | Description |
|-------------|------------|-------------|
| **STRIDE** | Software‑centric threats | Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege – both tools support STRIDE natively. |
| **LINDDUN** | Privacy threats | Linkability, Identifiability, Non‑repudiation, Detectability, Disclosure, Unawareness, Non‑compliance – can be applied with PyTM’s custom threat database. |
| **PASTA** | Risk‑driven | Aligns threats with business impact – useful for high‑level prioritisation. |

**Recommendation:** Use STRIDE on data‑flow diagrams as a baseline; add a light privacy pass using PyTM’s extensible data classification.

---

### 1.3 Data Flow Diagrams (DFDs) – Core Elements and Notation

DFDs are the cornerstone of threat modeling. They provide a visual representation of the system’s architecture, showing how data moves between components and where it is stored. Understanding the notation is critical for building effective models in both Threat Dragon and PyTM.

**Core DFD Elements**

| Element | Symbol | Description |
|---------|--------|-------------|
| **External Entity** | Rectangle or square (often labeled) | An external actor or system that interacts with the system under analysis. Examples: end‑user, third‑party API, legacy mainframe. They are **outside** the trust boundary. |
| **Process** | Circle or rounded rectangle | A component that transforms data (e.g., a microservice, a function, a batch job). Processes reside inside the system boundary and have inputs and outputs. |
| **Data Store** | Open‑ended rectangle (two parallel lines) | A repository where data is persisted (e.g., database, file system, cache). Data stores are passive – data flows to/from them. |
| **Data Flow** | Arrow | Indicates the direction of data movement between entities, processes, and data stores. Each flow is labelled with the data type or content (e.g., “PaymentRequest”, “UserCredentials”). |
| **Trust Boundary** | Dashed line or dotted rectangle | A logical boundary that separates areas of different trust levels (e.g., Internet vs. internal network, user session vs. backend). Crossing a trust boundary implies a change in security context and requires careful threat analysis. |

**Why DFDs Matter for Threat Modeling**

- **Scope definition** – DFDs visually bound the system and its external dependencies.
- **Threat identification** – Each element and flow can be analysed with STRIDE; e.g., a data flow crossing a trust boundary is susceptible to tampering or eavesdropping.
- **Communication** – DFDs are accessible to both technical and non‑technical stakeholders, facilitating collaborative threat brainstorming.

**Typical DFD Levels**

- **Level 0 (Context Diagram)** – The highest level: one process (the entire system) interacting with external entities. Useful for setting scope.
- **Level 1** – Decomposes the single process into major sub‑processes, data stores, and flows. This is the typical starting point for threat modeling.
- **Level 2+** – Further decomposition of specific sub‑processes for detailed analysis (optional).

In practice, a **Level 1 DFD** is sufficient for most threat models; deeper decomposition can be applied to high‑risk areas.

---

## 2. Threat Modeling Process for Microservices and Legacy Systems

### 2.1 Define Scope and System Representation

Start with a bounded system – e.g., a payment microservice cluster plus legacy batch processes. To build a robust DFD, follow these steps:

1. **Identify external entities** – Who or what sends data into the system or receives data from it? (Users, partner APIs, external databases, etc.)
2. **Identify internal processes** – What are the main functions that process the data? (API gateways, order service, payment processor, notification service, etc.)
3. **Identify data stores** – Where is data at rest? (Databases, S3 buckets, caches, message queues that retain messages, etc.)
4. **Map data flows** – For each interaction, draw an arrow showing what data moves and in which direction. Label flows with the type of data (e.g., “OrderDTO”, “PaymentToken”).
5. **Draw trust boundaries** – Enclose groups of components that share the same trust level (e.g., inside a VPC, inside a Kubernetes cluster). Mark the boundary where trust changes (e.g., from public Internet to private subnet).

Represent your system as:

- **Data Flow Diagrams (DFDs)** – the primary artifact.
- **Infrastructure‑as‑Code** (Kubernetes manifests, Terraform, etc.) – can be transformed into PyTM models.
- **API specifications** (OpenAPI) and event schemas.

With PyTM, you can directly encode these in Python. With Threat Dragon, you draw DFDs interactively.

### 2.2 Identify Assets and Trust Boundaries

- **Assets** – services, databases, message brokers, identity providers, external dependencies.
- **Data classification** – mark PII, financial data, secrets, logs (PyTM supports `Data` objects with classification).
- **Trust boundaries** – areas of changing trust (e.g., Internet → DMZ → internal network). Both tools allow you to mark boundaries on DFDs.

### 2.3 Enumerate Threats

Using STRIDE, list threats per element and data flow:

- **Automated generation** – PyTM includes a threat rule engine that automatically suggests threats based on element types.
- **Manual addition** – Threat Dragon allows you to attach STRIDE threats to diagram elements via its web UI.

### 2.4 Analyze Risk and Prioritize

Assign severity (e.g., CVSS) based on likelihood, impact, and exploitability.

- **PyTM** – can include CVSS scores and custom risk fields in its output.
- **Threat Dragon** – supports risk ratings per threat.

### 2.5 Define and Track Mitigations

Each threat should have one or more mitigations. Both tools allow linking mitigations to threats. PyTM can output mitigation checklists, while Threat Dragon stores them in its model.

### 2.6 Integrate Into Lifecycle and CI/CD

- **PyTM** – ideal for pipeline integration: run `pytm` as a CLI step to generate diagrams/reports and fail builds on critical risks.
- **Threat Dragon** – primarily manual; model files (TMF) can be versioned in Git and reviewed in pull requests.

---

## 3. Focused Tooling: OWASP Threat Dragon & OWASP PyTM

### 3.1 OWASP Threat Dragon – Visual, Collaborative Threat Modeling

**Description:** A browser‑based (or desktop) application that allows teams to create data‑flow diagrams and attach STRIDE threats with a few clicks.

**Key Features:**

- Drag‑and‑drop DFD creation with trust boundaries.
- STRIDE threat generation and manual threat addition.
- Threat model stored in a **Threat Model File (TMF)** – a JSON‑based format that can be version‑controlled.
- Supports team collaboration via GitHub/GitLab integration (when self‑hosted).
- Lightweight desktop version for offline use.

**Best For:**  
- Stakeholder workshops and collaborative reviews.  
- Teams that prefer a visual, interactive approach.  
- Quick prototyping of threat models without coding.

**Limitations for Automation:**  
- No official CLI or REST API – automation is limited to file‑based manipulation of TMF.  
- Lacks built‑in AI or LLM integration.

---

#### 3.1.1 Creating a DFD in Threat Dragon

**Step‑by‑Step:**

1. **Start a new model** – Open Threat Dragon and create a new threat model. Give it a name and description.
2. **Add external entities** – From the palette, drag the “Actor” or “External Entity” shape onto the canvas. For each external system, add a shape and label it (e.g., “Customer”, “Payment Gateway”).
3. **Add processes** – Drag the “Process” (rounded rectangle) for each internal component (e.g., “Order Service”, “Payment Processor”). Place them inside the trust boundary.
4. **Add data stores** – Drag the “Data Store” (open‑ended rectangle) for databases, caches, etc. (e.g., “Order DB”, “Cache”).
5. **Add data flows** – Use the “Flow” arrow tool to connect elements. Click from source to target. For each flow, double‑click to edit properties: name (e.g., “SubmitOrder”) and data type.
6. **Define trust boundaries** – Use the “Boundary” shape (dashed rectangle) to enclose all processes and data stores that share a trust level. You may have nested boundaries (e.g., a DMZ within a wider trusted network).
7. **Save the model** – Threat Dragon saves as a `.tmf` file. Commit this file to your version control system.

**Attaching Threats:**

- Right‑click any element or flow and select “Add Threat”.
- Choose a STRIDE category and describe the threat.
- Assign a risk level (Low, Medium, High, Critical).
- Optionally, add mitigations and references.

**Collaboration:** Share the model via the hosted version or export the `.tmf` and import in another instance. Use GitHub/GitLab integration to manage changes.

---

### 3.2 OWASP PyTM – Programmable Threat Modeling as Code

**Description:** A Python framework that treats threat models as executable scripts. You define your system using Python classes (`TM`, `Server`, `Datastore`, `Dataflow`, `Boundary`, etc.), and PyTM generates DFDs, sequence diagrams, and threat reports automatically.

**Key Features:**

- **Model as code** – all definitions live in Python, making them testable and versionable.
- **Automated threat generation** – a built‑in rule engine (threat database) creates STRIDE threats based on element types and data flows.
- **Outputs** – DFD (Graphviz), sequence diagrams, HTML/PDF reports, JSON, SQLite dump.
- **Extensible** – you can supply a custom threat database or override threat attributes.
- **CLI** – invoke via `pytm` command with options like `--dfd`, `--seq`, `--report`, `--json`.

**Best For:**  
- Integration into CI/CD pipelines and developer workflows.  
- Environments that demand automation and repeatability.  
- Teams comfortable with Python.

**AI‑Agent Friendliness (High):**  
- All inputs are plain Python; outputs include structured JSON.  
- LLMs can generate, modify, or extend PyTM scripts from prose or IaC.  
- Can be wrapped as a tool for MCP‑based agents.

---

#### 3.2.1 Creating a DFD in PyTM (Model as Code)

**Step‑by‑Step:**

1. **Install PyTM** – `pip install pytm`.
2. **Create a Python script** – Import the PyTM classes: `from pytm import TM, Server, Datastore, Dataflow, Boundary, Actor`.
3. **Instantiate the Threat Model** – `tm = TM("My System")`.
4. **Define elements**:

   - **External entities** – Use `Actor` for external users/systems. Example:
     ```python
     user = Actor("User")
     payment_gateway = Actor("Payment Gateway")
     ```
   - **Processes** – Use `Server` for internal services. Example:
     ```python
     order_service = Server("Order Service")
     payment_processor = Server("Payment Processor")
     ```
   - **Data stores** – Use `Datastore` (with `isSQL=True` for databases). Example:
     ```python
     order_db = Datastore("Order Database", isSQL=True)
     cache = Datastore("Cache", isSQL=False)
     ```
   - **Data flows** – Use `Dataflow` between elements. Specify `protocol` and `data` types. Example:
     ```python
     Dataflow(user, order_service, "SubmitOrder", protocol="HTTPS", data="OrderDTO")
     Dataflow(order_service, order_db, "StoreOrder", protocol="SQL", data="OrderRecord")
     ```
   - **Trust boundaries** – Use `Boundary` to group elements. Example:
     ```python
     internal = Boundary("Internal Network")
     internal.contains(order_service, payment_processor, order_db, cache)
     ```
5. **Set the entry point** – `tm.process()` will generate the DFD and threats.

6. **Run the script** – Save as `model.py` and execute:
   ```bash
   pytm --dfd model.py
   ```
   This generates a Graphviz `.dot` file and a DFD image (PNG/SVG). You can also generate reports:
   ```bash
   pytm --report model.py
   ```

**Example Complete Script (simple payment system):**

```python
from pytm import TM, Actor, Server, Datastore, Dataflow, Boundary

tm = TM("Payment System")

user = Actor("User")
gateway = Actor("External Payment Gateway")

order_service = Server("Order Service")
payment_processor = Server("Payment Processor")
notification = Server("Notification Service")

order_db = Datastore("Order Database", isSQL=True)
cache = Datastore("Cache", isSQL=False)

# Data flows
Dataflow(user, order_service, "SubmitOrder", protocol="HTTPS", data="OrderPayload")
Dataflow(order_service, order_db, "SaveOrder", protocol="SQL", data="OrderRecord")
Dataflow(order_service, payment_processor, "ProcessPayment", protocol="REST", data="PaymentRequest")
Dataflow(payment_processor, gateway, "ForwardPayment", protocol="HTTPS", data="PaymentDetails")
Dataflow(payment_processor, order_service, "PaymentResult", protocol="REST", data="Result")
Dataflow(order_service, user, "OrderConfirmation", protocol="HTTPS", data="Confirmation")
Dataflow(order_service, notification, "SendNotification", protocol="MQ", data="NotificationMessage")

# Boundaries
internal = Boundary("Internal Network")
internal.contains(order_service, payment_processor, notification, order_db, cache)

# Generate DFD and threats
tm.process()
```

**Running** – `pytm --dfd payment_model.py` produces the DFD; `pytm --json payment_model.py` gives JSON threat data.

---

### 3.3 Choosing Between the Two (or Combining)

| Criterion | Threat Dragon | PyTM |
|-----------|---------------|------|
| **Primary Interaction** | GUI drag‑and‑drop | Python scripting |
| **DFD Creation** | Visual, interactive | Code‑based, automated |
| **Automation** | Low (file‑based only) | High (CLI + JSON) |
| **CI/CD Integration** | Manual or Git‑based review | Seamless pipeline step |
| **AI Integration** | Indirect (TMF manipulation) | Direct (Python/JSON) |
| **Learning Curve** | Low | Medium (Python required) |
| **Collaboration** | Excellent (UI, sharing) | Good (Git, code reviews) |
| **Community / Use** | OWASP project, active | OWASP project, very active |

**Recommendation:** Use **both**:
- Start with **Threat Dragon** for initial brainstorming, visual mapping, and stakeholder alignment.
- Migrate the model to **PyTM** (or create a parallel PyTM model) for automation, pipeline integration, and AI‑enhanced analysis.

---

## 4. Automation, LLM, and AI‑Agent Integration

### 4.1 Why PyTM Excels for AI Agents and MCP Compatibility

PyTM meets all criteria for being AI‑agent friendly:

| Criterion | PyTM Implementation |
|-----------|----------------------|
| **CLI / REST API** | CLI (`pytm`) – can be invoked from any agent. |
| **Structured File Formats** | Python scripts as input; JSON, SQLite, Markdown as output. |
| **Documented Schemas** | Clear class definitions and threat database schema. |
| **Local Execution** | Runs locally with Python, no external auth required. |

These properties make it a natural fit for agents using MCP (Model Context Protocol) or similar frameworks – an agent can generate a PyTM script, execute it, and parse the JSON results to suggest improvements or create tickets.

### 4.2 Using Threat Dragon with AI Assistance

Although Threat Dragon lacks direct automation, you can still use AI to:

- Generate a **TMF file** (JSON) from a description of the system and import it into Threat Dragon.
- Read an exported TMF, augment it with additional threats, and re‑import.
- Use LLMs to review the threat model and suggest missing mitigations.

However, this is less seamless than PyTM.

### 4.3 AI Integration Patterns for PyTM

1. **Generate a PyTM model from prose**  
   Provide an LLM with a system description (e.g., “We have a payment service, a database, and an external API”) and ask it to output a complete PyTM Python script.

2. **Extend an existing model**  
   Supply the existing script and ask the LLM to add a new microservice or data flow, along with associated threats.

3. **Analyse JSON output**  
   Run `pytm --json` and feed the result to an LLM to produce a risk summary or action items.

4. **Automated threat refinement**  
   Use an agent to iteratively add custom threats or modify risk scores based on team feedback.

5. **MCP tool wrapper**  
   Create an MCP server that exposes PyTM operations as tools (e.g., `generate_threat_model`, `get_risks_json`), allowing agents to invoke them in a standardised way.

---

## 5. Practical Recommendations

### 5.1 Recommended Workflow

A pragmatic workflow that leverages both tools:

1. **Initial Discovery** – Use **Threat Dragon** to draw DFDs with team members, define assets and boundaries, and capture initial threats.
2. **Translate to PyTM** – Convert the DFD into a PyTM Python script (can be done manually or with AI assistance).
3. **Automate Analysis** – Run PyTM in your CI pipeline to produce HTML reports and JSON data; fail builds on high‑severity risks.
4. **Iterative Updates** – As the architecture evolves, update the PyTM script (or regenerate with AI) and refresh the Threat Dragon model manually if needed for team reviews.
5. **Track Mitigations** – Store mitigations in both tools and link to issue trackers (e.g., Jira) via PyTM’s exported data.

### 5.2 Tool Usage by Phase

| Phase | Recommended Tool(s) |
|-------|----------------------|
| **Architecture design / brainstorming** | Threat Dragon |
| **Threat enumeration & risk analysis** | PyTM (automated) + manual refinement in Threat Dragon |
| **CI/CD integration & reporting** | PyTM |
| **Collaborative reviews** | Threat Dragon (UI) or PyTM (if team reads Python) |
| **AI‑driven evolution** | PyTM (JSON/Python) |

### 5.3 AI‑Agent and MCP Integration with PyTM

To build an MCP‑compatible agent for threat modeling:

- Create a set of **MCP tools** that wrap the PyTM CLI:
  - `pytm_generate_model(script_content) -> output_files`
  - `pytm_export_json(script_path) -> json`
- Define **JSON schemas** for the PyTM script (or rely on the existing Python grammar) so the LLM can generate valid code.
- Use the agent to automate threat modeling for new services, or to re‑evaluate existing models after architectural changes.

Example agent prompt:

> “Given the following microservice architecture (…), write a PyTM script that models it. Include all relevant data flows, trust boundaries, and run the analysis. Then output a summary of top risks.”

The agent can run the script and return a human‑readable summary.

---

## 6. Summary

OWASP Threat Dragon and OWASP PyTM together offer a complete threat modeling solution for software applications – from intuitive visual modeling to fully automated, pipeline‑ready threat analysis. While Threat Dragon excels in team collaboration and early‑stage design, PyTM brings the power of “threat modeling as code”, enabling robust automation and seamless integration with AI agents and MCP ecosystems.

By adopting this dual‑tool approach, teams can continuously model and manage threats across both greenfield microservices and brownfield legacy systems, ensuring security is built in from the start and maintained throughout the software lifecycle. The key to success is a solid DFD foundation, which both tools support in complementary ways – Threat Dragon through visual design, PyTM through programmable definitions. Choose the right tool for each phase and benefit from the strengths of both.

---

**Enhancements summary:** Added Section 1.3 (DFD fundamentals) and detailed subsections under each tool explaining step‑by‑step DFD creation, including a full PyTM code example. This addresses your request to clarify “what are the entities etc.” and how to create DFDs with both tools.
