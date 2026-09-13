# 6a. Describe the local backend

> **Scope note:** Общий синтаксис блока backend, другие типы backend,
> частичная конфигурация и обработка credentials разобраны в 6c. Механика
> локов (force-unlock, lock ID) — в 6b.

## Core Concept

`local` backend — это **backend по умолчанию** в Terraform: если вы вообще
не объявляете блок `backend`, Terraform использует `local`. Он хранит state
как обычный JSON-файл на локальной файловой системе, выполняет все операции
(plan/apply) локально и блокирует state-файл, используя API файловой системы
на уровне ОС.

## Configuration

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

Поддерживаемые аргументы:

| Argument | Required | Description |
|---|---|---|
| `path` | No | Путь к state-файлу. По умолчанию — `terraform.tfstate` в root-модуле. |
| `workspace_dir` | No | Путь, используемый для хранения state не-default workspace'ов. |

Если вы вообще опускаете блок `backend`, это функционально идентично
объявлению `backend "local" {}` со значениями по умолчанию.

## Using local state as a data source

```hcl
data "terraform_remote_state" "foo" {
  backend = "local"

  config = {
    path = "${path.module}/../../terraform.tfstate"
  }
}
```

## Legacy CLI flags (local backend only)

Эти флаги появились раньше remote backend'ов и сохранены для обратной
совместимости. Они **влияют только на конфигурации, использующие `local`
backend** (или вообще без блока backend) — на любой другой тип backend они
не влияют.

- `-state=FILENAME` — переопределить файл, из которого Terraform читает
  предыдущий state.
- `-state-out=FILENAME` — переопределить файл, в который Terraform
  записывает новый state. Если вы задали `-state` без `-state-out`,
  Terraform переиспользует имя файла из `-state` и для вывода тоже,
  **перезаписывая входной файл**.
- `-backup=FILENAME` — переопределить автоматически сгенерированное имя
  файла резервной копии. Используйте `-backup=-`, чтобы полностью отключить
  резервные копии.

Использование всех трёх этих переопределений обходит обычный выбор имени
файла Terraform на основе workspace — если вы полагаетесь на несколько
workspace'ов, вам нужно будет самостоятельно подобрать разные имена файлов.

## Exam Quick Facts

- `local` — backend **по умолчанию** — отсутствие блока `cloud`/`backend` =
  local.
- Это один из немногих типов backend, предоставляющих **и** хранение, *и*
  локинг (через API системы/файлов), без необходимости во внешнем сервисе.
- `path` по умолчанию — `terraform.tfstate` относительно **root-модуля**.
- `-state`, `-state-out` и `-backup` — устаревшие флаги, работающие только
  с local backend — никогда не упоминайте их для remote backend'ов.
- Только `path` и `workspace_dir` — валидные аргументы в блоке
  `backend "local"`; больше ничего не поддерживается.
