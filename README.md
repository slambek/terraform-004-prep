# HashiCorp Certified: Terraform Associate (004) Prep

Personal study repository for preparing for the **HashiCorp Certified: Terraform Associate (004)** certification.

The repository contains structured theory notes, practical Terraform configurations, CLI exercises, and high-yield exam summaries focused on the official certification objectives.

---

## 📌 Exam Overview

* **Certification:** HashiCorp Certified: Terraform Associate (004)
* **Focus:** Terraform, Infrastructure as Code, HCL, CLI, state, modules, workflow, and HCP Terraform
* **Question Types:** Multiple choice, multiple select, true/false, and fill-in-the-blank
* **Primary Goal:** Build both conceptual understanding and practical Terraform skills required for the exam.

---

## 📂 Repository Structure

```text
.
├── README.md                         # Project overview and study roadmap
├── CHEATSHEET.md                     # High-yield commands, concepts, and exam triggers
│
├── notes/                            # Detailed study notes
│   ├── 01a-what-is-iac.md
│   ├── 01b-advantages-of-iac.md
│   └── 01c-multi-cloud-and-hybrid-workflows.md
│
└── labs/                             # Practical Terraform exercises
    └── ...
```

---

## 🗺 Study Roadmap

### Section 1 — Understand Infrastructure as Code (IaC) Concepts

* [x] **1a. Explain what IaC is**
* [x] **1b. Describe advantages of IaC patterns**
* [x] **1c. Explain multi-cloud, hybrid, and service-agnostic workflows**

### Section 2 — Understand Terraform Basics

* [ ] **2a. Explain Terraform's purpose and basic concepts**
* [ ] **2b. Understand Terraform providers and resources**
* [ ] **2c. Understand Terraform configuration and HCL**

### Section 3 — Use the Terraform CLI

* [ ] **3a. Understand core Terraform CLI commands**
* [ ] **3b. Initialize and validate configurations**
* [ ] **3c. Plan and apply infrastructure changes**
* [ ] **3d. Inspect and troubleshoot Terraform configurations**

### Section 4 — Interact with Terraform Modules

* [ ] **4a. Understand module structure**
* [ ] **4b. Use modules from the Terraform Registry**
* [ ] **4c. Configure module inputs and outputs**

### Section 5 — Navigate Terraform Workflow

* [ ] **5a. Understand the Terraform workflow**
* [ ] **5b. Understand resource dependencies**
* [ ] **5c. Understand resource lifecycle behavior**

### Section 6 — Implement and Maintain State

* [ ] **6a. Understand Terraform state**
* [ ] **6b. Understand local and remote state**
* [ ] **6c. Understand state locking**
* [ ] **6d. Manage and inspect state**

### Section 7 — Read, Generate, and Modify Configuration

* [ ] **7a. Understand HCL syntax**
* [ ] **7b. Use variables, locals, outputs, and expressions**
* [ ] **7c. Use functions and data sources**
* [ ] **7d. Understand dynamic configuration patterns**

### Section 8 — Understand HCP Terraform Capabilities

* [ ] **8a. Understand HCP Terraform workspaces**
* [ ] **8b. Understand remote state and remote operations**
* [ ] **8c. Understand VCS-driven workflows**
* [ ] **8d. Understand variables, permissions, and collaboration**
* [ ] **8e. Understand policy and governance capabilities**

---

## 🧠 Study Strategy

The preparation is divided into three layers:

### 1. Theory

Study the concepts in `notes/` and understand **why Terraform works the way it does**, rather than memorizing commands.

### 2. Practice

Use the Terraform CLI and small local configurations to practice:

```text
write → init → validate → plan → apply → inspect → destroy
```

Docker is used where possible to practice Terraform workflows without depending on paid cloud infrastructure.

### 3. Exam Recall

Use `CHEATSHEET.md` for:

* CLI command recognition
* Terraform terminology
* Common exam keywords
* "What does X do?" questions
* Differences between similar Terraform concepts
* High-yield facts and traps

---

## 🎯 Core Exam Topics

The preparation focuses on the concepts most important for Terraform Associate:

```text
Infrastructure as Code
        ↓
Terraform Basics
        ↓
HCL & Configuration
        ↓
Providers & Resources
        ↓
Terraform CLI
        ↓
Workflow
        ↓
Modules
        ↓
State
        ↓
Terraform Cloud / HCP Terraform
```

---

## ⚡ Quick Recall

### IaC

**What is IaC?**

> Managing infrastructure through human-readable configuration files instead of manual infrastructure changes.

### Terraform

**What is Terraform?**

> A declarative Infrastructure as Code tool that uses providers to manage infrastructure and services through APIs.

### Provider

**What is a provider?**

> A plugin that allows Terraform to interact with a specific platform or service.

### Resource

**What is a resource?**

> An infrastructure object managed by Terraform.

### State

**Why does Terraform use state?**

> To track the relationship between Terraform configuration and real infrastructure.

### Module

**What is a module?**

> A reusable collection of Terraform configuration.

### Workflow

```text
Write
  ↓
init
  ↓
validate
  ↓
plan
  ↓
apply
```

---

## 🧪 Practical Environment

Primary practice environment:

* Terraform CLI
* Docker provider
* Local development environment
* HCP Terraform where relevant

Cloud-specific resources are used only when they provide meaningful value for understanding the certification objectives.

---

## 📚 Official Documentation

The primary reference for this repository is the official **HashiCorp Terraform documentation and certification objectives**.

Third-party material may be used for additional explanations, but official HashiCorp documentation takes precedence when there is a discrepancy.

---

## 🚀 Progress

Current progress:

```text
Section 1  ████████████████████ 100%
Section 2  ░░░░░░░░░░░░░░░░░░░░   0%
Section 3  ░░░░░░░░░░░░░░░░░░░░   0%
Section 4  ░░░░░░░░░░░░░░░░░░░░   0%
Section 5  ░░░░░░░░░░░░░░░░░░░░   0%
Section 6  ░░░░░░░░░░░░░░░░░░░░   0%
Section 7  ░░░░░░░░░░░░░░░░░░░░   0%
Section 8  ░░░░░░░░░░░░░░░░░░░░   0%
```

**Overall:** 1 / 8 sections completed.

---

## 📝 Notes

These notes are written for **exam preparation**, so they intentionally prioritize:

* Clear mental models
* Exam-relevant terminology
* Short explanations
* Practical examples
* Quick recall
* Distinguishing similar Terraform concepts

The goal is not to document every Terraform feature, but to build a strong understanding of the concepts required for the **Terraform Associate (004)** exam.
