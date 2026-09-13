# 2a. Install and version Terraform providers

## Core Concept

**Terraform providers** — это плагины, которые позволяют Terraform взаимодействовать с облачными платформами, SaaS-сервисами и другими API.

Providers **версионируются отдельно от самого Terraform** и распространяются в основном через **Terraform Registry**.

Terraform должен знать:

* **какой provider** требуется,
* **где его взять**,
* **какие версии допустимы**.

## Key Exam Concepts

* **`required_providers`:**

  * Объявляет providers, которые требуются модулю.
  * Должен находиться внутри блока верхнего уровня **`terraform`**.
  * Требование к provider включает:

    * **Local name** — имя, используемое внутри конфигурации.
    * **Source address** — определяет, откуда взят provider.
    * **Version constraint** — определяет, какие версии допустимы.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.5.0"
    }
  }
}
```

* **Provider source address:**

  * Идентифицирует provider и его источник.
  * Формат:

```text
[hostname/]namespace/type
```

* Пример:

```text
hashicorp/aws
```

* Если hostname не указан, Terraform использует **`registry.terraform.io`**.

* **Local provider name:**

  * Имя, используемое для ссылки на provider в конфигурации.
  * Пример:

```hcl
aws = {
  source  = "hashicorp/aws"
  version = ">= 4.5.0"
}
```

* `aws` → **local name**

* `hashicorp/aws` → **source address**

* **Version constraints:**

  * Определяют, какие версии provider'а Terraform разрешено использовать.

  * Аргумент `version` относится к **`required_providers`**.

  * **Minimum version:**

```hcl
version = ">= 1.0"
```

Разрешает `1.0` и более новые версии.

* **Pessimistic constraint:**

```hcl
version = "~> 1.0.4"
```

Разрешает совместимые обновления patch-уровня в рамках minor-релиза `1.0`.

* **Exact version:**

```hcl
version = "3.1.0"
```

Разрешена только `3.1.0`.

* **`provider` block:**

  * Настраивает уже объявленный provider.
  * Используется для специфичных для provider'а настроек, таких как регион или endpoint.

```hcl
provider "aws" {
  region = "us-west-2"
}
```

**Important distinction:**

* `required_providers` → **какой provider и какие версии**
* `provider` block → **как provider настроен**

**Exam trap:** Не помещайте `version` внутрь блока `provider`. Version constraints относятся к `required_providers`.

## Core Workflow

1. **Declare the provider**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

2. **Configure the provider**

```hcl
provider "aws" {
  region = "us-west-2"
}
```

3. **Initialize Terraform**

```bash
terraform init
```

`terraform init`:

* находит требуемые providers,
* выбирает версии providers согласно constraints и lock-файлу,
* скачивает и устанавливает providers,
* создаёт или обновляет **`.terraform.lock.hcl`**.

4. **Use Terraform**

```bash
terraform plan
terraform apply
```

Terraform использует версию provider'а, выбранную в lock-файле.

5. **Upgrade providers intentionally**

```bash
terraform init -upgrade
```

Это ищет более новые версии provider'ов, удовлетворяющие настроенным version constraints, и обновляет lock-файл.

## Dependency Lock File

**`.terraform.lock.hcl`** фиксирует **конкретные версии providers, выбранные Terraform**, и их **checksums**.

* **Version constraint** → определяет, какие версии **допустимы**.
* **Lock file** → фиксирует, какая версия Terraform **выбрана**.

Пример:

```hcl
version = ">= 4.5.0"
```

Это разрешает `4.5.0` или более новую.

Если `.terraform.lock.hcl` зафиксировал:

```text
4.5.0
```

то обычный:

```bash
terraform init
```

будет обычно продолжать использовать `4.5.0`.

Чтобы намеренно рассмотреть более новые допустимые версии:

```bash
terraform init -upgrade
```

**Коммитьте `.terraform.lock.hcl` в систему контроля версий.**

Lock-файл также содержит **checksums**, которые Terraform использует для верификации пакетов providers.

## Exam Quick Facts

* **Version constraint** (`required_providers`) = что **допустимо**. **Lock file** = что фактически **выбрано**.
* `version` никогда не идёт внутрь блока `provider` — только внутрь `required_providers`.
* `terraform init` устанавливает providers и записывает/обновляет `.terraform.lock.hcl`; `terraform init -upgrade` — единственное, что пересматривает более новые допустимые версии.
* Коммитьте `.terraform.lock.hcl` в систему контроля версий.