# 8d. Configure and use HCP Terraform integration

> **Scope note:** Что такое workspaces/projects/execution mode — 8c; что
> делают runs и режимы запуска — 8a; возможности совместной
> работы/управления (run triggers как концепция, variable sets, policy) —
> 8b. Этот файл про то, как подключить Terraform **CLI** к HCP Terraform и
> связать workspace'ы между собой по данным.

## Core Concept

Блок `cloud` в вашем блоке конфигурации `terraform` связывает локальную
рабочую директорию с одним или несколькими workspace HCP Terraform,
включая **рабочий процесс CLI-driven run**: локальные команды `terraform`
выполняются удалённо, а HCP Terraform управляет state, так что вам не нужно
поддерживать отдельный backend.

## Connecting the CLI

1. **Предоставьте credentials** — выполните `terraform login`
   (рекомендуется) или задайте токен пользователя напрямую в конфигурации.
2. **Добавьте блок `cloud`** (входит в блок `terraform`):

```hcl
terraform {
  cloud {
    organization = "my-org"
    hostname     = "app.terraform.io"   # optional, this is the default

    workspaces {
      project = "networking-development"
      tags = {
        layer  = "networking"
        source = "cli"
      }
    }
  }
}
```

   - `organization` — обязателен, должна уже существовать.
   - `workspaces.name` — привязать к одному существующему workspace по
     имени. **Взаимоисключим** с `tags`.
   - `workspaces.tags` — карта (или legacy-список только из ключей) тегов;
     Terraform связывается с любыми существующими workspace с совпадающими
     тегами или предлагает создать один с этими тегами при init, если
     совпадений нет.
   - `workspaces.project` — ограничивает сопоставление по тегу/имени
     workspace внутри именованного проекта.
   - Блок `cloud` **не** поддерживает аргумент `prefix` (в отличие от
     legacy-backend `remote`) — при миграции используйте вместо `prefix`
     `tags`.
3. **Запустите `terraform init`** — обязательно после добавления/изменения
   блока `cloud`.
4. **Мигрируйте state** (опционально, только если у предыдущего backend уже
   есть state).

### `terraform login`

```bash
terraform login [hostname]
```

- Только интерактивный — открывает браузер на машине, где запущен
  Terraform; для сценариев без участия человека/автоматизации используйте
  ручную настройку credentials.
- По умолчанию использует `app.terraform.io`, если hostname не указан;
  организации HCP Europe используют `app.terraform.io/eu`.
- По умолчанию записывает API-токен в **открытом виде** в
  `credentials.tfrc.json`; вы можете настроить credentials helper, чтобы
  хранить токены в другом месте (напр. в secrets manager вашей
  организации).
- Работает с любым сервером, реализующим протокол login, а не только с HCP
  Terraform/Terraform Enterprise.

## Migrating state

| Starting point | What to do |
|---|---|
| Local или `local`/другой backend, HCP Terraform ещё нет | Добавьте блок `cloud`, `terraform init`, подтвердите запрос миграции |
| Уже на legacy backend `remote` | Замените `backend "remote" {}` на `cloud {}` (продолжает использовать те же workspace) |
| Массовая миграция множества файлов state | Используйте CLI-инструмент `tf-migrate` (отдельная загрузка) |
| Доступ к CLI нежелателен | Мигрируйте напрямую через **API** HCP Terraform |

**Local → HCP Terraform (CLI):** `terraform init` обнаруживает новый блок
`cloud` и предлагает миграцию. Поскольку workspace HCP Terraform **обязаны**
иметь имя, Terraform может предложить переименовать CLI-workspace (которые
представляют окружения в рамках одного конфига) в отдельные workspace HCP
Terraform — распространённый паттерн:
`<COMPONENT>-<ENVIRONMENT>-<REGION>` (напр. `networking-prod-us-east`).

**`remote` backend → блок `cloud`:**

```hcl
terraform {
-  backend "remote" {
+  cloud {
     organization = "my-org"
     workspaces {
-      prefix = "my-app-"
+      tags = {
+        app = "mine"
+      }
     }
   }
}
```

После миграции конфигурации на основе `prefix` на теги, обращайтесь к
workspace по их **полному имени** через CLI (напр. `terraform workspace
select my-app-prod`, а не старой форме относительно префикса).

**Через API (вручную, скриптуемо):** закодируйте файл state в base64,
вычислите хеш MD5, выполните `POST` для создания workspace при
необходимости, **заблокируйте** его, отправьте `POST` со state (и MD5),
чтобы создать версию state, затем **разблокируйте** его.

**Требования для любой миграции:** Terraform ≥ 1.1 для блока `cloud`
(используйте backend `remote` на более старых версиях); сначала остановите
все операции Terraform, работающие с исходным state; мигрируйте только в
workspace, которые **никогда** не выполняли запуск.

## Excluding files from upload

CLI-driven remote plan/apply загружает копию вашей рабочей директории.
Добавьте файл `.terraformignore` в корень конфига, чтобы исключить пути
(правила в стиле `.gitignore`: `#`-комментарии, пустые строки
игнорируются, завершающий `/` для директорий, ведущий `!` для отрицания —
избегайте отрицания в больших деревьях, это вредит производительности). Без
этого файла Terraform всё равно по умолчанию исключает `.git/` и
`.terraform/` (кроме `.terraform/modules`).

## Import via CLI integration

`terraform import` **не** поддерживает удалённое выполнение — даже при
настроенном блоке `cloud` import всегда выполняется **локально**;
workspace лишь хранит итоговый state. Поскольку выполнение происходит
локально, переменные окружения workspace недоступны — любые credentials
провайдера, нужные для импорта, должны быть заданы в вашей локальной
оболочке.

## Cross-workspace data sharing

Два data source читают root-level outputs другого workspace:

| Data source | Notes |
|---|---|
| `terraform_remote_state` | Работает с local или любым удалённым backend; читающему workspace нужно явно предоставить доступ к state источника; предоставляет доступ ко **всему** этому state, а не только к outputs |
| `tfe_outputs` (из provider'а `tfe`) | **Рекомендуется** для HCP Terraform/Enterprise — получает только outputs через API, не требуя полного доступа к state |

Прежде чем любой из них заработает между workspace в HCP Terraform,
исходный workspace должен явно разрешить доступ потребителю через настройки
**remote state sharing**. Вызовы `tfe_outputs` также требуют `TFE_TOKEN`
(токен API команды или пользователя), заданный как переменная окружения в
потребляющем workspace, чтобы он мог обращаться к API. Этот механизм —
именно то, с чем обычно сочетается **run trigger** (8b): trigger ставит в
очередь downstream-запуск, а один из этих data source — то, как этот
запуск фактически читает значения upstream-workspace.

## Dynamic credentials

Альтернатива хранению долгоживущих облачных credentials как переменных
workspace. Использует **OIDC**: каждый запуск аутентифицируется в
облачном провайдере (AWS, GCP, Azure или Vault) с помощью подписанного
токена workload identity и получает взамен короткоживущие credentials на
каждый запуск — ничего статичного, что можно было бы утечь или нужно
ротировать.

- Сначала вы устанавливаете **доверительные отношения**: облачный провайдер
  настраивается на приём OIDC-токенов HCP Terraform (проверяется через
  TLS-сертификат / OIDC discovery HCP Terraform), ограниченных ролью и
  политикой.
- Условия доверия проверяют OIDC claims, в частности:
  - `aud` (audience) — уникальная строка, идентифицирующая эту
    конфигурацию провайдера, предотвращающая работу токена, предназначенного
    для одного провайдера, против другого.
  - `sub` (subject) — кодирует организацию, проект, workspace и фазу
    запуска (`organization:...:project:...:workspace:...:run_phase:plan|apply`),
    точно ограничивая, какой(ие) workspace могут использовать эти
    доверительные отношения.
- Включается для конкретного workspace через переменные окружения, напр.
  для Vault: `TFC_VAULT_PROVIDER_AUTH=true`, плюс переменные
  адреса/роли/namespace провайдера. У каждого поддерживаемого провайдера
  свой набор `TFC_<PROVIDER>_*`.
- Всегда ограничивайте claims вашей конкретной организацией/workspace —
  пропуск этого может позволить запускам другой организации HCP Terraform
  аутентифицироваться против ваших доверительных отношений.
- Изменение **имени** вашей организации/проекта/workspace может сломать
  существующие доверительные отношения, поскольку claim `sub` основан на
  имени.

## Exam Quick Facts

- `cloud { workspaces { name = ... } }` и `tags = { ... }`
  **взаимоисключимы** — выберите один режим адресации на конфигурацию.
- У блока `cloud` **нет аргумента `prefix`** — это legacy-концепция,
  специфичная только для backend `remote`; при миграции используйте
  вместо него `tags`.
- `terraform login` **только интерактивный**; для автоматизации нужно
  настраивать credentials вручную.
- `terraform import` **всегда выполняется локально**, даже при настроенной
  CLI-интеграции — переменные окружения workspace ему недоступны.
- Предпочитайте `tfe_outputs`, а не `terraform_remote_state` для
  кросс-workspace чтения в HCP Terraform/Enterprise — это не требует
  раскрытия всего исходного state.
- Dynamic credentials меняют статичные долгоживущие облачные ключи на
  короткоживущие, производные от OIDC credentials на каждый запуск —
  нечего хранить или ротировать как переменную workspace.
- Переименование организации/проекта/workspace может незаметно сломать
  доверительные отношения dynamic credentials, построенные на старом
  значении claim `sub`.
