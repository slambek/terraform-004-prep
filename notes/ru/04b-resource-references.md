# 4b. Refer to resource attributes and create cross-resource references

## Basic Reference Syntax

```text
<RESOURCE_TYPE>.<NAME>            → a managed resource
data.<TYPE>.<NAME>                → a data resource
<RESOURCE_TYPE>.<NAME>.<ATTR>     → one exported/configured attribute
```

Любое имя, не совпадающее с одним из других паттернов именованных значений Terraform (`var.`, `local.`, `module.`, `data.`, `path.`, `terraform.`), интерпретируется как **ссылка на ресурс**.

Ссылка на атрибут другого ресурса — где угодно в выражении — это именно то, что создаёт **implicit dependency** (см. 4f) между ними.

## Shape of a Resource Reference Depends on `count`/`for_each`

| Resource has… | Reference value is… |
| :--- | :--- |
| Ни `count`, ни `for_each` | Единичный **объект** — доступ к атрибутам через точечную или скобочную нотацию. |
| `count` | **Список объектов** — по одному на instance. |
| `for_each` | **Map объектов** — с ключами по ключу `for_each`. |

## Nested Blocks and Attribute Access

Дано:

```hcl
resource "aws_instance" "example" {
  ami = "ami-abc123"
  ebs_block_device {
    device_name = "sda2"
    volume_size = 16
  }
  ebs_block_device {
    device_name = "sda3"
    volume_size = 20
  }
}
```

* Аргумент верхнего уровня: `aws_instance.example.ami`
* Экспортированный атрибут: `aws_instance.example.id`
* Все значения по повторяющимся вложенным блокам: **splat expression** — `aws_instance.example.ebs_block_device[*].device_name` возвращает список всех `device_name`.
* Вложенные блоки, идентифицируемые логическим ключом (тип блока, принимающий метку), доступны по индексу: `aws_instance.example.device["foo"].size`; чтобы получить map по всем keyed-блокам, используйте `for`-выражение: `{for k, device in aws_instance.example.device : k => device.size}`.

## Referencing Multi-Instance Resources

**`count`** (результат — список):

```hcl
aws_instance.example[*].id   # list of every instance's id
aws_instance.example[0].id   # just the first instance's id
```

**`for_each`** (результат — map, индексируемый по ключу, а не по позиции):

```hcl
aws_instance.example["a"].id            # id of the "a"-keyed instance
[for value in aws_instance.example : value.id]   # list of all ids
```

⚠️ Splat-выражения (`[*]`) работают только на **списках** — они **не** применяются напрямую к ресурсу `for_each` (который является map). Чтобы сделать splat для ресурса `for_each`, сначала преобразуйте его в список: `values(aws_instance.example)[*].id`.

## Resource Address Reference (Full Syntax)

**Resource address** идентифицирует ноль или более resource instance в любом месте дерева конфигурации:

```text
[module path][resource spec]
```

**Module path** — `module.<module_name>[<module index>]`. Отсутствие module path означает **root module**. Несколько сегментов `module.` указывают на вложенность: `module.foo[0].module.bar["a"]`. Адрес без resource spec (`module.foo`) адресует все ресурсы внутри этого модуля (instance).

**Resource spec** — `<resource_type>.<resource_name>[<instance index>]`. Без префикса module path это соответствует только ресурсам в **root module** (Terraform 0.12+; более ранние версии сопоставлялись с любым дочерним модулем — это поведение больше не актуально для экзамена).

**Index values:**

* `[N]` — числовой индекс от 0 в ресурс на основе `count`. Отсутствие индекса при `count > 1` означает *все* instance.
* `["INDEX"]` — строковый ключ в ресурс на основе `for_each`.

```hcl
resource "aws_instance" "web" {
  count = 4
}
```

* `aws_instance.web[3]` → только последний instance.
* `aws_instance.web` → все четыре instance.

## Values Not Yet Known (`(known after apply)`)

Некоторые значения атрибутов ресурса (например, сгенерированный уникальный ID) невозможно предсказать, пока реальная система фактически не создаст объект. Terraform представляет их как **заполнители неизвестных значений (unknown value placeholders)** во время plan — в выводе плана они показываются как `(known after apply)`.

Заметные эффекты неизвестных значений:

* `count` **не может** быть неизвестным — Terraform должен знать количество instance во время planning.
* Если аргументы блока `data` включают неизвестное значение, чтение этого data source **откладывается до apply** (см. 4a), и его результаты также неизвестны до этого момента.
* Неизвестное значение, присвоенное во input блока `module` или в `value` блока `output`, распространяет "неизвестность" на каждую ссылку на эту переменную/output.

## Sensitive Resource Attributes

Provider может пометить конкретные атрибуты ресурса как **sensitive** в своей схеме. Тогда Terraform:

* Показывает `(sensitive value)` вместо реального значения в выводе plan/apply.
* Также считает sensitive любое значение, **производное от** этого атрибута (Terraform v0.15+ — более ранние версии скрывали только сам атрибут, а не производные значения).
* **Требует**, чтобы вы явно пометили `output` как `sensitive = true`, если он раскрывает sensitive-атрибут ресурса — иначе Terraform выдаёт ошибку.
* Всё равно **фиксирует sensitive-значение в открытом виде в state** — sensitivity это защита отображения/UI, а не механизм шифрования (см. 4h о том, как правильно с этим работать).

## Exam Quick Facts

* Без `count`/`for_each` → ссылка это **объект**. С `count` → **список**. С `for_each` → **map**.
* Splat (`[*]`) работает на списках (включая ресурсы `count` и повторяющиеся вложенные блоки) — **не** напрямую на map `for_each` (вместо этого используйте `values(...)[*]`).
* Resource address — это `[module path][resource spec]`; отсутствие module path означает root module.
* `count` никогда сам не может быть неизвестным значением.
* Sensitive-атрибуты ресурса заставляют делать outputs sensitive, но всё равно хранятся в открытом виде в state.