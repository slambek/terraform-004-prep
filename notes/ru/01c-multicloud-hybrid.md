# 1c. Explain How Terraform Manages Multi-Cloud, Hybrid Cloud, and Service-Agnostic Workflows

## Core Concept
Terraform **service-agnostic** (не привязан к сервису). Он использует **providers** (плагины) для взаимодействия с публичными облаками, on-prem-инфраструктурой, SaaS и Kubernetes через их API, используя единый унифицированный декларативный язык (HCL) и единый workflow (`init` -> `plan` -> `apply`).

---

## Key Exam Concepts & Capabilities

### 1. Provider Abstraction
- **Plugins, Not Core:** Terraform Core ничего не знает про AWS, Azure или Kubernetes. **Providers** обрабатывают вызовы API к целевым платформам.
- **Terraform Registry:** Содержит более 1000 публичных, community и кастомных провайдеров.
- *Exam takeaway:* Terraform не привязан к единственному вендору или экосистеме.

### 2. Multi-Cloud & Cross-Provider Dependencies
- **Single Resource Graph:** Terraform отображает зависимости между разными провайдерами в едином графе выполнения.
- *Example Chain:* Создать AWS VPC → развернуть Kubernetes-кластер на AWS → создать Cloudflare DNS-запись.
- *Exam takeaway:* Terraform автоматически обрабатывает кросс-провайдерную оркестрацию на основе зависимостей ресурсов.

### 3. Unified Hybrid & Workload Management
- **One Workflow for Everything:** Управляет IaaS (AWS, VMware), оркестрацией (Kubernetes, Helm) и SaaS (Datadog, GitHub, Cloudflare).
- **Kubernetes Integration:** Провижинит K8s-кластеры И управляет K8s-манифестами/ресурсами (Services, Pods, Deployments) в рамках одного и того же инструмента.

### 4. Standardization & Self-Service
- **Modules:** Ops-команды публикуют стандартизированные, заранее одобренные модули инфраструктуры.
- **Developer Enablement:** Команды приложений разворачивают соответствующую требованиям инфраструктуру самостоятельно, без ручных заявок.

### 5. Policy, Governance & Ephemeral Envs
- **Sentinel (Policy as Code):** Обеспечивает соблюдение compliance, лимитов по стоимости и правил безопасности *до* `apply` (HCP Terraform / Enterprise). Полная механика — уровни enforcement, policy sets, OPA как альтернативный фреймворк — в **8b**.
- **Disposable Infrastructure:** Быстро создаёт и уничтожает (`destroy`) временные окружения (QA, Staging, демо), чтобы снизить затраты.

---

## Exam Quick Summary

| Concept | How Terraform Implements It |
| :--- | :--- |
| **Service-Agnostic** | Providers переводят HCL в API любой платформы |
| **Multi-Cloud Orchestration** | Resource Graph управляет кросс-провайдерными зависимостями |
| **Hybrid & K8s** | Провижинит облачную инфраструктуру И ресурсы внутри кластера |
| **Governance** | Sentinel / OPA policies проверяют код перед выполнением |
| **Cost Savings** | Быстрое создание и уничтожение ephemeral-окружений |

---

## Quick Recall

**Terraform Multi-Cloud Formula:**  
`Providers` → `Single Configuration` → `Resource Graph (Dependencies)` → `Unified CLI Workflow`

**Exam Keyword Trigger:**  
Когда спрашивают, как Terraform обрабатывает **multi-cloud/service-agnostic** окружения, ищи ответы, упоминающие: **Providers + Resource Graph + Single CLI Workflow**.