# 4g. Validate configuration using custom conditions

## Four Ways to Validate, and When Each Runs

| Mechanism | Runs when | Blocks the operation? |
| :--- | :--- | :--- |
| **Input variable `validation`** | Немедленно, до того как Terraform сгенерирует план. | Да — выдаёт ошибку и останавливается. |
| **`precondition`** | После того как план уже существует, но до того как Terraform создаёт/читает охватывающий resource/data source/output. | Да. |
| **`postcondition`** | После planning **и** применения изменений к ресурсу (или чтения data source). | Да — останавливает дальнейшие нижестоящие действия, но **не** отменяет уже произошедшее. |
| **`check` block** | Самым последним шагом `plan` или `apply`, после всего остального. | **Нет** — сообщает предупреждение и продолжает. |

Все четыре требуют `error_message` (строковое выражение — подходят литералы, heredoc'и, `format()` и т.д.).

## Input Variable Validation

```hcl
variable "image_id" {
  type = string
  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI ID, starting with \"ami-\"."
  }
}
```

Выполняется, пока Terraform строит план — отлавливает неправильную конфигурацию до того, как её вообще увидит API provider'а, с более понятным сообщением, чем стандартная ошибка provider'а.

## Preconditions and Postconditions

Оба живут внутри блока `lifecycle` на блоке `resource`, `data` или `output`.

* **Precondition** — проверяет *предположение* до того, как Terraform что-то сделает. Preconditions имеют приоритет над ошибками аргументов provider'а — они выполняются первыми.

```hcl
resource "aws_instance" "example" {
  ami = data.aws_ami.example.id
  lifecycle {
    precondition {
      condition     = data.aws_ami.example.architecture == "x86_64"
      error_message = "The selected AMI must be for the x86_64 architecture."
    }
  }
}
```

* **Postcondition** — проверяет *гарантию* после того, как ресурс создан или data source прочитан. Внутри postcondition `self` ссылается на собственный результат охватывающего блока:

```hcl
data "aws_ami" "example" {
  id = var.aws_ami_id
  lifecycle {
    postcondition {
      condition     = self.tags["Component"] == "nomad-server"
      error_message = "tags[\"Component\"] must be \"nomad-server\"."
    }
  }
}
```

Блоки `output` поддерживают только `precondition` (нет `self`, поскольку у output нет отдельной идентичности, которую можно было бы проверить постфактум):

```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip
  precondition {
    condition     = length([for r in aws_security_group.web.ingress : r if r.to_port == 80]) > 0
    error_message = "Security group must allow HTTP ingress traffic."
  }
}
```

**Choosing between them:** используйте precondition для предположения, которое вы хотите проверить *до* создания целевого блока (помогает будущим мейнтейнерам понять ожидаемые входные данные); используйте postcondition для гарантии, которую нужно проверить *после* создания (помогает мейнтейнерам понять, что должно сохраняться). Если у ресурса много зависимостей, один postcondition на самом ресурсе зачастую полезнее, чем дублирование того же precondition на каждой зависимости.

## Check Blocks

Не привязаны к жизненному циклу какого-либо одного ресурса — валидируют поведение вашей инфраструктуры в целом, не блокируя запуск:

```hcl
check "health_check" {
  data "http" "terraform_io" {
    url = "https://www.terraform.io"
  }
  assert {
    condition     = data.http.terraform_io.status_code == 200
    error_message = "${data.http.terraform_io.url} returned an unhealthy status code"
  }
}
```

* Выполняется последним, после завершения plan/apply.
* Неудачный `assert` выдаёт **только предупреждение** — Terraform всё равно завершает операцию.
* Блок `data`, объявленный внутри `check`, ограничен этим check — вы **не можете** ссылаться на него в другом месте конфигурации.
* Terraform фиксирует результаты check (`pass`/`fail`) в **state-файле** под `check_results`.
* В HCP Terraform включение **health assessments** периодически перезапускает check-блоки (плюс preconditions/postconditions) как **continuous validation**, независимо от какого-либо plan/apply — полезно для отлова дрейфа или внешних сбоев (например, истечение сертификата) между запусками.

## Order of Evaluation (Exam-Relevant Sequence)

```text
1. Input variable validations   — before plan generation
2. Preconditions                — after plan, before create/read
3. Postconditions               — after apply (or data read)
4. Check blocks                 — last step of plan or apply
```

*Точный* момент выполнения preconditions/postconditions может сдвигаться в зависимости от того, известно ли ссылочное значение на момент plan или только после apply — если условие зависит от значения, известного только после apply (например, назначенный AWS ID тома), Terraform откладывает его проверку до этого момента.

## Exam Quick Facts

* Только блоки `check` не блокируют (предупреждение + продолжение); валидация переменных, preconditions и postconditions — все они **останавливают** операцию при неудаче.
* Precondition = проверка **до** действия; postcondition = проверка **после** действия; `self` валиден только внутри postconditions (и preconditions) на охватывающем блоке resource/data, но не в validation переменной.
* Блоки `output` поддерживают только `precondition` — никогда `postcondition`.
* На блок `data`, вложенный в блок `check`, нельзя сослаться нигде за пределами этого check.
* Порядок выполнения: validation переменных → preconditions → postconditions → checks (последними, не блокируют).
* Каждый механизм валидации требует `error_message`.