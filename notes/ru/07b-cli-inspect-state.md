# 7b. Use the CLI to inspect state

> **Scope note:** Назначение state, refresh/`-refresh-only`, drift и блоки
> `moved`/`removed` вместе с миграцией между state разобраны в 6d — этот
> файл про семейство подкоманд `terraform state` и связанные команды
> инспекции как повседневные инструменты, а не про рабочие процессы
> drift/рефакторинга, в которых они используются. Механика
> `terraform import` — в 7a.

## Core Concept

`terraform state <subcommand>` предоставляет продвинутую, дружелюбную к
скриптам инспекцию и модификацию state без ручного редактирования JSON
state. Все подкоманды работают одинаково с локальным или удалённым state —
удалённые чтение и запись просто требуют сетевого round trip'а.

## Subcommands

```
terraform state list
terraform state show ADDRESS
terraform state mv SOURCE DESTINATION
terraform state pull
terraform state push
terraform state rm ADDRESS
terraform state replace-provider
```

| Command | What it does |
|---|---|
| `list` | Перечисляет адреса ресурсов, отслеживаемых в state (только чтение) |
| `show ADDRESS` | Выводит атрибуты одного ресурса из state |
| `mv` | Переименовывает адрес ресурса или перемещает его между двумя файлами state |
| `pull` | Скачивает текущий state, выводит в stdout |
| `push` | Перезаписывает удалённый state локальным файлом (опасно, защищено проверками lineage/serial — см. 6d) |
| `rm` | Убирает ресурс из state, не уничтожая реальный объект |
| `replace-provider` | Обновляет адрес provider'а, записанный для ресурсов |

`terraform show` (без префикса `state`) даёт **читаемый человеком** дамп
всего state — или, если дан сохранённый файл плана, предыдущего state этого
плана — и поддерживает `-json` для машиночитаемого вывода.

## Design characteristics

- **Дружелюбность к командной строке**: вывод структурирован для передачи
  по конвейеру в `grep`, `awk` и подобные инструменты; предпочитайте
  цепочки этих команд прямому парсингу JSON state.
- **Автоматические бэкапы**: каждая подкоманда, которая *изменяет* state
  (`mv`, `rm`, `push`, `replace-provider`), сначала записывает файл бэкапа.
  Отключить это нельзя — удаляйте старые файлы бэкапа вручную, если они не
  нужны. Команды только для чтения (`list`, `show`, `pull`) **не** пишут
  бэкапы.
- Путь файла бэкапа контролируется через `-backup=FILENAME` (тот же
  механизм, что и legacy-флаг local backend в 6a).
- Удалённый state ведёт себя идентично локальному для всех подкоманд
  `state` — то же использование CLI, просто медленнее из-за сетевых
  вызовов.

## `terraform state mv` — rename or relocate

```bash
# Rename within the same state
terraform state mv aws_instance.old_name aws_instance.new_name

# Move into a different state file (legacy cross-state approach — full
# workflow with pull/push in 6d)
terraform state mv -state-out=../other/terraform.tfstate \
  aws_instance.example aws_instance.example
```

- Обновляет **только state** — вашей `.tf`-конфигурации это не касается.
  После перемещения вам нужно самостоятельно добавить/убрать
  соответствующие блоки resource, иначе следующий `plan` предложит
  уничтожить/пересоздать.
- Имена назначения должны быть уникальны в пределах целевого state; `mv`
  может переименовывать по ходу перемещения, чтобы избежать коллизий.

## Replacing a resource via CLI

> Полная семантика `-replace` (и её связь с устаревшим `terraform taint`)
> разобрана в **3d/3e** — здесь только практический рабочий процесс
> `state list` → `-replace` для поиска нужного адреса.

```bash
terraform state list                        # find the address
terraform plan  -replace="aws_instance.example"
terraform apply -replace="aws_instance.example"
```

## Exam Quick Facts

- Каждая подкоманда `state`, **изменяющая** state, всегда пишет бэкап; это
  нельзя отключить — можно только перенести его через `-backup`.
- Подкоманды только для чтения (`list`, `show`, `pull`) никогда не пишут
  бэкапы.
- `state mv` меняет **state**, никогда не конфигурацию — согласование
  дрейфа конфига после перемещения лежит на вас.
- `-replace` нужен **адрес** ресурса — `state list` — самый быстрый способ
  его найти (полная семантика `-replace`/`taint`: 3d/3e).
- `terraform show -json` может читать либо последний state, **либо**
  зафиксированный предыдущий state из сохранённого файла плана.
- Все подкоманды `terraform state` работают одинаково с локальным или
  удалённым state — единственное отличие — задержка round trip'а.
