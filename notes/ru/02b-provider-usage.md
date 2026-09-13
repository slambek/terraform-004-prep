# 2b. Describe how Terraform uses providers

> **Scope note:** конфигурации providers *внутри модулей* (передача providers дочерним модулям, `configuration_aliases`) относятся к **Section 5 (Modules)** — там они и будут разобраны, поскольку для них нужен контекст модуля. Эта заметка сфокусирована на providers на уровне root-модуля.

## 1. Provider Plugin Architecture

У Terraform есть две части:

```text
Terraform Configuration
        ↓
Terraform Core   (config, state, dependency graph, plan, apply)
        ↓ RPC
Provider Plugin  (auth, API calls, resources, data sources)
        ↓
Cloud / SaaS / API
```

Terraform Core не обладает встроенным знанием об AWS, Azure, Kubernetes и т.д. — это знание целиком живёт в provider-плагине, и Core общается с ним по RPC.

## 2. What a Provider Implements

* **Resources** — позволяют Terraform создавать/управлять инфраструктурой (`resource "aws_instance" "example" {}`). Каждый тип ресурса реализуется ровно одним provider'ом.
* **Data sources** — позволяют Terraform *читать* уже существующую информацию, вместо того чтобы что-то создавать (`data "aws_ami" "example" {}`).

```text
resource → creates / manages
data     → reads only
```

## 3. Provider Configuration (the `provider` block)

Блок `provider` настраивает, **как** Terraform использует уже объявленный provider (регион, endpoint, auth и т.д.) — он **не** устанавливает provider и не задаёт его версию (это делает `required_providers`, см. 2a).

```hcl
provider "aws" {
  region = var.region   # can reference variables / other known-before-apply values
}
```

Credentials часто лучше передавать через переменные окружения или другие внешние механизмы, а не хардкодить в конфигурации, чтобы секреты не попадали в систему контроля версий.

## 4. Provider Aliases — Multiple Configs of the *Same* Provider

Используйте `alias`, когда нужно несколько конфигураций одного и того же provider'а (например, два региона AWS):

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

Явно выберите aliased-конфигурацию на ресурсе:

```hcl
resource "aws_instance" "example" {
  provider = aws.west
}
```

*(Сравните с 2c: aliases = несколько конфигураций **одного** provider'а; 2c = несколько **разных** providers в одной конфигурации — легко перепутать на экзамене.)*

## 5. Default Configuration & the Implied Empty Default

Блок `provider` **без** `alias` — это конфигурация по умолчанию — её использует любой ресурс, который не задаёт `provider = ...`.

Если **у всех** конфигураций provider'а есть alias, явного default нет, поэтому Terraform создаёт **подразумеваемую пустую конфигурацию по умолчанию (implied empty default)**. Ресурс без мета-аргумента `provider` попытается использовать эту пустую конфигурацию — что приведёт к ошибке, если у provider'а есть обязательные аргументы.

```text
provider "aws" { alias = "east" ... }
provider "aws" { alias = "west" ... }
        ↓
aws  →  implied empty default (dangerous if aws needs required args)
```

## Exam Quick Facts

* Terraform Core ↔ Provider Plugin общаются по **RPC**; у Core нет собственной сервис-специфичной логики.
* Каждый тип ресурса принадлежит ровно одному provider'у; **resource** управляет, **data source** только читает.
* Блок `provider` настраивает использование (регион, auth и т.д.) — он никогда не задаёт версию и ничего не устанавливает.
* Нет `alias` → конфигурация по умолчанию. Alias есть у всех конфигураций → подразумеваемый *пустой* default (может привести к ошибке, если provider требует аргументы).
* Конфигурация provider внутри модулей — отдельная тема — см. Section 5.