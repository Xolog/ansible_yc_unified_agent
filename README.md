# Ansible Role: YC Unified Agent

Ansible-роль для установки и настройки [Yandex Cloud Unified Agent](https://cloud.yandex.ru/docs/monitoring/concepts/data-collection/unified-agent/) в виде Docker-контейнера. Агент собирает системные метрики (CPU, RAM, диск, сеть) и отправляет их в Yandex Cloud Monitoring.

---

## Требования

- Ansible >= 2.12
- Docker и Docker Compose v2, установленные на целевом хосте
- Коллекция `community.docker` (`ansible-galaxy collection install community.docker`)
- Docker-сеть, указанная в `yc_unified_agent_network_name`, должна существовать заранее
- Сервисный аккаунт в Yandex Cloud с ролью `monitoring.editor` (при использовании IAM через метаданные инстанса)

---

## Переменные

### Основные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `yc_unified_agent_folder_id` | `"id-your-folder"` | ID каталога Yandex Cloud, в который отправляются метрики |

### Docker

| Переменная | По умолчанию | Описание |
|---|---|---|
| `yc_unified_agent_dockerize` | `true` | Запускать агент в Docker-контейнере |
| `yc_unified_agent_image` | `"cr.yandex/yc/unified-agent"` | Docker-образ агента |
| `yc_unified_agent_version` | `"latest"` | Тег образа |
| `yc_unified_agent_container_name` | `"yc_unified_agent"` | Имя контейнера |
| `yc_unified_agent_network_name` | `"prom_network"` | Внешняя Docker-сеть |

### Конфигурация

| Переменная | По умолчанию | Описание |
|---|---|---|
| `yc_unified_agent_conf_dir` | `"/var/lib/yc_unified_agent"` | Каталог для хранения конфигурационных файлов на хосте |
| `yc_unified_agent_address` | `"127.0.0.1"` | Адрес для прослушивания статус-порта |
| `yc_unified_agent_port` | `"16241"` | Порт статус-сервера агента |

### Тома и порты

| Переменная | По умолчанию | Описание |
|---|---|---|
| `yc_unified_agent_volumes` | см. ниже | Список томов для монтирования в контейнер |
| `yc_unified_agent_ports` | см. ниже | Список маппингов портов контейнера |

Значения по умолчанию для `yc_unified_agent_volumes`:

```yaml
yc_unified_agent_volumes:
  - "{{ yc_unified_agent_conf_dir }}/config.yml:/etc/yandex/unified_agent/config.yml"
  - "/proc:/ua_proc"
```

Значения по умолчанию для `yc_unified_agent_ports`:

```yaml
yc_unified_agent_ports:
  - "{{ yc_unified_agent_address }}:{{ yc_unified_agent_port }}:16241"
```

### Конфигурация агента

Переменная `yc_unified_agent_config` содержит конфигурацию агента, которая будет записана в `config.yml`. По умолчанию настраивает:

- **Storage** `main` — файловое хранилище буферизации (100 МБ, сегменты по 10 МБ)
- **Channel** `cloud_monitoring` — отправка метрик в Yandex Cloud Monitoring через IAM (cloud_meta)
- **Routes**:
  - `linux_metrics` — системные метрики с namespace `sys`
  - `agent_metrics` — метрики самого агента с namespace `ua` (только health)
- **Import** — подгрузка дополнительных конфигураций из `/etc/yandex/unified_agent/conf.d/*.yml`

Подстановка `FOLDER_ID` в конфиге происходит через конфиг.

---

## Пример использования

### Минимальный playbook

```yaml
- name: Install Yandex Cloud Unified Agent
  hosts: monitoring
  roles:
    - role: yc_unified_agent
      vars:
        yc_unified_agent_folder_id: "b1g8jvfde5s3xxxxxxxx"
```

### Расширенный пример с кастомной сетью и конфигом

```yaml
- name: Install Yandex Cloud Unified Agent
  hosts: monitoring
  roles:
    - role: yc_unified_agent
      vars:
        yc_unified_agent_folder_id: "b1g8jvfde5s3xxxxxxxx"
        yc_unified_agent_version: "24.03.02"
        yc_unified_agent_network_name: "my_custom_network"
        yc_unified_agent_conf_dir: /opt/yc_unified_agent
        yc_unified_agent_volumes:
          - "/opt/yc_unified_agent/config.yml:/etc/yandex/unified_agent/config.yml"
          - "/opt/yc_unified_agent/conf.d:/etc/yandex/unified_agent/conf.d"
```

### Добавление дополнительных метрик через conf.d

Смонтируйте директорию `conf.d` и добавьте туда файлы с дополнительными маршрутами:

```yaml
yc_unified_agent_volumes:
  - "{{ yc_unified_agent_conf_dir }}/config.yml:/etc/yandex/unified_agent/config.yml"
  - "{{ yc_unified_agent_conf_dir }}/conf.d:/etc/yandex/unified_agent/conf.d"
```

---

## Структура роли

```bash
yc_unified_agent/
├── defaults/
│   └── main.yml          # Переменные по умолчанию
├── tasks/
│   └── main.yml          # Создание директории, шаблона конфига, запуск контейнера
├── templates/
│   └── config.yml.j2     # Шаблон конфигурации агента
└── README.md
```

---

## Теги

Роль не определяет собственные теги. Для частичного применения используйте стандартные флаги Ansible (`--start-at-task`, `--step`).

---

## Известные ограничения

- Роль поддерживает только запуск агента в Docker (`yc_unified_agent_dockerize: true`). Установка агента напрямую на хост не реализована.
- Docker-сеть (`yc_unified_agent_network_name`) должна быть создана до запуска роли — роль не создаёт её автоматически.

---

## Лицензия

MIT

---

## Автор

Pavel Antsupov
