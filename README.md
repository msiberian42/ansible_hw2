# Ansible Playbook: ClickHouse + Vector

## Описание

Данный Ansible playbook предназначен для автоматической подготовки серверов и установки двух компонентов:

* **ClickHouse** — сервер базы данных;
* **Vector** — агент для сбора и обработки логов.

Playbook состоит из двух plays:

1. `Install Clickhouse` — устанавливает ClickHouse на хосты группы `clickhouse`, запускает сервис и создаёт базу данных `logs`.
2. `Install Vector` — устанавливает Vector на хосты группы `vector`, разворачивает конфигурацию и systemd unit.

Playbook рассчитан на Linux-системы с systemd.

---

## Требования

На управляющей машине должны быть установлены:

* Ansible;
* SSH-клиент;
* доступ по SSH к целевым серверам.

На целевых серверах должны быть:

* SSH-сервер;
* Python;
* `sudo` для пользователя;
* systemd.

Пользователь, под которым Ansible подключается к серверу, должен иметь права на выполнение административных операций через `sudo`.

---

## Структура проекта

```text
.
├── site.yml
├── inventory
│   └── prod.yml
├── group_vars
│   ├── clickhouse
│   │   └── vars.yml
│   └── vector
│       └── vars.yml
└── templates
    ├── vector.toml.j2
    └── vector.service.j2
```

---

## Inventory

Playbook использует две группы хостов:

* `clickhouse` — серверы, на которых устанавливается ClickHouse;
* `vector` — серверы, на которых устанавливается Vector.

Пример `inventory/prod.yml`:

```yaml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_host: 51.250.12.14
      ansible_user: pavel
      ansible_python_interpreter: /usr/bin/python3

vector:
  hosts:
    clickhouse-01:
      ansible_host: 51.250.12.14
      ansible_user: pavel
      ansible_python_interpreter: /usr/bin/python3
```

Один сервер может одновременно находиться в обеих группах.

Проверить inventory можно командой:

```bash
ansible-inventory -i inventory/prod.yml --graph
```

---

## Переменные

### ClickHouse

Переменные ClickHouse находятся в:

```text
group_vars/clickhouse/vars.yml
```

Пример:

```yaml
---
clickhouse_version: "22.3.3.44"

clickhouse_packages:
  - clickhouse-client
  - clickhouse-server
  - clickhouse-common-static
```

### Параметры ClickHouse

| Переменная            | Описание                       | Пример                                                               |
| --------------------- | ------------------------------ | -------------------------------------------------------------------- |
| `clickhouse_version`  | Версия ClickHouse              | `22.3.3.44`                                                          |
| `clickhouse_packages` | Список устанавливаемых пакетов | `clickhouse-client`, `clickhouse-server`, `clickhouse-common-static` |

---

### Vector

Переменные Vector находятся в:

```text
group_vars/vector/vars.yml
```

Пример:

```yaml
---
vector_version: "0.34.0"
vector_arch: "x86_64"
vector_install_dir: "/opt/vector"
vector_config_dir: "/etc/vector"
```

### Параметры Vector

| Переменная           | Описание                       | Пример        |
| -------------------- | ------------------------------ | ------------- |
| `vector_version`     | Версия Vector                  | `0.34.0`      |
| `vector_arch`        | Архитектура дистрибутива       | `x86_64`      |
| `vector_install_dir` | Директория установки Vector    | `/opt/vector` |
| `vector_config_dir`  | Директория конфигурации Vector | `/etc/vector` |

---

## Что делает playbook

### ClickHouse

Play `Install Clickhouse` выполняет следующие действия:

1. Скачивает RPM-пакеты ClickHouse.
2. При ошибке скачивания использует альтернативную загрузку `clickhouse-common-static`.
3. Устанавливает пакеты ClickHouse.
4. Перезапускает сервис `clickhouse-server`.
5. Создаёт базу данных `logs`.

После выполнения на сервере должен быть доступен:

```bash
clickhouse-client
```

а сервис должен работать:

```bash
systemctl status clickhouse-server
```

---

### Vector

Play `Install Vector` выполняет следующие действия:

1. Скачивает архив Vector указанной версии.
2. Создаёт директорию установки.
3. Распаковывает Vector.
4. Создаёт символическую ссылку на бинарный файл:
   `/usr/local/bin/vector`.
5. Создаёт директорию конфигурации.
6. Создаёт директорию данных Vector.
7. Разворачивает конфигурацию из Jinja2-шаблона `vector.toml.j2`.
8. Разворачивает systemd unit из шаблона `vector.service.j2`.
9. При изменении конфигурации или systemd unit перезапускает Vector.

Проверить сервис можно:

```bash
systemctl status vector
```

Проверить установленную версию:

```bash
vector --version
```

---

## Шаблоны

Конфигурация Vector хранится в:

```text
templates/vector.toml.j2
```

Playbook разворачивает её на сервер:

```text
/etc/vector/vector.toml
```

Systemd unit хранится в:

```text
templates/vector.service.j2
```

и разворачивается в:

```text
/etc/systemd/system/vector.service
```

Для обоих файлов используется Ansible-модуль `ansible.builtin.template`.

При изменении конфигурации Vector вызывается handler:

```yaml
- name: Restart vector
  ansible.builtin.systemd:
    name: vector
    state: restarted
    enabled: true
    daemon_reload: true
```

Таким образом, после изменения template конфигурация автоматически применяется к сервису.

---

## Запуск playbook

Перед запуском рекомендуется проверить доступность серверов:

```bash
ansible all -i inventory/prod.yml -m ping
```

Проверить синтаксис playbook:

```bash
ansible-playbook -i inventory/prod.yml site.yml --syntax-check
```

Проверить playbook с помощью `ansible-lint`:

```bash
ansible-lint site.yml
```

### Обычный запуск

```bash
ansible-playbook -i inventory/prod.yml site.yml --become
```

Если sudo требует пароль:

```bash
ansible-playbook -i inventory/prod.yml site.yml --become --ask-become-pass
```

---

## Check mode

Перед применением изменений можно выполнить playbook в режиме проверки:

```bash
ansible-playbook -i inventory/prod.yml site.yml --check
```

В этом режиме Ansible пытается определить изменения, которые будут выполнены при обычном запуске.


---

## Запуск только ClickHouse

Поскольку ClickHouse и Vector находятся в отдельных inventory-группах, можно ограничить выполнение playbook:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit clickhouse --become
```

Проверка:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit clickhouse --check
```

---

## Запуск только Vector

Установить и настроить только Vector:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit vector --become
```

Проверка:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit vector --check
```

---

## Теги

Запуск отдельных компонентов выполняется с помощью `--limit`:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit clickhouse
```

или:

```bash
ansible-playbook -i inventory/prod.yml site.yml --limit vector
```


---

## Handlers

Playbook использует handlers для управления сервисами.

Для ClickHouse:

```text
Start clickhouse service
```

Handler перезапускает:

```text
clickhouse-server
```

Для Vector:

```text
Restart vector
```

Handler перезапускает:

```text
vector
```

Handler Vector вызывается при изменении конфигурации.

---

## Проверка результата

После выполнения playbook можно проверить ClickHouse:

```bash
systemctl status clickhouse-server
```

Проверить наличие базы:

```bash
clickhouse-client -q "SHOW DATABASES"
```

В списке должна присутствовать:

```text
logs
```

Проверить Vector:

```bash
systemctl status vector
```

и:

```bash
vector --version
```

Проверить конфигурацию Vector:

```bash
vector validate --config /etc/vector/vector.toml
```

