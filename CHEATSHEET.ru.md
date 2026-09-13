# Terraform Associate (004) — Exam-Day Cheatsheet

Только то, что реально путают или спрашивают на тесте. За полным разбором —
ссылки на `notes/` в конце каждого блока.

---

## 1. Core Philosophy (IaC, providers, state — базовые понятия)

* **Declarative** (Terraform) = **WHAT**; **Imperative** (Bash/Python) = **HOW, step-by-step**.
* **Idempotency** ≠ **Immutability**: idempotent = повторный прогон даёт тот же результат; immutable = ресурс не модифицируется на месте, а пересоздаётся.
* **Day 0** (provisioning: VPC, VM, БД) → Terraform. **Day 1+** (OS-патчи, конфигурация приложений) → Ansible/Chef и т.п. Terraform может участвовать и там, и там, но это не его основная роль.
* Terraform Core **не знает** про AWS/Azure/K8s — это знание целиком в **provider-плагине**; Core общается с ним по **RPC**.
* Единственный **built-in провайдер** (питает `terraform_remote_state`) — `terraform.io/builtin/terraform`; не нуждается в `required_providers`, в отличие от всех остальных.
* Terraform **требует** state — это не опция. Три роли state: (1) маппинг config → реальный объект, (2) метаданные (зависимости, provider-конфиг), (3) кэш атрибутов для производительности (самая необязательная из трёх).
* Каждый реальный объект должен маппиться **ровно в один** resource instance в state.
* **Никогда** не редактировать state-файл вручную.

📄 [01a](notes/ru/01a-what-is-iac.md) · [01b](notes/ru/01b-advantages-of-iac.md) · [01c](notes/ru/01c-multicloud-hybrid.md) · [02b](notes/ru/02b-provider-usage.md) · [02d](notes/ru/02d-state-fundamentals.md)

---

## 2. Providers

* `required_providers` (в блоке `terraform {}`) = **какой провайдер и какие версии разрешены**. `provider {}` блок = **как он настроен** (регион, auth). `version` **никогда** не идёт внутрь `provider {}` — только в `required_providers`.
* Версия провайдера **отслеживается** в `.terraform.lock.hcl` (коммитить в VCS). `terraform init` без `-upgrade` переиспользует зафиксированную версию; `-upgrade` — единственное, что пересматривает более новые допустимые версии.
* Source address по умолчанию резолвится в `registry.terraform.io`, если hostname не указан.
* **Alias** (`provider "aws" { alias = "west" }`) = несколько **конфигураций одного и того же** провайдера. **Multiple providers** (2c) = несколько **разных** провайдеров в одной конфигурации. Это классическая ловушка на экзамене — не путать.
* Если **все** конфигурации провайдера имеют alias — Terraform создаёт **implied empty default**, что упадёт с ошибкой, если у провайдера есть обязательные аргументы.
* Terraform резолвит провайдер ресурса по **префиксу типа** (`aws_instance` → `aws`), поэтому named local name имеет значение.
* Cross-provider зависимости (напр. `random` → `aws`) резолвятся автоматически через единый resource graph — специальный синтаксис не нужен.

📄 [02a](notes/ru/02a-install-providers.md) · [02b](notes/ru/02b-provider-usage.md) · [02c](notes/ru/02c-multiple-providers.md)

---

## 3. Core CLI Workflow

* Порядок `terraform init`: **backend → child-модули → провайдеры → lock-файл**. Всегда безопасно перезапускать. Смена версии модуля/провайдера или backend-блока **требует** повторного `init`.
* `terraform validate` = только **синтаксис + внутренняя согласованность**, без API-вызовов и без реального контекста variable-значений. Требует инициализированную директорию (`init -backend=false` подойдёт). Останавливается на **первом классе** ошибок — ожидай несколько итераций.
* `terraform fmt` = только **стиль** (без опций кастомизации), не проверяет корректность против провайдера — это работа `validate`. Порядок: `fmt` → `validate` → `plan` → `apply`.
* `terraform plan`: **refresh → diff → propose**, ничего не меняет сам по себе. Три взаимоисключающих режима: **Normal** (default), **Destroy** (`-destroy`), **Refresh-only** (`-refresh-only`, только обновляет state/outputs под реальность, конфиг не трогает).
  * `-target` / `-replace` — **исключительное использование**, не рутина.
  * Без `-out` → **speculative plan** (только просмотр, для PR-ревью). С `-out` → план-файл, который можно применить **точно как есть**.
  * Сохранённые план-файлы могут содержать **sensitive-значения в открытом виде** — никогда не коммитить.
  * `-detailed-exitcode`: `0` нет изменений / `1` ошибка / `2` есть изменения.
* `terraform apply`: automatic-plan-mode **запрашивает подтверждение**; saved-plan-mode (`apply tfplan`) — **нет** (файл сам по себе — подтверждение), и туда нельзя добавить доп. флаги планирования. При ошибке в середине apply: **state обновляется частично успевшими изменениями**, лока снимается, **автоматического отката нет**.
* `terraform destroy` = буквально `terraform apply -destroy`. Убрать один ресурс, не разрушая всё — просто удали/закомментируй блок и сделай `apply`; `destroy` для этого не нужен.
* Флаг `-target` встречается в `plan`/`apply`/`destroy` — везде это «крайняя мера», предпочтительнее дробить конфигурацию.

📄 [03a](notes/ru/03a-terraform-workflow.md)–[03g](notes/ru/03g-terraform-fmt.md)

---

## 4. Configuration Language

**Resource vs Data:**
* `resource` = полный CRUD-lifecycle; `data` = **только чтение**, никогда не создаёт/не меняет.
* Data source read откладывается до **apply**, только если его аргументы зависят от значений, ещё не известных на plan (напр. зависит от изменяющегося managed-ресурса). `depends_on` на data-блоке **всегда** форсирует apply-time чтение (0.13+).

**References & shape:**
* Без `count`/`for_each` → ссылка это **объект**. С `count` → **список**. С `for_each` → **карта** (ключ ≠ индекс).
* Splat (`[*]`) работает на **списках** (включая `count`-ресурсы), **не** напрямую на `for_each`-картах — сначала `values(...)[*]`.
* Адрес ресурса: `[module path][resource spec]`; без module path — root module.

**Variables & Outputs:**
* Без `default` переменная **обязательна** для вызывающей стороны. `default` должен быть **литералом** — не может ссылаться на другие объекты.
* Приоритет присвоения значений (низкий → высокий): env vars (`TF_VAR_*`) → `terraform.tfvars` → `*.auto.tfvars` → `-var-file` → `-var`.
* `sensitive = true` **обязателен** на output, если значение производно от sensitive-ресурса/переменной — иначе ошибка. Но `terraform output <name>` и `-json` **не редактируют** sensitive-значения — редактируется только «все outputs разом» human-view.
* `ephemeral` (1.10+) на output — **только child-модули**, никогда root.

**Complex Types:**
* Collection types (`list`, `map`, `set`) — **один** тип элемента. Structural types (`object`, `tuple`) — **разные** типы по фиксированной схеме.
* Object ↔ map конверсия может быть **lossy** (лишние ключи отбрасываются).
* `optional(TYPE, DEFAULT)` подставляет DEFAULT и при **отсутствии**, и при явном **`null`** — единственный способ гарантировать non-null внутри модуля.
* `any` — placeholder, а не тип; используй только когда значение проходит насквозь без инспекции.

**Expressions & Functions:**
* Именованные значения (`var.`, `local.`, `path.*`, `terraform.workspace`) — **не реальные объекты**, нельзя итерировать через `for`.
* Нельзя писать свои функции в HCL — только provider-defined функции (`provider::name::func()`).
* `file()` никогда не интерполирует; `templatefile()` — интерполирует.

**Dependencies:**
* **Implicit** (ссылка на атрибут) — предпочтительный способ, без доп. синтаксиса. **Explicit** (`depends_on`) — только когда реальная зависимость невидима через ссылки; работает на `resource`/`module`/`data`/`output`. Стоит времени apply (убирает параллелизм).
* Порядок в `.tf`-файле **не влияет** на порядок выполнения — только граф зависимостей. Destroy идёт в **обратном** порядке от create.

**Validation (порядок — частый вопрос на экзамене):**
```
1. Input variable validation   — до генерации плана
2. Preconditions                — после плана, до create/read
3. Postconditions               — после apply/чтения data source
4. Check blocks                 — последний шаг plan/apply, НЕ блокирует
```
* Только `check`-блоки не блокируют операцию (warning + продолжение); все остальные — halt.
* `output` поддерживает только `precondition` (нет `self`, нет `postcondition`).
* `data`-блок внутри `check` нельзя референсить снаружи.

**Sensitive Data & Vault:**
* `sensitive = true` прячет значение только из **CLI/UI вывода** — в state/plan остаётся **plaintext**.
* Единственные способы **не хранить** значение вообще: `ephemeral` (variable/output/block) и write-only аргументы (`_wo` + `_wo_version`, 1.11+).
* Vault provider даёт **короткоживущие** credentials вместо статичных — но **не редактирует** ничего в state/plan; то, что прочитано из Vault, всё равно попадает в state в открытом виде.

📄 [04a](notes/ru/04a-resource-vs-data-blocks.md)–[04h](notes/ru/04h-sensitive-data-vault.md)

---

## 5. Modules

* `source` — обязательно **литеральная строка** (без выражений). Смена `source` **или** `version` всегда требует `terraform init`.
* Реестровый формат: `<NAMESPACE>/<NAME>/<PROVIDER>`. Приватный реестр = то же самое + hostname-префикс.
* **Версии модулей НЕ фиксируются в `.terraform.lock.hcl`** (в отличие от провайдеров!) — без точной версии (`=`, без `>=`/`~>`) можно получить разные версии на разных машинах.
* Каждый модуль — **изолированное пространство имён**: дочерний модуль не видит `var.*`/`local.*` родителя. Значения заходят **только** через аргументы `module`-блока, выходят **только** через `output`.
* **Провайдер-конфигурация — исключение**: наследуется **неявно** от вызывающего. Дочерние модули обычно **не должны** иметь свои `provider {}` блоки. Для non-default алиаса нужны **и** `providers` мета-аргумент у родителя, **и** `configuration_aliases` у дочернего модуля.
* `count`/`for_each` на `module`-блоке — взаимоисключающие, как и на `resource`/`data`.
* Никогда не поставлять с модулем: `terraform.tfstate`, `.terraform/`, `*.tfvars`.
* Рекомендация: **плоское** дерево модулей (один уровень вложенности), связанное выражениями из root-модуля.

📄 [05a](notes/ru/05a-module-sources.md)–[05d](notes/ru/05d-module-versions.md)

---

## 6. State, Backends & Drift

* **`local`** — backend по умолчанию (нет блока `backend` = local). Даёт **и** хранение, **и** locking (через файловые API), без внешнего сервиса.
* State locking **автоматический** для любой операции, которая может писать в state, но **опционален** — зависит от поддержки backend'ом. Неудачный lock **останавливает** операцию — не проходит "молча" без лока.
* `force-unlock LOCK_ID` — только для **своего** зависшего лока; чужой активный лок форсировать нельзя (риск коррупции).
* Только **один** `backend`-блок на конфигурацию; `backend` **не может** ссылаться на переменные/locals/data source. `cloud`-блок и `backend`-блок — **взаимоисключающие**.
* `-refresh=false` пропускает авто-refresh на `plan`/`apply`. `-refresh-only` — явный режим: показывает, что изменится **только в state**, не трогая инфраструктуру/конфиг. Deprecated `terraform refresh` делает то же самое, но **без возможности проверить перед применением** — используй `-refresh-only`.
* **`removed` + `import` блоки** (≥1.7) — рекомендуемый способ переноса ресурса между state-файлами (config-driven, оставляет след в конфиге). `terraform state mv` — legacy-альтернатива, требует ручного `pull`/`push` на remote backend.
  * `removed { lifecycle { destroy = false } }` — убирает из state **без удаления** реального объекта (ключевой edge case).
  * `moved` — переименование/перемещение **внутри одного** state, не кросс-стейт миграция.
* `terraform state push` защищён от **differing lineage** и **higher serial** на destination (обходится `-force`, не рекомендуется без бэкапа).
* Configuration drift (внешние изменения ломают конфиг) ≠ state drift (внешние изменения, не ломающие конфиг) — второе чинится через `-refresh-only`.

📄 [06a](notes/ru/06a-local-backend.md)–[06d](notes/ru/06d-resource-drift-state-mgmt.md)

---

## 7. Maintaining Infrastructure

* `terraform import` привязывает **ровно один** ресурс за вызов; нужен уже существующий (пусть и пустой) `resource`-блок в конфиге **до** импорта.
* Config-driven `import`-блок (≥1.5) + `plan -generate-config-out=FILE` безопаснее: можно посмотреть план заранее, и импорт+модификация происходят **за один шаг** — но сгенерированный конфиг **нельзя** применять напрямую, только после ревью/прунинга.
* Сложные импорты (напр. Network ACL) создают **несколько** state-записей — их нужно вручную отразить в конфиге, иначе Terraform запланирует **destroy**.
* `terraform state` подкоманды (`list`, `show`, `mv`, `pull`, `push`, `rm`, `replace-provider`) работают одинаково для local/remote state (разница только в сетевой задержке). Все **модифицирующие** подкоманды пишут бэкап автоматически — отключить нельзя, только сменить путь (`-backup`).
* `-replace` — современная рекомендованная замена deprecated `terraform taint`.
* `TF_LOG` — мастер-переключатель логов; `TF_LOG_CORE`/`TF_LOG_PROVIDER` — изолированно по компоненту; `TF_LOG_PATH` **сам по себе ничего не включает** — нужен ещё и один из уровней.
* Уровни логов (по убыванию): **TRACE → DEBUG → INFO → WARN → ERROR**. `TRACE` — для баг-репортов.
* 4 слоя диагностики, ближе к пользователю первыми: **язык (HCL) → state → core → provider**.

📄 [07a](notes/ru/07a-import-existing-infrastructure.md)–[07c](notes/ru/07c-verbose-logging.md)

---

## 8. HCP Terraform

* Три способа запускать runs: **VCS-driven** (основной), **CLI-driven** (через `cloud`-блок, стрим логов локально), **API-driven**.
* Workspace обрабатывает runs **строго по очереди**; кроме **plan-only runs** и **planning-стадии saved-plan** — они очередь игнорируют.
* **Speculative plan** — просмотровый, **никогда не применяется**. **Saved plan** — наоборот, **никогда не auto-apply**, даже если у workspace включён auto-apply; и автоматически отбрасывается, если state изменился до подтверждения.
* No-op план не запускает apply сам по себе — нужен явный режим **Allow empty apply** (обычно для апгрейда версии state-файла).
* Sentinel enforcement levels, от строгого к мягкому: **hard-mandatory → soft-mandatory → advisory**. Sentinel/OPA поддерживают только **workspaces**; только **Terraform policy** (нативный HCL-фреймворк) поддерживает и **Stacks**.
* Variable set конфликт по одному ключу решается **лексикографическим порядком имени сета** (не порядком создания!). Workspace-специфичная переменная всегда **побеждает** variable set; **priority variable set** побеждает почти всё, включая CLI-флаги.
* Health assessments требуют: последний run **успешен**, execution mode **Remote или Agent** (Local — не поддерживается), хотя бы один успешный apply в истории.
* Проекты: у каждого workspace/Stack **ровно один** проект. Права проекта по убыванию: **Admin → Maintain → Write → Read**. Org-level права "Manage Workspaces" всё равно кидают новые workspace в **Default Project**.
* Execution mode наследуется **workspace ← project ← organization**; смена на уровне проекта влияет только на **будущие** workspace.
* Run triggers требуют отдельный опт-ин "Auto-apply run triggers" — сам по себе триггер **не** авто-применяет запущенный run.
* Для кросс-workspace чтения outputs: `tfe_outputs` (рекомендуется для HCP Terraform/Enterprise, тянет только outputs через API) **предпочтительнее** `terraform_remote_state` (требует доступа ко **всему** state, плюс явного разрешения от source-workspace).
* `cloud`-блок не имеет `prefix`-аргумента (legacy-концепция `remote`-backend) — используй `tags`. `terraform login` — **только интерактивный**, для автоматизации нужны credentials вручную. `terraform import` **всегда** выполняется **локально**, даже с настроенной CLI-интеграцией.

📄 [08a](notes/ru/08a-hcp-terraform-create-infrastructure.md)–[08d](notes/ru/08d-cli-integration.md)