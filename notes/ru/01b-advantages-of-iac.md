# 1b. Describe the Advantages of IaC Patterns

## Core Concept
Infrastructure as Code (IaC) устраняет операционные риски, сложность и ручные ошибки управления через GUI/CLI ("ClickOps"), применяя практики software engineering к провижининг инфраструктуры.

## Key Exam Concepts & Advantages

### 1. Reliability & Risk Mitigation
- **Eliminates Human Error:** Убирает ручные несоответствия, пропущенные шаги и ошибки конфигурации между окружениями.
- **Predictability (`terraform plan`):** Позволяет командам заранее просматривать точные изменения и тестировать код в non-prod окружениях перед применением в production.
- **Disaster Recovery & Scale:** Обеспечивает быстрое, предсказуемое восстановление инфраструктуры при сбоях системы или всплесках трафика.

### 2. Consistency & Idempotency
- **Prevents Configuration Drift:** Гарантирует, что окружения `dev`, `staging` и `prod` остаются идентичными с течением времени.
- **Idempotent Operations:** Повторный запуск одной и той же конфигурации даёт один и тот же результат без непреднамеренных дублей или побочных эффектов.

### 3. Reusability & Standardization
- **Modules:** Переиспользуемые компоненты инфраструктуры, которые экономят время, обеспечивают соблюдение архитектурных стандартов и позволяют делиться лучшими практиками через реестры.
- **Unified Workflow:** Использует единый язык (HCL) и единый workflow (`init` -> `plan` -> `apply`) для множества публичных/приватных облаков и SaaS-провайдеров.

### 4. Auditability & Collaboration
- **Version Control (VCS):** Хранение `.tf`-кода в Git даёт чёткую историю изменений, механизмы код-ревью и мгновенные откаты.
- **Remote State & Locking:** Remote backends (например, HCP Terraform) предотвращают состояния гонки, когда несколько инженеров применяют изменения одновременно.

### 5. Efficiency & Parallel Automation
- **Resource Graph:** Terraform автоматически вычисляет зависимости между ресурсами, провижиня независимые друг от друга ресурсы параллельно.
- **Mutation & Smart Updates:** Сравнивает текущее состояние с желаемым, изменяя только то, что изменилось, оставляя нетронутой валидную инфраструктуру.

## Exam Quick Comparison

| Manual Management (ClickOps) | IaC Pattern (Terraform) |
| :--- | :--- |
| Высокий configuration drift | Гарантированная consistency и idempotency |
| Непрозрачная история изменений | Прозрачный VCS audit trail (Git) |
| Последовательные ручные шаги | Параллельный provisioning через Dependency Graph |
| Высокий риск при масштабировании/сбоях | Предсказуемое выполнение через `terraform plan` |