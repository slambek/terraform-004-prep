# 3f. Destroy Terraform-managed infrastructure

## `terraform destroy` = a Convenience Alias

```bash
terraform destroy
```

в точности эквивалентно:

```bash
terraform apply -destroy
```

Она принимает почти все те же опции, что и `apply`, за исключением того, что она **не** принимает аргумент-файл плана, и она принудительно включает режим планирования "destroy" (см. 3d). Как и обычный `apply`, она запрашивает подтверждение (`yes`) перед продолжением, поскольку уничтожение необратимо.

## Previewing a Destroy Without Executing It

```bash
terraform plan -destroy
```

Генерирует **speculative destroy plan** — показывает точно, что было бы удалено, не выполняя этого. Полезно перед тем, как решиться на destroy, особенно в общих окружениях.

## Destroying Specific Resources

```bash
terraform destroy -target aws_instance.example
```

`-target` ограничивает уничтожение конкретным ресурсом (и всем, что от него зависит), а не всем workspace целиком. Та же оговорка про исключительное использование, что и для `plan`/`apply` (3d/3e) — предпочтительнее разбивать конфигурации, чем регулярно использовать `-target`.

## Removing a Resource Without Destroying Everything

Вам не нужен `terraform destroy`, чтобы удалить только *один* ресурс. Обычный workflow:

1. **Закомментируйте (или удалите) resource-блок** в конфигурации (а также всё, что на него ссылается, например значение output, указывающее на его атрибуты — его тоже нужно удалить/закомментировать, иначе `validate` завершится ошибкой).
2. Запустите `terraform apply`.
3. Terraform обнаруживает, что ресурса больше нет в конфигурации, и предлагает **уничтожить** только этот ресурс — оставляя всё остальное нетронутым.

Это стандартный способ постепенно сокращать workspace, в противовес полному сносу всего через `destroy`.

## When to Use Full `destroy`

* Ephemeral / короткоживущие окружения (dev-песочницы, тестовая инфраструктура CI) после завершения задачи.
* Вывод из эксплуатации целого workspace или окружения приложения.

Обычно не используется для долгоживущей production-инфраструктуры.

## Exam Quick Facts

* `terraform destroy` под капотом буквально является `terraform apply -destroy`.
* `terraform plan -destroy` предпросматривает destroy, не выполняя его (speculative destroy plan).
* Удаление одного ресурса из живого workspace = **удалить/закомментировать его блок + `terraform apply`** (Terraform предложит уничтожить только этот ресурс) — для этого `terraform destroy` **не** нужен.
* `-target` ограничивает destroy конкретными ресурсами — исключительное использование, не рутинное.