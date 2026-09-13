# 5c. Use modules in configuration

## The `module` Block's Meta-Arguments

Помимо `source` (5a) и `version` (5d), каждый блок `module` поддерживает следующие нативные для Terraform мета-аргументы:

| Meta-argument | Purpose |
| :--- | :--- |
| `count` | Создать N почти идентичных экземпляров модуля. Взаимоисключим с `for_each`. |
| `for_each` | Создать по одному экземпляру на каждую запись в карте или множестве строк — лучше всего, когда экземплярам нужна **различающаяся** конфигурация, ключом для которой служит что-то осмысленное, а не просто идентичные копии. |
| `providers` | Замапить альтернативную/aliased конфигурацию provider'а в дочерний модуль (см. 5b). |
| `depends_on` | Заставить модуль дождаться upstream-ресурса, который иначе не упоминается в его аргументах (общую механику см. в 4f). |

## Referencing Module Outputs

```hcl
module.<LABEL>.<OUTPUT_NAME>
```

Таким образом доступны только значения, которые модуль явно раскрывает через блоки `output` (5b).

## Multiple Module Instances

**`count`** — хорош для идентичных/почти идентичных копий, индексируемых числами:

```hcl
locals {
  instance_names = ["example-instance-1", "example-instance-2", "example-instance-3"]
}

module "ec2_instance" {
  source = "terraform-aws-modules/ec2-instance/aws"
  count  = length(local.instance_names)
  name   = local.instance_names[count.index]
}
```

**`for_each`** — хорош, когда каждому экземпляру нужна отдельная конфигурация, ключом для которой служит карта или множество:

```hcl
module "ec2_instance" {
  source   = "terraform-aws-modules/ec2-instance/aws"
  for_each = var.instance_configs   # a map keyed by name
  name     = each.key
  ami      = each.value.ami
}
```

## Explicit Module Dependencies

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-example-bucket-12345"
}

module "ec2_instance" {
  source     = "terraform-aws-modules/ec2-instance/aws"
  depends_on = [aws_s3_bucket.example]
}
```

Те же правила, что и при любом другом использовании `depends_on` (4f): по возможности предпочитайте неявные зависимости (ссылку на атрибут); прибегайте к `depends_on` только тогда, когда реальная зависимость существует, но не видна ни через одну ссылку в аргументах.

## Passing Alternate Provider Configurations

Тот же мета-аргумент `providers` и настройка alias `aws.usw1`, что и в примере из 5b —
здесь показано расширенным вторым alias'ом, чтобы проиллюстрировать общую форму:

```hcl
module "tunnel" {
  source = "./tunnel"
  providers = {
    aws.src = aws.usw1   # aws.usw1 / aws.usw2 declared as in 5b
    aws.dst = aws.usw2
  }
}
```

(Полная механика — включая требование `configuration_aliases` у дочернего модуля — изложена в 5b/2b.)

## Recommended Module File Structure

Ни один из этих файлов не является для Terraform чем-то особым — модуль может быть одним `.tf`-файлом — но такая раскладка является конвенцией сообщества:

```text
.
├── LICENSE
├── README.md
├── main.tf
├── variables.tf
└── outputs.tf
```

* `main.tf` — основные ресурсы модуля.
* `variables.tf` — его входной интерфейс.
* `outputs.tf` — его экспортируемые значения.
* `README.md` / `LICENSE` — документация и лицензирование; сам Terraform их игнорирует, но реестры (публичные или приватные) и хостинги кода их отображают.

**Никогда не распространяйте это вместе с модулем** (и добавьте в `.gitignore`):

* `terraform.tfstate` / `terraform.tfstate.backup` — state специфичен для конкретного развёртывания, а не для кода модуля.
* `.terraform/` — локальный кэш плагинов/модулей (см. 3b).
* `*.tfvars` — inputs модуля приходят из аргументов блока `module` вызывающей стороны, а не из `.tfvars`, и такие файлы часто содержат секреты.

## Module Composition: Keep the Tree Flat

Как только вы начинаете вызывать модули, конфигурация становится иерархической, а не плоской — но настоятельная рекомендация состоит в том, чтобы держать дерево модулей в пределах **одного уровня** дочерних модулей и связывать модули между собой обычными выражениями, а не вкладывать их глубоко:

```hcl
module "network" {
  source          = "./modules/aws-network"
  base_cidr_block = "10.0.0.0/8"
}

module "consul_cluster" {
  source     = "./modules/aws-consul-cluster"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.subnet_ids
}
```

Этот стиль «композиции модулей» собирает маленькие, узкоспециализированные модули в более крупную систему из **root**-модуля, вместо того чтобы один модуль внутренне встраивал и управлял ресурсами другого модуля — это сохраняет каждую часть независимо понятной и переиспользуемой в разных комбинациях.

## Exam Quick Facts

* `count` и `for_each` на блоке `module` взаимоисключимы — то же правило, что и на блоках `resource`/`data`.
* `providers` и список `depends_on` — оба являются **мета-аргументами**, а не специфичными для модуля inputs — их поддерживает каждый блок module, независимо от того, что объявляет сам модуль.
* Никогда не поставляйте `terraform.tfstate`, `.terraform/` или `*.tfvars` вместе с исходным кодом модуля.
* Конвенция сообщества: `main.tf` / `variables.tf` / `outputs.tf` / `README.md` / `LICENSE` — ничего из этого не требуется самим Terraform.
* Предпочитайте **плоское** дерево модулей (один уровень потомков), связанное выражениями в root-модуле, глубокой вложенности.
