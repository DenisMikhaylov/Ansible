# Лабораторная работа №3: Создание, проверка и выполнение рабочих книг Ansible

## Цель работы
Научиться создавать простые playbook, проверять их синтаксис, выполнять в режиме симуляции (check mode) и производить реальный запуск на целевых хостах.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12 (ansible-controller)
- **2 управляемых хоста (Managed Nodes)** — Linux Debian 12 (server1, server2)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

**Выполните на Control Node от пользователя ansible:**

```bash
mkdir -p ~/lab3-playbooks
cd ~/lab3-playbooks
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab3-playbooks/inventory.ini
```

**Содержимое файла:**

```ini
[webservers]
web-server ansible_host=ip server1

[databases]
db-server ansible_host=ip server2

[all:children]
webservers
databases
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab3-playbooks/ansible.cfg
```

**Содержимое файла:**

```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
remote_user = ansible
```

---

## Часть 2. Создание первого playbook

### Шаг 2.1. Создание простого playbook

Создайте файл `first-playbook.yml`:

```bash
nano ~/lab3-playbooks/first-playbook.yml
```

**Содержимое файла:**

```yaml
---
- name: Мой первый playbook
  hosts: all
  become: no
  
  tasks:
    - name: Шаг 1 - Вывод приветствия
      debug:
        msg: "Привет! Это выполняется на сервере {{ ansible_hostname }}"
    
    - name: Шаг 2 - Проверка uptime сервера
      command: uptime
      register: uptime_result
    
    - name: Шаг 3 - Показать uptime
      debug:
        msg: "Uptime сервера: {{ uptime_result.stdout }}"
    
    - name: Шаг 4 - Проверка версии ядра
      command: uname -r
      register: kernel_version
    
    - name: Шаг 5 - Показать версию ядра
      debug:
        msg: "Версия ядра: {{ kernel_version.stdout }}"
```

### Шаг 2.2. Проверка синтаксиса

```bash
ansible-playbook first-playbook.yml --syntax-check
```

**Ожидаемый вывод:**
```
playbook: first-playbook.yml
```

Если есть ошибки — исправьте их перед продолжением.

### Шаг 2.3. Просмотр целевых хостов

```bash
ansible-playbook first-playbook.yml --list-hosts
```

**Ожидаемый вывод:**
```
playbook: first-playbook.yml

  play #1 (all): Мой первый playbook    TAGS: []
    pattern: ['all']
    hosts (2):
      web-server
      db-server
```

### Шаг 2.4. Просмотр списка задач

```bash
ansible-playbook first-playbook.yml --list-tasks
```

**Ожидаемый вывод:**
```
playbook: first-playbook.yml

  play #1 (all): Мой первый playbook    TAGS: []
    tasks:
      Шаг 1 - Вывод приветствия    TAGS: []
      Шаг 2 - Проверка uptime сервера    TAGS: []
      Шаг 3 - Показать uptime    TAGS: []
      Шаг 4 - Проверка версии ядра    TAGS: []
      Шаг 5 - Показать версию ядра    TAGS: []
```

### Шаг 2.5. Выполнение в режиме симуляции (check mode)

```bash
ansible-playbook first-playbook.yml --check
```

**Ожидаемый вывод (пример):**

```
PLAY [Мой первый playbook] ***************************************************

TASK [Шаг 1 - Вывод приветствия] *********************************************
ok: [web-server] => {
    "msg": "Привет! Это выполняется на сервере web-server"
}
ok: [db-server] => {
    "msg": "Привет! Это выполняется на сервере db-server"
}

TASK [Шаг 2 - Проверка uptime сервера] ***************************************
ok: [web-server]
ok: [db-server]

TASK [Шаг 3 - Показать uptime] ***********************************************
ok: [web-server] => {
    "msg": "Uptime сервера: 10:30:15 up 1 day, 2:15,  2 users,  load average: 0.00, 0.01, 0.05"
}
ok: [db-server] => {
    "msg": "Uptime сервера: 10:30:15 up 2 days, 5:30,  1 user,  load average: 0.05, 0.02, 0.00"
}

TASK [Шаг 4 - Проверка версии ядра] ******************************************
ok: [web-server]
ok: [db-server]

TASK [Шаг 5 - Показать версию ядра] ******************************************
ok: [web-server] => {
    "msg": "Версия ядра: 6.1.0-13-amd64"
}
ok: [db-server] => {
    "msg": "Версия ядра: 6.1.0-13-amd64"
}

PLAY RECAP ********************************************************************
web-server                 : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
db-server                  : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

**Обратите внимание:** `changed=0` — значит, в режиме симуляции никаких изменений не было.

### Шаг 2.6. Реальное выполнение playbook

```bash
ansible-playbook first-playbook.yml
```

**Ожидаемый вывод:** аналогичен check mode, но может отличаться порядком выполнения.

---




## Часть 3. Использование опций выполнения (20 минут)

### Шаг 3.1. Запуск с ограничением хостов

```bash
# Запустить на одном хосте
ansible-playbook first-playbook.yml --limit web-server

# Запустить на первом хосте в группе
ansible-playbook install-playbook.yml --limit webservers[0]
```

**Проверьте:**

```bash
ansible-playbook first-playbook.yml --limit web-server --list-hosts
```

### Шаг 3.2. Пошаговый режим

```bash
ansible-playbook file-playbook.yml --step
```

**Работа с пошаговым режимом:**
- `y` — выполнить задачу
- `n` — пропустить задачу
- `a` — выполнить все оставшиеся задачи
- `p` — пропустить все оставшиеся задачи

### Шаг 3.3. Повторный запуск (идемпотентность)

```bash
# Ещё раз запустим file-playbook
ansible-playbook file-playbook.yml
```

**Ожидаемый результат:** Все задачи покажут `ok`, а не `changed`, так как файлы уже созданы и соответствуют желаемому состоянию.

```
TASK [Создание временной директории] ******************************************
ok: [web-server]
ok: [db-server]

TASK [Создание текстового файла] **********************************************
ok: [web-server]
ok: [db-server]

PLAY RECAP ********************************************************************
web-server                 : ok=4    changed=0    ...
db-server                  : ok=4    changed=0    ...
```

**Вывод:** Это демонстрация **идемпотентности** Ansible — повторный запуск не вносит изменений.

### Шаг 3.4. Запуск с увеличенной детализацией

```bash
# Базовый уровень
ansible-playbook first-playbook.yml -v

# Повышенный уровень
ansible-playbook first-playbook.yml -vv

# Максимальный уровень (для отладки)
ansible-playbook first-playbook.yml -vvv
```

---



| Команда | Назначение |
|---------|------------|
| `ansible-playbook playbook.yml --syntax-check` | Проверка синтаксиса |
| `ansible-playbook playbook.yml --list-hosts` | Показать целевые хосты |
| `ansible-playbook playbook.yml --list-tasks` | Показать список задач |
| `ansible-playbook playbook.yml --check` | Режим симуляции (dry run) |
| `ansible-playbook playbook.yml --limit host` | Выполнить только на host |
| `ansible-playbook playbook.yml --step` | Пошаговое выполнение |
| `ansible-playbook playbook.yml -v` | Подробный вывод |


6. Успешно автоматизировали создание файлов и установку ПО

**Готовность к следующей лабораторной:** Вы освоили базовый цикл работы с playbook и готовы к изучению переменных, циклов и условий.
