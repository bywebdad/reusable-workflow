# Reusable GitHub Actions Workflows

Переиспользуемые workflows для внутренних проектов: билд, деплой и эфемерные окружения для PR.

---

## Workflows

### `build-and-push.yml` — сборка и пуш Docker-образа в GHCR

### `deploy.yml` — деплой на Dokploy / Devtron / VDS

### `ephemeral-env.yml` — эфемерные preview-окружения для PR на Kubernetes

При открытии или обновлении PR собирает Docker-образ, пушит в GHCR с тегом по имени ветки, деплоит Helm-чартом в отдельный namespace на Kubernetes и оставляет комментарий с URL. При закрытии PR — удаляет Helm-релиз, namespace и образ из GHCR.

---

## Подключение ephemeral-env в другом проекте

**Шаг 1.** Создать файл `.github/workflows/preview.yml`:

```yaml
name: Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  preview:
    uses: YOUR_ORG/reusable-workflow/.github/workflows/ephemeral-env.yml@main
    with:
      project_name: my-app
      organization: YOUR_ORG
      helm_chart_path: ./deployment/my-chart
      base_domain: preview.example.com
      # cluster_name: general          # если имя кластера отличается от 'general'
      # build_method: dockerfile        # dockerfile (default) или compose
      # helm_extra_args: --values ./deployment/preview-values.yaml
    secrets: inherit
```

**Шаг 2.** Добавить секреты в репозиторий (или на уровне организации):

| Секрет | Описание |
|---|---|
| `DOCTL_TOKEN` | DigitalOcean API token — нужен для доступа к кластеру |
| `GHCR_TOKEN` | GitHub token со scope `write:packages` — сборка и удаление образов |

> `secrets: inherit` автоматически прокидывает все секреты репозитория и организации, дополнительно настраивать ничего не нужно.

**Шаг 3.** Убедиться, что Helm-чарт поддерживает эти values:

```yaml
# values.yaml (минимум для preview)
image:
  tag: ""          # будет подставлен slug ветки

ingress:
  host: ""         # будет подставлен slug.base_domain
```

Workflow передаёт `--set image.tag=<slug>` и `--set ingress.host=<slug>.<base_domain>`. Остальное — через `helm_extra_args`.

---

## Входные параметры ephemeral-env.yml

| Параметр | Обязательный | По умолчанию | Описание |
|---|---|---|---|
| `project_name` | ✅ | — | Имя проекта (GHCR path + Helm release name) |
| `organization` | ✅ | — | GitHub org/user (для GHCR) |
| `helm_chart_path` | ✅ | — | Путь к Helm-чарту в репо, напр. `./deployment/my-chart` |
| `base_domain` | ✅ | — | Базовый домен, напр. `preview.example.com` |
| `cluster_name` | ❌ | `general` | Имя кластера в DigitalOcean |
| `build_method` | ❌ | `dockerfile` | `dockerfile` или `compose` |
| `dockerfile_path` | ❌ | `Dockerfile` | Путь к Dockerfile |
| `build_context` | ❌ | `.` | Контекст сборки для docker build |
| `docker_compose_file` | ❌ | `docker-compose.yml` | Путь к compose-файлу |
| `compose_base_tag` | ❌ | `dev` | Тег, который создаёт compose (для перетегирования) |
| `helm_extra_args` | ❌ | `''` | Дополнительные аргументы для `helm upgrade` |
| `url_scheme` | ❌ | `https` | Схема URL в комментарии к PR |

---

## Что происходит при каждом событии

| Событие PR | Действие |
|---|---|
| `opened`, `synchronize`, `reopened` | Сборка образа → пуш в GHCR → `helm upgrade --install` → комментарий с URL |
| `closed` (merge или закрытие) | `helm uninstall` → `kubectl delete namespace` → удаление образа из GHCR → обновление комментария |

Имя ветки нормализуется в Kubernetes-совместимый slug: lowercase, только `a-z0-9-`, максимум 53 символа. Этот slug используется как имя Helm-релиза, namespace и тег образа.

Комментарий к PR создаётся один раз и обновляется при каждом пуше — не засоряет PR лишними сообщениями.

---

## Замечания

- `GHCR_TOKEN` должен принадлежать владельцу пакета (org или user) для удаления версий образа.
- Удаление образа из GHCR — best-effort: если версия не найдена, шаг пропускается без ошибки.
- Для compose-сборки workflow перетегирует образ с `compose_base_tag` → slug ветки. Убедитесь, что compose-файл строит образ с именем `ghcr.io/<org>/<project>:<compose_base_tag>`.
