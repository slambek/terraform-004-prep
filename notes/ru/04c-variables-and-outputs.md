# 4c. Use variables and outputs

## Why: The Module Interface

Variables, locals и outputs — это то, как модули общаются:

* **Variables** — входной интерфейс модуля (как аргументы функции).
* **Locals** — временные, ограниченные модулем именованные выражения (чтобы избежать повторного набора одного и того же выражения).
* **Outputs** — возвращаемые значения / экспортируемые данные модуля.

## Variable Block

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web server"
  default     = "t2.micro"
}
```

Ключевые аргументы (все опциональны, кроме самой метки):

| Argument | Purpose |
| :--- | :--- |
| `type` | Type constraint (см. 4d). Без `type` → принимается любой тип. |
| `default` | Делает переменную опциональной. Должен быть **литеральным значением** — не может ссылаться на другие объекты в конфигурации. Без default вызывающая сторона **обязана** передать значение (Terraform никогда не оставляет переменную неприсвоенной). |
| `description` | Документирует назначение — пишите с точки зрения **потребителя**. |
| `validation` | Один или несколько блоков `condition` + `error_message` — вычисляются **до генерации плана** (см. 4g). |
| `sensitive` | Скрывает значение из вывода CLI; любое выражение, ссылающееся на него, также становится sensitive. Всё равно хранится в открытом виде в state (см. 4h). |
| `nullable` | По умолчанию `true` — разрешает явно передавать `null`. Установите `false`, чтобы требовать non-null значение. |
| `ephemeral` | Terraform 1.10+. Значение доступно во время выполнения, но **полностью исключается из файлов state и plan** (см. 4h). |

Зарезервированные имена переменных, которые нельзя использовать: `source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals`.

## Locals Block

```hcl
locals {
  app_name = "${var.project_name}-${var.environment}"
}
```

* Ссылка через `local.<name>`.
* Local может ссылаться на другие locals (даже в пределах того же блока), пока нет циклической зависимости.
* Ограничен модулем — не является input, не экспортируется автоматически (чтобы выйти за пределы модуля, должен пройти через output).

## Output Block

```hcl
output "instance_ip_addr" {
  value       = aws_instance.server.private_ip
  description = "The private IP address of the main server instance."
}
```

Служит четырём целям: раскрытие атрибутов ресурса дочернего модуля родительскому, отображение значений в выводе CLI root-модуля, позволение другим конфигурациям читать root-outputs через `terraform_remote_state`, и передача данных в инструменты автоматизации.

| Argument | Purpose |
| :--- | :--- |
| `value` | **Обязателен.** Любое валидное выражение; результат сохраняется в state. |
| `description` | Документирует output, опять же с точки зрения потребителя. |
| `sensitive` | Скрывает значение в выводе CLI. **Обязателен**, если значение производно от sensitive-атрибута ресурса или sensitive-переменной. Всё равно фиксируется в state в открытом виде (`-json`/`-raw` для `terraform output` раскроет его в открытом виде в любом случае). |
| `ephemeral` | Terraform 1.10+, **только дочерние модули** — нельзя задать на outputs root-модуля. Исключает значение из файлов state/plan (см. 4h). |
| `depends_on` | Явная зависимость (редко — при использовании добавляйте комментарий с объяснением почему; см. 4f). |
| `precondition` | Валидирует значение output перед его раскрытием/сохранением (см. 4g). |

## Assigning Variable Values (Order of Precedence)

Terraform использует **последнее найденное значение**, примерно в таком порядке (более позднее переопределяет более раннее):

1. Переменные окружения — `TF_VAR_<name>`.
2. `terraform.tfvars` (загружается автоматически, если присутствует).
3. Файлы `*.auto.tfvars` (загружаются автоматически, в алфавитном порядке).
4. `-var-file=FILENAME` в командной строке (можно повторять).
5. `-var 'NAME=VALUE'` в командной строке (можно повторять, наивысший приоритет среди перечисленного).

Если у Terraform Community Edition всё ещё нет значения и нет default, он **запрашивает интерактивно** — если только не указан `-input=false`, в этом случае возникает ошибка.

Файлы `.tfvars` используют HCL-подобный синтаксис (или JSON), но **не могут содержать определения ресурсов или другие конструкции конфигурации** — только присвоения значений переменным.

## Querying Outputs

```bash
terraform output                 # all outputs, human-readable (redacts sensitive)
terraform output <name>          # a single output BY NAME — sensitive values are NOT redacted here
terraform output -raw <name>     # unquoted string, for piping into other commands
terraform output -json           # machine-readable — sensitive values NOT redacted here either
```

Terraform скрывает sensitive-outputs только во время операций `plan`/`apply`/`destroy` и при запросе **всех** outputs вместе — запрос конкретного output по имени, или использование `-json`, всегда раскрывает реальное значение.

## Exam Quick Facts

* Переменная без `default` **требует** значения от вызывающей стороны — Terraform никогда не запускается с неприсвоенной переменной.
* `default` должен быть литералом — ссылки на другие ресурсы/переменные не допускаются.
* Приоритет присвоения (низкий→высокий): env vars → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` → `-var`.
* `sensitive` на outputs **обязателен**, когда значение приходит из sensitive-атрибута ресурса или sensitive-переменной.
* `terraform output <name>` и `terraform output -json` **не** скрывают sensitive-значения — это делает только человекочитаемое представление "all outputs".
* `ephemeral` на outputs — только для дочерних модулей — никогда не валиден на output root-модуля.