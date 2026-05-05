# Лабораторная работа: Структура конфигурации Ansible, инвентаризация и шаблоны хостов

## Цель работы
Изучить структуру конфигурации Ansible, научиться создавать и использовать статический инвентарь, освоить различные шаблоны хостов и групп для гибкого управления целевыми узлами.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста (Managed Nodes)** — Linux Debian 12

### Схема сети (пример):

```
Control Node:    (hostname: ansible-controller)
Managed Node1:   (hostname: server1)
Managed Node2:   (hostname: server2)
```

---





### Шаг 1. Создание рабочей директории

```bash
mkdir -p ~/ansible-lab/inventories/{production,staging}
cd ~/ansible-lab
```

---

## Часть 1. Структура конфигурации Ansible 

### Шаг 1.1. Создание конфигурационного файла

**Создайте файл `ansible.cfg` в корне проекта:**

```bash
nano ~/ansible-lab/ansible.cfg
```

**Содержимое:**

```ini
[defaults]
# Путь к инвентарю по умолчанию
inventory = ./inventories/production/hosts

# Отключение проверки SSH ключей (только для лабораторной)
host_key_checking = False

# Пользователь по умолчанию
remote_user = ansible

# Количество параллельных процессов
forks = 5

# Таймаут соединения
timeout = 10

# Включение цветного вывода
force_color = 1

# Путь к ролям
roles_path = ./roles

# Стратегия сбора фактов
gathering = smart

# Включение логирования
log_path = ./ansible.log

[ssh_connection]
# Ускорение SSH-соединений
pipelining = True

[colors]
# Настройка цветов вывода
highlight = white
verbose = blue
error = red
debug = dark gray

[diff]
# Настройка отображения различий
always = no
context = 3
```

### Шаг 1.2. Проверка приоритетов конфигурации

```bash
# Проверка, какой файл конфигурации используется
ansible --version | grep "config file"

# Просмотр всех текущих настроек
ansible-config dump

# Просмотр значения конкретного параметра
ansible-config dump | grep DEFAULT_HOST_KEY_CHECKING
```

### Шаг 1.3. Эксперимент с приоритетами

```bash
# Создание локального конфига в рабочей директории (переопределит глобальный)
echo "[defaults]" > ~/ansible-lab/ansible.cfg.local
echo "forks = 20" >> ~/ansible-lab/ansible.cfg.local

# Временное использование другого конфига через переменную окружения
ANSIBLE_CONFIG=~/ansible-lab/ansible.cfg.local ansible-config dump | grep DEFAULT_FORKS

# Проверка текущего значения forks
ansible-config dump | grep DEFAULT_FORKS
```

---

## Часть 2. Создание инвентаризации 

### Шаг 2.1. Создание production инвентаря

**Создайте файл `inventories/production/hosts`:**

```bash
mkdir -p ~/ansible-lab/inventories/production
nano ~/ansible-lab/inventories/production/hosts
```

**Содержимое (формат INI):**

```ini
# Production инвентарь



# Группа веб-серверов (используем реальные хосты)
[webservers]
web-server ansible_host=ip server 1

[database]
db-server ansible_host=ip server2

# Группа с переменными
[app_servers]
server1 ansible_host=ip server 1 app_port=8080
server2 ansible_host=ip server 2 app_port=8081

[app_servers:vars]
app_user=app_admin
app_home=/opt/app

# Вложенные группы
[uk:children]
webservers
app_servers


[uk:vars]
ntp_server=europe.pool.ntp.org
timezone=Europe/London

# Переменные для всех хостов
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=ansible
```

**Примечание:** В текущей лабораторной у нас только 2 managed хоста (`ip server 1` и `ip server 2*`).  В реальной работе используйте актуальные IP-адреса ваших серверов.

### Шаг 2.2. Создание staging инвентаря

**Создайте файл `inventories/staging/hosts`:**

```bash
mkdir -p ~/ansible-lab/inventories/staging
nano ~/ansible-lab/inventories/staging/hosts
```

**Содержимое:**

```yaml
# Staging инвентарь в формате YAML
all:
  children:
    webservers:
      hosts:
        web-stg01:
          ansible_host: ip server 1
          ansible_port: 2222  # нестандартный порт для теста
        web-stg02:
          ansible_host:ip server 2
    databases:
      hosts:
        db-stg01:
          ansible_host: ip server 2
    databases:
    staging:
      children:
        webservers
        databases
      vars:
        environment: staging
        debug_mode: true
        backup_enabled: false
```

### Шаг 2.3. Использование диапазонов и wildcards

**Создайте демонстрационный файл `inventories/demo-hosts.ini`:**

```bash
mkdir -p ~/ansible-lab/inventories/demo
nano ~/ansible-lab/inventories/demo/hosts.ini
```

**Содержимое:**

```ini
# Демонстрация диапазонов
[demo_range]
web[01:10].example.com
192.168.x.[1:200]

# Демонстрация wildcards
[demo_wildcard]
dc1-*.example.com
*.internal.domain

# Смешивание диапазонов и wildcards
[demo_complex]
cache-[a:f].prod-[01:05].local
```

### Шаг 2.4. Добавление переменных в инвентарь

**Создайте файлы переменных:**

```bash
mkdir -p ~/ansible-lab/inventories/production/group_vars
mkdir -p ~/ansible-lab/inventories/production/host_vars
```

**Создайте `group_vars/all.yml`:**

```bash
nano ~/ansible-lab/inventories/production/group_vars/all.yml
```

```yaml
---
# Переменные для всех хостов
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
default_user: ansible
python_interpreter: /usr/bin/python3
```

**Создайте `group_vars/webservers.yml`:**

```bash
nano ~/ansible-lab/inventories/production/group_vars/webservers.yml
```

```yaml
---
# Переменные для веб-серверов
web_package: nginx
web_port: 80
web_root: /var/www/html
max_clients: 500
```

**Создайте `host_vars/web-server.yml`:**

```bash
nano ~/ansible-lab/inventories/production/host_vars/web-server.yml
```

```yaml
---
# Переменные для конкретного хоста
app_version: 2.1.3
custom_port: 8080
```

### Шаг 2.5. Проверка инвентаря

```bash
cd ~/ansible-lab

# Просмотр списка хостов
ansible all -i inventories/production/hosts --list-hosts

# Просмотр всех переменных для хоста
ansible-inventory -i inventories/production/hosts --host web-server

# Графическое представление инвентаря
ansible-inventory -i inventories/production/hosts --graph

# Проверка доступности
ansible all -i inventories/production/hosts -m ping
```

---

## Часть 3. Шаблоны хостов и групп (30 минут)

### Шаг 3.1. Базовые шаблоны

```bash
cd ~/ansible-lab

# Проверка доступности всех хостов
ansible all -i inventories/production/hosts -m ping

# Проверка конкретного хоста
ansible web-server -i inventories/production/hosts -m ping

# Проверка группы
ansible webservers -i inventories/production/hosts -m ping
```

### Шаг 3.2. Оператор OR (логическое ИЛИ) — двоеточие

```bash
# Запуск на двух конкретных хостах
ansible web-server:db-server -i inventories/production/hosts -m shell -a "hostname"

# Запуск на группе и дополнительном хосте
ansible webservers:database -i inventories/production/hosts -m shell -a "hostname"
```

### Шаг 3.3. Оператор NOT (исключение) — восклицательный знак

```bash
# Все серверы, кроме db-server
ansible all:!db-server -i inventories/production/hosts -m ping

# Все веб-серверы, кроме конкретного
ansible webservers:!web-server -i inventories/production/hosts -m ping

# Все серверы, исключая группу database
ansible all:!database -i inventories/production/hosts -m ping
```

### Шаг 3.4. Оператор INTERSECTION (пересечение) — амперсанд

```bash
# Если у вас есть пересекающиеся группы, создайте тестовый инвентарь
cat > ~/ansible-lab/inventories/test-intersection.ini << 'EOF'
[production]
server1 ansible_host=ip server 1
server2 ansible_host=ip server2

[staging]
server2 ansible_host=ip server 2
server3 ansible_host=ip server3

[both:children]
production
staging
EOF

# Хосты, которые есть и в production, и в staging
ansible production:&staging -i ~/ansible-lab/inventories/test-intersection.ini --list-hosts
```

### Шаг 3.5. Комбинированные шаблоны

```bash
# Сложный шаблон: все хосты, кроме db-server, из группы webservers
ansible 'webservers:!db-server' -i inventories/production/hosts -m ping

# Пересечение webservers с database (маловероятно, но синтаксис важен):
ansible 'webservers:&database' -i inventories/production/hosts -m ping
```

### Шаг 3.6. Wildcard-шаблоны

```bash
# Создание тестового инвентаря с wildcard-именами
cat > ~/ansible-lab/inventories/wildcard-test.ini << 'EOF'
[test]
web01.example.com
web02.example.com
db01.example.com
db02.example.com
cache01.local
cache02.local
EOF

# Шаблон "web*" выберет все хосты, начинающиеся с web
ansible 'web*' -i ~/ansible-lab/inventories/wildcard-test.ini --list-hosts

# Шаблон "*.example.com" выберет все хосты в домене example.com
ansible '*.example.com' -i ~/ansible-lab/inventories/wildcard-test.ini --list-hosts

# Шаблон "*.local" выберет все хосты в домене local
ansible '*.local' -i ~/ansible-lab/inventories/wildcard-test.ini --list-hosts
```

### Шаг 3.7. Регулярные выражения

```bash
# Регулярное выражение для выбора хостов, начинающихся с web или db
ansible '~(web|db).*' -i ~/ansible-lab/inventories/wildcard-test.ini --list-hosts

# Регулярное выражение для выбора хостов, заканчивающихся на 01
ansible '~.*01.*' -i ~/ansible-lab/inventories/wildcard-test.ini --list-hosts
```

### Шаг 3.8. Индексация в группах

**Вернёмся к основному инвентарю:**

```bash
# Первый хост в группе webservers (индекс 0)
ansible webservers[0] -i inventories/production/hosts -m ping

# Последний хост в группе webservers
ansible webservers[-1] -i inventories/production/hosts -m ping

# Диапазон: первые два хоста
ansible webservers[0:2] -i inventories/production/hosts -m ping

# Диапазон: со второго и до конца
ansible webservers[1:] -i inventories/production/hosts -m ping
```

### Шаг 3.9. Использование `--limit` для переопределения

```bash
# Создадим простой плейбук для демонстрации --limit
cat > ~/ansible-lab/demo-playbook.yml << 'EOF'
---
- name: Демонстрация шаблонов хостов
  hosts: all
  gather_facts: no
  tasks:
    - name: Вывод имени хоста
      debug:
        msg: "Выполняется на {{ ansible_hostname }}"
EOF

# Запуск на всех, но ограничиваем одним хостом
ansible-playbook -i inventories/production/hosts demo-playbook.yml --limit web-server

# Запуск на всех, но исключаем db-server
ansible-playbook -i inventories/production/hosts demo-playbook.yml --limit 'all:!db-server'

# Использование файла с хостами для ограничения
echo "web-server" > ~/ansible-lab/limit-hosts.txt
ansible-playbook -i inventories/production/hosts demo-playbook.yml --limit @~/ansible-lab/limit-hosts.txt
```

---

