# HashiCorp Certified: Terraform Associate (004) — Конспекты подготовки

Репозиторий для подготовки к сертификации **HashiCorp Certified: Terraform Associate (004)**.

Все цели экзамена из разделов 1–8 покрыты. Конспекты организованы по принципу:
**один `.md` файл в `notes/ru/` = одна цель экзамена**. Для тем, которые
пересекаются между несколькими целями, используются явные примечания
`Scope note`, чтобы основные материалы находились в одном месте и не
дублировались по разным файлам.

## Структура репозитория

```text
.
├── README.en.md          # README на английском
├── README.ru.md          # README на русском
├── CHEATSHEET.en.md      # Краткая шпаргалка на английском
├── CHEATSHEET.ru.md      # Краткая шпаргалка на русском
└── notes/
    ├── en/               # Конспекты на английском
    │   ├── 01a-what-is-iac.md
    │   ├── 01b-advantages-of-iac.md
    │   ├── ...
    │   └── 08d-cli-integration.md
    └── ru/               # Конспекты на русском
        ├── 01a-what-is-iac.md
        ├── 01b-advantages-of-iac.md
        ├── ...
        └── 04h-sensitive-data-vault.md
```

## Разделы и цели

| Цель                                                                                                 | Файл                                                                                              |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 1a. Объяснить, что такое IaC                                                                         | [01a-what-is-iac.md](notes/ru/01a-what-is-iac.md)                                                 |
| 1b. Описать преимущества подходов IaC                                                                | [01b-advantages-of-iac.md](notes/ru/01b-advantages-of-iac.md)                                     |
| 1c. Объяснить, как Terraform работает с multi-cloud, hybrid cloud и сервисно-независимыми сценариями | [01c-multicloud-hybrid.md](notes/ru/01c-multicloud-hybrid.md)                                     |
| 2a. Устанавливать и версионировать Terraform providers                                               | [02a-install-providers.md](notes/ru/02a-install-providers.md)                                     |
| 2b. Описать, как Terraform использует providers                                                      | [02b-provider-usage.md](notes/ru/02b-provider-usage.md)                                           |
| 2c. Писать конфигурацию Terraform с использованием нескольких providers                              | [02c-multiple-providers.md](notes/ru/02c-multiple-providers.md)                                   |
| 2d. Объяснить, как Terraform использует и управляет state                                            | [02d-state-fundamentals.md](notes/ru/02d-state-fundamentals.md)                                   |
| 3a. Описать рабочий процесс Terraform                                                                | [03a-terraform-workflow.md](notes/ru/03a-terraform-workflow.md)                                   |
| 3b. Инициализировать рабочую директорию Terraform                                                    | [03b-terraform-init.md](notes/ru/03b-terraform-init.md)                                           |
| 3c. Проверять конфигурацию Terraform                                                                 | [03c-terraform-validate.md](notes/ru/03c-terraform-validate.md)                                   |
| 3d. Создавать и анализировать execution plan Terraform                                               | [03d-terraform-plan.md](notes/ru/03d-terraform-plan.md)                                           |
| 3e. Применять изменения инфраструктуры с помощью Terraform                                           | [03e-terraform-apply.md](notes/ru/03e-terraform-apply.md)                                         |
| 3f. Уничтожать инфраструктуру, управляемую Terraform                                                 | [03f-terraform-destroy.md](notes/ru/03f-terraform-destroy.md)                                     |
| 3g. Применять форматирование и стилевые изменения к конфигурации                                     | [03g-terraform-fmt.md](notes/ru/03g-terraform-fmt.md)                                             |
| 4a. Использовать и различать блоки `resource` и `data`                                               | [04a-resource-vs-data-blocks.md](notes/ru/04a-resource-vs-data-blocks.md)                         |
| 4b. Обращаться к атрибутам ресурсов и создавать связи между ресурсами                                | [04b-resource-references.md](notes/ru/04b-resource-references.md)                                 |
| 4c. Использовать variables и outputs                                                                 | [04c-variables-and-outputs.md](notes/ru/04c-variables-and-outputs.md)                             |
| 4d. Понимать и использовать complex types                                                            | [04d-complex-types.md](notes/ru/04d-complex-types.md)                                             |
| 4e. Создавать динамическую конфигурацию с помощью expressions и functions                            | [04e-dynamic-expressions-functions.md](notes/ru/04e-dynamic-expressions-functions.md)             |
| 4f. Определять зависимости ресурсов в конфигурации                                                   | [04f-resource-dependencies.md](notes/ru/04f-resource-dependencies.md)                             |
| 4g. Проверять конфигурацию с помощью пользовательских условий                                        | [04g-validate-custom-conditions.md](notes/ru/04g-validate-custom-conditions.md)                   |
| 4h. Управлять чувствительными данными, включая secrets management с Vault                            | [04h-sensitive-data-vault.md](notes/ru/04h-sensitive-data-vault.md)                               |
| 5a. Объяснить, откуда Terraform получает modules                                                     | [05a-module-sources.md](notes/ru/05a-module-sources.md)                                           |
| 5b. Описать область видимости variables внутри modules                                               | [05b-variable-scope-in-modules.md](notes/ru/05b-variable-scope-in-modules.md)                     |
| 5c. Использовать modules в конфигурации                                                              | [05c-using-modules.md](notes/ru/05c-using-modules.md)                                             |
| 5d. Управлять версиями modules                                                                       | [05d-module-versions.md](notes/ru/05d-module-versions.md)                                         |
| 6a. Описать local backend                                                                            | [06a-local-backend.md](notes/ru/06a-local-backend.md)                                             |
| 6b. Описать state locking                                                                            | [06b-state-locking.md](notes/ru/06b-state-locking.md)                                             |
| 6c. Настраивать remote state с помощью блока backend                                                 | [06c-remote-state-backend-block.md](notes/ru/06c-remote-state-backend-block.md)                   |
| 6d. Управлять resource drift и Terraform state                                                       | [06d-resource-drift-state-mgmt.md](notes/ru/06d-resource-drift-state-mgmt.md)                     |
| 7a. Импортировать существующую инфраструктуру в Terraform workspace                                  | [07a-import-existing-infrastructure.md](notes/ru/07a-import-existing-infrastructure.md)           |
| 7b. Использовать CLI для просмотра state                                                             | [07b-cli-inspect-state.md](notes/ru/07b-cli-inspect-state.md)                                     |
| 7c. Описать, когда и как использовать подробное логирование                                          | [07c-verbose-logging.md](notes/ru/07c-verbose-logging.md)                                         |
| 8a. Использовать HCP Terraform для создания инфраструктуры                                           | [08a-hcp-terraform-create-infrastructure.md](notes/ru/08a-hcp-terraform-create-infrastructure.md) |
| 8b. Описать возможности HCP Terraform для совместной работы и управления                             | [08b-collaboration-governance.md](notes/ru/08b-collaboration-governance.md)                       |
| 8c. Описать организацию и использование HCP Terraform workspaces и projects                          | [08c-workspaces-and-projects.md](notes/ru/08c-workspaces-and-projects.md)                         |
| 8d. Настраивать и использовать интеграцию HCP Terraform                                              | [08d-cli-integration.md](notes/ru/08d-cli-integration.md)                                         |

## Как использовать репозиторий

* **`CHEATSHEET.ru.md`** — для быстрого повторения в день экзамена. Здесь
  собраны только факты, на которых чаще всего ошибаются: edge cases,
  значения по умолчанию, правила вида «X никогда не делает Y» и различия
  между похожими концепциями. Материал сгруппирован по темам, а не по
  отдельным файлам, поэтому дублирование между целями удалено.
* **`notes/ru/`** — для подробного изучения на русском языке. Каждый файл
  посвящён ровно одной цели экзамена и содержит `> Scope note`, если тема
  пересекается с другой целью. Такое примечание указывает, где находятся
  полные детали, если в текущем файле материал намеренно сокращён.
* **`notes/en/`** — английская версия конспектов.

Внутри каждого файла с конспектом структура следующая:
**Core Concept** → подробности цели → **Exam Quick Facts** в конце.

`Exam Quick Facts` содержит краткий список наиболее важных нюансов именно
для этой цели. В `CHEATSHEET.ru.md` эти списки объединены и очищены от
дубликатов.