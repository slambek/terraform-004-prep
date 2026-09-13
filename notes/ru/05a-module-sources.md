# 5a. Explain how Terraform sources modules

## The `source` Argument

Каждый блок `module` требует аргумент `source`, сообщающий Terraform, **откуда** взять код модуля. Это должна быть **литеральная строка** — выражения и интерполяция не допускаются. Каждый раз при изменении `source` нужно заново запустить `terraform init` (или `terraform get`), чтобы Terraform заново получил код.

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

## Source Types

| Source type | Syntax | Notes |
| :--- | :--- | :--- |
| **Local path** | `./PATH` или `../PATH` | Файлы уже на диске. Terraform рассматривает это как прямую ссылку (без копирования) для относительных путей. Абсолютные пути (`/...` или буква диска) копируются в локальный кэш модулей — **избегайте абсолютных путей**, они привязывают конфигурацию к файловой структуре одной конкретной машины. |
| **Public Terraform Registry** | `<NAMESPACE>/<NAME>/<PROVIDER>` | напр. `hashicorp/consul/aws`. Это **основной способ делиться модулями** между конфигурациями/командами. |
| **Private registry (HCP Terraform/Enterprise)** | `<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>` | Такая же форма, как у публичного, с префиксом hostname, напр. `app.terraform.io/example_corp/vpc/aws`. `localterraform.com` — универсальный hostname, который всегда резолвится в тот экземпляр платформы, где выполняется конфигурация. |
| **GitHub** | `github.com/ORG/REPO` (HTTPS) или `git::github.com/ORG/REPO` (SSH) | Под капотом выполняет `git clone` — использует вашу локальную конфигурацию Git-credentials. |
| **Generic Git repo** | `git::ssh://...` или `git::https://...` | Любой Git URL. Поддерживает query-параметры `?ref=BRANCH_OR_TAG_OR_SHA` и `?depth=N` (shallow clone). |
| **Bitbucket** | `bitbucket.org/PATH` | Размещён на Git — те же механики клонирования/credentials, что и у generic Git. |
| **Mercurial repo** | `hg::PROTOCOL://...` | Выполняет `hg clone`; поддерживает фрагмент `#revision`. |
| **HTTP/HTTPS URL** | `https://...` | Terraform отправляет GET (`?terraform-get=1`) и ожидает либо заголовок ответа `X-Terraform-Get`, либо HTML-тег `<meta name="terraform-get" ...>`, указывающий на реальный источник — косвенная адресация через "vanity URL". Если сам URL заканчивается распознаваемым расширением архива (`.zip`, `.tar.gz` и т. д.), Terraform пропускает редирект и обрабатывает URL напрямую как архив. |
| **S3 bucket object** | `s3::https://BUCKET-URL/module.zip` | Объект должен быть архивом; credentials резолвятся через стандартную цепочку AWS SDK credential chain (env vars → shared credentials file → shared config → instance profile) или через inline query-параметры (`aws_profile` и т. д. — никогда их не коммитьте). |
| **GCS bucket object** | `gcs::https://www.googleapis.com/storage/v1/BUCKET/PATH` | Аутентификация по конвенциям Google Cloud SDK (`GOOGLE_OAUTH_ACCESS_TOKEN`, `GOOGLE_APPLICATION_CREDENTIALS`, GCE default credentials или `gcloud auth application-default login`). |

## Subdirectories Within a Source Package

Добавьте `//` после корня пакета, чтобы указать на поддиректорию внутри него — любые query-параметры (например, `ref=`) размещайте **после** сегмента поддиректории:

```hcl
module "consul" {
  source = "hashicorp/consul/aws//modules/consul-cluster"
}

module "vpc" {
  source = "git::https://example.com/network.git//modules/vpc?ref=v1.2.0"
}
```

Terraform скачивает/распаковывает **весь пакет** на локальный диск, но читает модуль только из указанной поддиректории — а это значит, что модули внутри поддиректорий одного и того же пакета могут ссылаться друг на друга обычными локальными путями.

## Duplicate Sources, Unique Labels

Вы можете использовать **один и тот же** `source` в двух или более отдельных блоках `module` — нужна только **уникальная метка** для каждого блока. Это позволяет создавать несколько по-разному сконфигурированных копий одного и того же модуля без `count`/`for_each` (см. 5c для этого).

## Exam Quick Facts

* `source` должен быть литеральной строкой — никогда не выражением и не ссылкой на переменную.
* Изменение `source` (или `version` — см. 5d) всегда требует повторного запуска `terraform init`.
* Формат Terraform Registry — `<NAMESPACE>/<NAME>/<PROVIDER>`; приватный реестр просто добавляет hostname спереди.
* `//` отмечает начало пути поддиректории внутри скачанного пакета; query-параметры (например, `ref=`) идут **после** него.
* Абсолютные локальные пути файловой системы нежелательны — они привязывают конфигурацию к одной машине.
* Повторное использование одного и того же `source` в нескольких блоках `module` допустимо, если метка каждого блока уникальна.
