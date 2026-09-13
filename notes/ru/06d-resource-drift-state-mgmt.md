# 6d. Manage resource drift and Terraform state

> **Scope note:** Синтаксис конфигурации backend — в 6c; локинг — в 6b;
> специфика local backend — в 6a. Этот файл покрывает назначение state,
> поведение refresh/drift, команды инспекции/рефакторинга state и блоки
> `moved`/`removed`.

## Core Concept

Terraform state — это запись, связывающая инстансы ресурсов вашей
конфигурации с реальными удалёнными объектами. Перед любым plan/apply
Terraform выполняет **refresh** — сверяет state с реальной инфраструктурой,
чтобы вычислить точный diff. «Drift» — это любое несоответствие между state
(или конфигом) и реальной инфраструктурой, вызванное изменениями, сделанными
**вне** Terraform.

## Why State Exists

- Отображает инстансы ресурсов конфигурации в реальные объекты (через ID).
- Отслеживает метаданные и повышает производительность на крупной
  инфраструктуре.
- Формат state — JSON, но **прямое редактирование файла не рекомендуется** —
  используйте подкоманды CLI `terraform state`, которые изолируют вас от
  изменений формата между версиями Terraform.
- Terraform ожидает строгое отображение **один-к-одному** между
  сконфигурированными инстансами ресурсов и удалёнными объектами. Нарушение
  этого (напр. через `terraform import` или `state rm`) — ваша
  ответственность привести в соответствие.

## Inspecting & Modifying State (CLI)

> Полный каталог подкоманд `terraform state` (`list`, `show`, `mv`, `pull`,
> `push`, `rm`, `replace-provider`) как повседневный инструментарий разобран
> в **7b**; `terraform import` полностью разобран в **7a**. Этот раздел
> покрывает только защитные механизмы, специфичные для миграции/работы с
> drift.

Защитные механизмы `terraform state push` (можно обойти через `-force`, не
рекомендуется без предварительного бэкапа):

- **Различающийся lineage** (уникальный ID, назначенный при создании state)
  → отклоняется, поскольку это подразумевает пуш несвязанного state.
- **Больший serial** у назначения → отклоняется, поскольку у назначения
  есть изменения, которых нет у вашего state.

## Refresh & Refresh-Only Mode

> Семантика флага `-refresh=false` и трёх взаимоисключающих режимов
> планирования (включая `-refresh-only`) разобраны в **3d** — этот раздел
> покрывает только то, как refresh-only вписывается в рабочий процесс с
> drift.

- Каждый `plan`/`apply` сначала выполняет **неявный refresh в памяти** (3d).
- `terraform plan -refresh-only` / `terraform apply -refresh-only` явно
  показывают drift: они показывают, во что изменился бы *state*, чтобы
  соответствовать реальной инфраструктуре, **не трогая реальную
  инфраструктуру или конфигурацию**.
- Устаревшая подкоманда `terraform refresh` делает то же обновление, но
  **молча перезаписывает state без возможности проверки** — `-refresh-only`
  безопаснее и предпочтительнее, поскольку вы можете просмотреть его и
  решить не применять.
- Refresh-only apply также может обновить **значения outputs** — это важно,
  если другие workspace'ы потребляют их через `terraform_remote_state`
  (см. 8d).

## Drift Detection & Remediation Workflow

1. `terraform plan -refresh-only` → просмотрите обнаруженные различия.
2. `terraform apply -refresh-only` → примите, обновив **только state**, чтобы
   он соответствовал реальности (без изменений инфраструктуры).
3. Если дрейфующий объект должен управляться конфигом, добавьте
   соответствующий блок resource, затем `terraform import RESOURCE.NAME id`,
   чтобы его привязать.
4. После этого запустите обычный `terraform plan`/`apply`, чтобы согласовать
   конфиг с теперь уже точным state (например, повторно прикрепить security
   group, о которой Terraform больше не знал).

## Refactoring State — Splitting Configuration

Причины для разделения state: долгие apply, ресурсы с разной частотой
жизненного цикла, ресурсы, переходящие к другой команде, или извлечение
переиспользуемого модуля. Рекомендация по группировке: разделяйте по
изменчивости/частоте изменений, stateful vs. stateless и границам владения
команд.

Перед миграцией определите **межресурсные зависимости** — предпочитайте
динамические ссылки хардкодингу:

- Собственный data source провайдера (напр. поиск `aws_vpc`).
- Data source `tfe_outputs` для кросс-workspace outputs в HCP
  Terraform/Enterprise.
- Data source `terraform_remote_state` для любого другого удалённого или
  локального backend (требует явных прав доступа к workspace в HCP
  Terraform — см. 8d для полного сравнения с `tfe_outputs`).
- `terraform graph`, чтобы визуализировать зависимости, прежде чем что-либо
  отрезать.

### Migration approach 1 — `removed` + `import` blocks (recommended, Terraform ≥ 1.7)

Предпочтителен, поскольку блоки оставляют управляемую конфигом запись о
перемещении.

В **исходной** конфигурации:

```hcl
removed {
  from = aws_instance.example
  lifecycle {
    destroy = false   # keep the real object; just drop it from state
  }
}
```

Запустите `plan`/`apply` — это уберёт привязку из state **без** уничтожения
ресурса.

В **целевой** конфигурации:

```hcl
resource "aws_instance" "example" {
  instance_type = "t3.micro"
  ami           = data.aws_ami.example.id
}

import {
  id = "i-07b510cff5f79af00"
  to = aws_instance.example
}
```

Запустите `plan`/`apply` — это привяжет существующий объект к новому state
без его пересоздания. Terraform может даже автоматически сгенерировать для
вас блок resource из блока `import` через
`terraform plan -generate-config-out=FILE` (Terraform ≥ 1.5), но
сгенерированную конфигурацию нужно просмотреть, закоммитить и заново
выполнить plan — применить напрямую сгенерированный конфиг нельзя.

### `removed` block reference

```hcl
removed {
  from = aws_instance.example
  lifecycle {
    destroy = true   # default: also destroys the real resource
  }
}
```

- `from` (обязателен) — адрес ресурса, который нужно убрать из state.
- `lifecycle.destroy` — `true` (по умолчанию) также уничтожает реальный
  объект; `false` только убирает привязку в state, передавая объект куда-то
  ещё.
- Поддерживает блоки `connection` / `provisioner`, но здесь валидны
  **только provisioner'ы времени destroy**, и `when` обязателен для любого
  provisioner'а, вложенного в блок `removed`.

### `moved` block reference

Переименовывает/перемещает адрес ресурса **без** его уничтожения —
используется для рефакторинга на месте (напр. переименование ресурса,
перемещение его в модуль), а не для миграции между state.

```hcl
moved {
  from = aws_instance.a
  to   = aws_instance.b
}
```

Terraform ищет в state объект по адресу `from`, переименовывает его в `to` и
строит план относительно **нового** адреса — так что destroy/recreate не
происходит.

### Migration approach 2 — `terraform state mv` (legacy)

```bash
terraform state pull > source.tfstate
terraform state pull > destination.tfstate     # run from destination dir

terraform state mv \
  -state source/source.tfstate \
  -state-out destination/destination.tfstate \
  aws_instance.example aws_instance.example

terraform state push source.tfstate            # from source dir
terraform state push destination.tfstate       # from destination dir
```

- Работает напрямую с локальными файлами state, если используется `local`
  backend; с удалённым backend нужно сначала `pull` оба state вниз, а после
  — `push` оба обратно наверх.
- Требует Terraform ≥ 1.0. Рискованнее, чем `removed`+`import` (ручной
  pull/push несёт некоторый риск повредить удалённый state) — HashiCorp
  рекомендует `removed`/`import` для новых миграций.
- После перемещения обновите оба конфига, чтобы они соответствовали (уберите
  из источника, добавьте в назначение), и выполните `plan` для каждого,
  чтобы подтвердить **ноль** запланированных изменений перед слиянием.

## Exam Quick Facts

- `-refresh-only` **никогда** не изменяет реальную инфраструктуру — только
  файл state (и outputs). Обычный неявный refresh в `plan`/`apply` — это
  отдельная вещь.
- Подкоманда `refresh` устарела в пользу `-refresh-only`, потому что она
  перезаписывает state **без возможности предварительного просмотра**.
- `removed { lifecycle { destroy = false } }` убирает ресурс из state **без**
  удаления реального объекта — это ключевой edge case «НЕ уничтожает», о
  котором нужно помнить.
- `moved` — для **переименования/перемещения внутри одного и того же
  state**; это не способ переместить ресурс в *другой* файл state.
- `terraform state mv` между файлами state требует ручных `pull`/`push` на
  удалённом backend; это legacy — HashiCorp рекомендует вместо этого
  `removed`+`import` для новой работы.
- Защиты `terraform state push` (несовпадение `lineage`, больший `serial`)
  можно обойти через `-force`, что явно не рекомендуется без
  предварительного бэкапа через `state pull`.
- Автоматически сгенерированный конфиг из `import -generate-config-out`
  **нельзя** применить напрямую — его нужно просмотреть/закоммитить и заново
  выполнить plan.
