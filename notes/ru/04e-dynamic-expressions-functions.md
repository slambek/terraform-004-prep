# 4e. Write dynamic configuration using expressions and functions

## Named Values Reference Table

| Syntax | Refers to |
| :--- | :--- |
| `<RESOURCE_TYPE>.<NAME>` | Управляемый ресурс (полное поведение см. в 4b). |
| `var.<NAME>` | Input-переменная — автоматически конвертируется под её type constraint. |
| `local.<NAME>` | Local-значение. |
| `module.<MODULE_NAME>` | Outputs дочернего модуля — объект (без `count`/`for_each`), map (`for_each`) или list (`count`) объектов, повторяя форму ссылки на ресурс (4b). |
| `data.<TYPE>.<NAME>` | Data resource (см. 4a). |
| `path.module` | Путь в файловой системе к модулю, содержащему выражение. Избегайте в операциях записи — вызовы локальных модулей делят общие каталоги исходников, что рискует состояниями гонки. |
| `path.root` | Путь в файловой системе к root-модулю конфигурации. |
| `path.cwd` | Изначальная рабочая директория до любого `-chdir`. По возможности предпочитайте `path.root`/`path.module`. |
| `terraform.workspace` | Имя текущего выбранного workspace. |

⚠️ Это **не настоящие объекты** — вы не можете итерировать сам `aws_instance` через `for`-выражение, и вы не можете заменить путь с точками на нотацию с квадратными скобками.

**Block-local значения** (валидны только внутри конкретных контекстов блока): `count.index` (в ресурсах `count`), `each.key` / `each.value` (в ресурсах `for_each`), `self` (в блоках `provisioner`/`connection`, а также в `precondition`/`postcondition`, ссылающихся на охватывающий ресурс — см. 4g).

**Portability warning:** `path.cwd` и `terraform.workspace` встраивают контекст о *где*/*как* запущен Terraform. Использование их напрямую в аргументе ресурса (например, "запекание" `terraform.workspace` в глобально уникальное имя) может сломаться, если модуль вызывается более одного раза, или запускается с другой машины/директории. Предпочтительнее принимать input-переменную `prefix` и позволить *вызывающей стороне* решать, выводить ли её из workspace.

## Conditional Expressions

```text
condition ? true_value : false_value
```

```hcl
locals {
  name = (var.name != "" ? var.name : random_id.id.hex)
}
```

Частые применения: вычисление резервного значения, переключение `count` между 0/1/N на основе булевого флага, или условное оставление опционального атрибута неустановленным (`null`, см. 4d).

```hcl
resource "aws_instance" "ubuntu" {
  count                       = var.high_availability ? 3 : 1
  associate_public_ip_address = count.index == 0 ? true : false
}
```

## Splat Expressions

`[*]` итерирует по списку, извлекая один атрибут из каждого элемента — полную механику для ресурсов/вложенных блоков см. в 4b. Общая форма: `list_expr[*].attribute`.

## `for` Expressions

Преобразует коллекцию в другой list, map или object:

```hcl
[for instance in aws_instance.web_app : instance.id]                 # list
{for k, device in aws_instance.example.device : k => device.size}    # map/object
```

Нужно всякий раз, когда атрибуты ресурса на основе `for_each` нужно превратить в плоский list-output (одного splat недостаточно для map — см. 4b), или когда нужно преобразовать ключи/значения одной коллекции в другую.

## Built-In Functions

```text
function_name(arg1, arg2, ...)
```

* Вы **не можете** написать собственные функции на языке конфигурации — только providers могут предоставлять кастомные **provider-defined functions**, вызываемые как `provider::<local_name>::<function>(...)`.
* Экспериментируйте интерактивно с `terraform console`.

### Frequently Used Functions

| Function | Purpose |
| :--- | :--- |
| `templatefile(path, vars_map)` | Рендерит файл шаблона, интерполируя плейсхолдеры `${key}` из переданного map — например, для динамической генерации user-data скрипта EC2. |
| `lookup(map, key, default)` | Извлекает значение из map по ключу с опциональным fallback-значением, если ключ отсутствует. |
| `file(path)` | Читает сырое содержимое файла как есть — **без интерполяции**; используйте только для файлов, не требующих модификации на каждый запуск (например, публичный SSH-ключ). |
| `slice(list, start, end)` | Возвращает под-список — `end` не включается. |
| `merge(map1, map2, ...)` | Объединяет maps, при конфликте побеждают ключи более поздних аргументов. |
| `regexall(pattern, string)` | Возвращает все совпадения regex — полезно внутри блоков `validation` (4g) для обеспечения правил допустимых символов. |
| `jsonencode(value)` | Сериализует любое значение в строку JSON — стандартный "аварийный выход" для значений типа `any` (4d). |
| `length(collection)` | Количество элементов — используется в outputs, условиях и проверках размера при `for_each`. |

### `templatefile` Example

```hcl
resource "aws_instance" "web" {
  user_data = templatefile("user_data.tftpl", {
    department = var.user_department
    name       = var.user_name
  })
}
```

Файл `.tftpl` ссылается на `${department}` и `${name}` — Terraform интерполирует их во время plan/apply, делая скрипт переиспользуемым для разных значений переменных.

## Exam Quick Facts

* Named values — **не настоящие объекты** — нельзя заменить нотацией с квадратными скобками, нельзя итерировать сам тип ресурса.
* `path.cwd` и `terraform.workspace` рискуют сломать переносимость модуля, если "запечь" их напрямую в аргументы ресурса — предпочтительнее input-переменная.
* Splat (`[*]`) работает на **списках**; используйте `for`-выражение, чтобы преобразовать `for_each` **map** в список или другой map.
* Вы не можете определять кастомные функции на HCL — только providers могут добавлять функции (`provider::name::function()`).
* `file()` никогда не интерполирует; `templatefile()` — интерполирует.
* `terraform console` — инструмент для экспериментов с поведением функций перед использованием в конфигурации.