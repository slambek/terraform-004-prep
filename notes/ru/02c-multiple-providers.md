# 2c. Write Terraform configuration using multiple providers

## Core Concept

Одна конфигурация Terraform может использовать **более одного provider'а одновременно** — например `aws` + `random`, или `aws` + `cloudflare` + `github`. Terraform Core неважно, сколько providers задействовано; он строит единый **resource graph** для всех них и автоматически разрешает зависимости, даже через границы providers.

**Important distinction (exam trap):**

* **2b (aliases)** → несколько **конфигураций *одного и того же* provider'а** (`aws` + `aws.west`).
* **2c (эта заметка)** → несколько **разных providers**, используемых вместе в одной конфигурации (`aws` + `random`).

Это отдельные концепции, которые легко перепутать на экзамене.

## Declaring Multiple Providers

Каждый используемый provider должен быть перечислен в `required_providers`, внутри блока верхнего уровня `terraform`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

* У каждого provider'а есть своё **независимое version constraint** (подробности в 2a).
* У каждого provider'а есть свой **отдельный `provider` block**, если он нуждается в конфигурации (подробности в 2b). `random` не требует конфигурации, поэтому `provider "random" {}` можно полностью опустить — Terraform подразумевает пустую конфигурацию по умолчанию.

```hcl
provider "aws" {
  region = "us-west-2"
}

provider "random" {}
```

## Cross-Provider Resource References

Ресурсы разных providers могут ссылаться на атрибуты друг друга. Terraform обнаруживает это и добавляет как **implicit dependency** в resource graph, независимо от того, какому provider'у принадлежит какой ресурс.

```hcl
resource "random_pet" "name" {
  length = 2
}

resource "aws_instance" "web" {
  ami           = "ami-a0cfeed8"
  instance_type = "t2.micro"

  tags = {
    Name = random_pet.name.id   # cross-provider reference: random -> aws
  }
}
```

* `random_pet.name` принадлежит provider'у `random`.
* `aws_instance.web` принадлежит provider'у `aws`.
* Terraform создаст `random_pet.name` **первым**, потому что `aws_instance.web` зависит от его output.

## How Terraform Resolves the Local (Provider) Name

Terraform определяет, каким provider'ом реализован тип ресурса, по **префиксу имени типа ресурса**:

```text
aws_instance      → prefix "aws"     → aws provider
random_pet        → prefix "random"  → random provider
google_compute_*  → prefix "google"  → google provider
```

Именно поэтому важно использовать **предпочтительное local name** provider'а (см. 2a/2b): это позволяет Terraform автоматически определить provider, не требуя мета-аргумента `provider = ...` на каждом ресурсе.

## Multi-Cloud / Multi-Service Example

Концептуально ничего не меняется, когда providers представляют разные облачные платформы или SaaS-инструменты — тот же паттерн `required_providers` + `provider` block применяется к каждому из них:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}
```

Единый resource graph Terraform координирует порядок зависимостей между ресурсами `aws` и `cloudflare` точно так же, как это было с `aws` и `random` выше. Это тот самый механизм, на который ссылается 1c ("Multi-Cloud Formula").

## Provider Source Tiers (Terraform Registry)

Когда вы добавляете новый provider в конфигурацию, его значок Registry показывает, кто его поддерживает — полезный контекст при выборе providers для реальной конфигурации:

| Tier | Maintained by | Namespace example |
| :--- | :--- | :--- |
| **Official** | HashiCorp | `hashicorp`, `ansible` |
| **Partner Premier** | Проверенный технологический партнёр | Стороннее сообщество/организация |
| **Partner** | Технологический партнёр | Стороннее сообщество/организация |
| **Community** | Индивидуальные/community-мейнтейнеры | Аккаунт мейнтейнера |
| **Archived** | Больше не поддерживается | `hashicorp` или сторонний |

## Built-in Provider (No Declaration Needed)

Terraform поставляется ровно с **одним встроенным provider'ом**: он обеспечивает работу data source `terraform_remote_state`. Его source address — `terraform.io/builtin/terraform`. Поскольку он встроен в Terraform Core, ему **не** нужна запись в `required_providers` — в отличие от всех остальных providers, используемых в конфигурации.

## Exam Quick Facts

* Конфигурация может свободно смешивать любое количество **разных** providers.
* Каждый provider объявляется независимо в `required_providers` со своим `source` и `version`.
* Terraform автоматически разрешает зависимости **между** providers через resource graph — для cross-provider ссылок не нужен специальный синтаксис.
* Terraform определяет provider для ресурса по префиксу типа ресурса (`aws_*` → `aws`).
* Смешивание нескольких **разных** providers (2c) ≠ несколько **конфигураций одного** provider'а через `alias` (2b).
* Встроенный provider `terraform` (для `terraform_remote_state`) — единственное исключение, не требующее записи в `required_providers`.