# Лабораторная работа №13: Использование ролей Ansible

## Цель работы
Научиться создавать, структурировать и использовать роли Ansible для повторного использования кода, стандартизации конфигураций и упрощения управления комплексными проектами.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab13-roles/{playbooks,inventory,group_vars,host_vars}
cd ~/lab13-roles
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab13-roles/inventory/production.ini
```

```ini
[webservers]
web-server ansible_host=192.168.1.11

[databases]
db-server ansible_host=192.168.1.12

[all:children]
webservers
databases

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
ansible_become=yes
ansible_become_method=sudo

[webservers:vars]
http_port=80
https_port=443
app_name=mywebapp
```

### Шаг 1.3. Создание конфигурационного файла Ansible

```bash
nano ~/lab13-roles/ansible.cfg
```

```ini
[defaults]
inventory = ./inventory/production.ini
roles_path = ./roles
host_key_checking = False
remote_user = ansible
become = yes
become_method = sudo
become_user = root
retry_files_enabled = False
stdout_callback = yaml

[ssh_connection]
pipelining = True
```

### Шаг 1.4. Проверка доступности хостов

```bash
ansible all -m ping
```

**Ожидаемый результат:**
```
web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
db-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Часть 2. Создание роли Nginx (25 минут)

### Шаг 2.1. Создание каркаса роли с помощью ansible-galaxy

```bash
cd ~/lab13-roles
ansible-galaxy init roles/nginx
```

**Результат выполнения команды:**
```
roles/nginx/
├── README.md
├── .travis.yml
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── tests/
│   ├── inventory
│   └── test.yml
├── vars/
│   └── main.yml
├── templates/
└── files/
```

### Шаг 2.2. Настройка переменных по умолчанию

```bash
nano ~/lab13-roles/roles/nginx/defaults/main.yml
```

```yaml
---
# Переменные по умолчанию для роли nginx

# Версия Nginx
nginx_version: "latest"

# Порт для прослушивания
nginx_listen_port: 80

# Имя сервера
nginx_server_name: "localhost"

# Корневая директория
nginx_root_directory: "/var/www/html"

# Количество рабочих процессов
nginx_worker_processes: "auto"

# Максимальное количество соединений на воркер
nginx_worker_connections: 1024

# Скрывать версию Nginx
nginx_server_tokens: "off"

# Копировать кастомный index.html
nginx_custom_index: false

# Состояние сервиса
nginx_service_state: started
nginx_service_enabled: true
```

### Шаг 2.3. Настройка переменных роли (высокий приоритет)

```bash
nano ~/lab13-roles/roles/nginx/vars/main.yml
```

```yaml
---
# Высокоприоритетные переменные роли nginx

# Пути к конфигурационным файлам
nginx_config_path: "/etc/nginx/nginx.conf"
nginx_sites_available: "/etc/nginx/sites-available"
nginx_sites_enabled: "/etc/nginx/sites-enabled"
nginx_conf_d: "/etc/nginx/conf.d"

# Имя сервиса
nginx_service_name: "nginx"

# Пользователь и группа
nginx_user: "www-data"
nginx_group: "www-data"

# Формат логов
nginx_log_format: |
    '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"'
```

### Шаг 2.4. Создание обработчиков

```bash
nano ~/lab13-roles/roles/nginx/handlers/main.yml
```

```yaml
---
# Обработчики для роли nginx

- name: reload nginx
  ansible.builtin.systemd:
    name: "{{ nginx_service_name }}"
    state: reloaded
    daemon_reload: yes

- name: restart nginx
  ansible.builtin.systemd:
    name: "{{ nginx_service_name }}"
    state: restarted

- name: validate nginx config
  ansible.builtin.command: nginx -t
  register: nginx_validation
  changed_when: false
  listen: "validate nginx"
```

### Шаг 2.5. Создание задач роли

```bash
nano ~/lab13-roles/roles/nginx/tasks/main.yml
```

```yaml
---
# Основные задачи роли nginx

- name: Установка Nginx
  ansible.builtin.apt:
    name: "{{ 'nginx' if nginx_version == 'latest' else 'nginx=' + nginx_version }}"
    state: "{{ 'latest' if nginx_version == 'latest' else 'present' }}"
    update_cache: yes
  notify: restart nginx
  tags: [nginx, installation]

- name: Создание корневой директории веб-сервера
  ansible.builtin.file:
    path: "{{ nginx_root_directory }}"
    state: directory
    owner: "{{ nginx_user }}"
    group: "{{ nginx_group }}"
    mode: '0755'
  tags: [nginx, configuration]

- name: Копирование кастомной страницы index.html
  ansible.builtin.copy:
    src: index.html
    dest: "{{ nginx_root_directory }}/index.html"
    owner: "{{ nginx_user }}"
    group: "{{ nginx_group }}"
    mode: '0644'
  when: nginx_custom_index
  notify: reload nginx
  tags: [nginx, configuration]

- name: Настройка конфигурации Nginx
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "{{ nginx_config_path }}"
    owner: root
    group: root
    mode: '0644'
  notify:
    - validate nginx
    - reload nginx
  tags: [nginx, configuration]

- name: Настройка виртуального хоста по умолчанию
  ansible.builtin.template:
    src: default_site.conf.j2
    dest: "{{ nginx_sites_available }}/default"
    owner: root
    group: root
    mode: '0644'
  notify:
    - validate nginx
    - reload nginx
  tags: [nginx, configuration]

- name: Включение сайта по умолчанию
  ansible.builtin.file:
    src: "{{ nginx_sites_available }}/default"
    dest: "{{ nginx_sites_enabled }}/default"
    state: link
  notify: reload nginx
  tags: [nginx, configuration]

- name: Запуск и включение Nginx сервиса
  ansible.builtin.systemd:
    name: "{{ nginx_service_name }}"
    state: "{{ nginx_service_state }}"
    enabled: "{{ nginx_service_enabled }}"
    daemon_reload: yes
  tags: [nginx, service]
```

### Шаг 2.6. Создание шаблонов

**Основной конфиг Nginx:**

```bash
nano ~/lab13-roles/roles/nginx/templates/nginx.conf.j2
```

```nginx
# {{ ansible_managed }}
# Nginx конфигурация, управляемая Ansible
# Создано: {{ ansible_date_time.iso8601 }}
# Хост: {{ ansible_hostname }}

user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Формат логов
    log_format main {{ nginx_log_format }};

    access_log /var/log/nginx/access.log main;

    # Безопасность
    server_tokens {{ nginx_server_tokens }};

    # Основные настройки
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 10M;

    # GZip сжатие
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/rss+xml
        text/javascript;

    # Включение виртуальных хостов
    include {{ nginx_conf_d }}/*.conf;
    include {{ nginx_sites_enabled }}/*;
}
```

**Виртуальный хост по умолчанию:**

```bash
nano ~/lab13-roles/roles/nginx/templates/default_site.conf.j2
```

```nginx
# {{ ansible_managed }}
# Виртуальный хост по умолчанию

server {
    listen {{ nginx_listen_port }} default_server;
    listen [::]:{{ nginx_listen_port }} default_server;

    root {{ nginx_root_directory }};
    index index.html index.htm index.nginx-debian.html;

    server_name {{ nginx_server_name }};

    location / {
        try_files $uri $uri/ =404;
    }

    # Страница статуса Nginx
    location /nginx_status {
        stub_status;
        access_log off;
        allow 127.0.0.1;
        deny all;
    }

    # Обработка ошибок
    error_page 404 /404.html;
    location = /404.html {
        root {{ nginx_root_directory }};
        internal;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root {{ nginx_root_directory }};
        internal;
    }
}
```

### Шаг 2.7. Создание кастомной страницы

```bash
nano ~/lab13-roles/roles/nginx/files/index.html
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>Ansible Role Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 50px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: rgba(0,0,0,0.3);
            border-radius: 10px;
        }
        h1 { color: #ffd700; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Успешная настройка Nginx через Ansible Role!</h1>
        <p>Сервер: {{ ansible_hostname }}</p>
        <p>IP адрес: {{ ansible_default_ipv4.address }}</p>
        <p>Время: {{ ansible_date_time.iso8601 }}</p>
        <hr>
        <p>Этот сайт настроен с помощью роли Nginx</p>
    </div>
</body>
</html>
```

### Шаг 2.8. Настройка метаданных роли

```bash
nano ~/lab13-roles/roles/nginx/meta/main.yml
```

```yaml
---
galaxy_info:
  author: "Student"
  description: "Роль для установки и настройки Nginx на Debian 12"
  license: "MIT"
  min_ansible_version: "2.9"
  platforms:
    - name: Debian
      versions:
        - bookworm
  galaxy_tags:
    - web
    - nginx
    - webserver

dependencies: []
```

---

## Часть 3. Создание роли для базовой настройки сервера (15 минут)

### Шаг 3.1. Создание каркаса роли common

```bash
cd ~/lab13-roles
ansible-galaxy init roles/common
```

### Шаг 3.2. Настройка переменных по умолчанию

```bash
nano ~/lab13-roles/roles/common/defaults/main.yml
```

```yaml
---
# Переменные по умолчанию для роли common

# Часовой пояс
timezone: "Europe/Moscow"

# Базовые пакеты для установки
common_packages:
  - curl
  - wget
  - htop
  - git
  - vim
  - net-tools
  - ca-certificates
  - gnupg
  - lsb-release
  - software-properties-common

# Настройки sysctl
sysctl_settings:
  net.core.somaxconn: 1024
  net.ipv4.tcp_max_syn_backlog: 4096
  vm.swappiness: 10
  net.ipv4.ip_forward: 1
```

### Шаг 3.3. Создание задач роли common

```bash
nano ~/lab13-roles/roles/common/tasks/main.yml
```

```yaml
---
# Основные задачи роли common

- name: Обновление apt кэша
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
  tags: [common, packages]

- name: Установка базовых пакетов
  ansible.builtin.apt:
    name: "{{ common_packages }}"
    state: present
  tags: [common, packages]

- name: Настройка часового пояса
  ansible.builtin.timezone:
    name: "{{ timezone }}"
  tags: [common, configuration]

- name: Настройка sysctl параметров
  ansible.builtin.sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop: "{{ sysctl_settings | dict2items }}"
  tags: [common, configuration]

- name: Создание директории для ansible-файлов
  ansible.builtin.file:
    path: /opt/ansible
    state: directory
    owner: root
    group: root
    mode: '0755'
  tags: [common, configuration]

- name: Создание файла с информацией о настройке
  ansible.builtin.copy:
    content: |
      # Система настроена Ansible
      Дата: {{ ansible_date_time.iso8601 }}
      Хост: {{ ansible_hostname }}
      IP: {{ ansible_default_ipv4.address }}
      Роль: {{ role_name }}
    dest: /opt/ansible/provision_info.txt
    owner: root
    group: root
    mode: '0644'
  tags: [common, configuration]

- name: Отключение автозапуска ненужных сервисов
  ansible.builtin.systemd:
    name: "{{ item }}"
    enabled: no
    state: stopped
  loop:
    - bluetooth
    - cups
    - avahi-daemon
  ignore_errors: yes
  tags: [common, services]
```

### Шаг 3.4. Настройка метаданных роли common

```bash
nano ~/lab13-roles/roles/common/meta/main.yml
```

```yaml
---
galaxy_info:
  author: "Student"
  description: "Базовая роль для настройки Debian серверов"
  license: "MIT"
  min_ansible_version: "2.9"
  platforms:
    - name: Debian
      versions:
        - bookworm
  galaxy_tags:
    - system
    - base
    - common

dependencies: []
```

---

## Часть 4. Использование ролей в плейбуках (15 минут)

### Шаг 4.1. Создание плейбука с использованием секции `roles`

```bash
nano ~/lab13-roles/playbooks/site.yml
```

```yaml
---
- name: Базовая настройка всех серверов
  hosts: all
  become: yes
  gather_facts: yes

  roles:
    - role: common
      tags: [common, base]

- name: Настройка WEB серверов
  hosts: webservers
  become: yes
  gather_facts: yes

  roles:
    - role: nginx
      tags: [web, nginx]
      vars:
        nginx_listen_port: "{{ http_port }}"
        nginx_server_name: "{{ ansible_fqdn }}"
        nginx_custom_index: true
        nginx_root_directory: "/var/www/{{ app_name }}"
        nginx_worker_processes: 2
        nginx_worker_connections: 2048

- name: Настройка DB серверов (только базовая настройка ОС)
  hosts: databases
  become: yes
  gather_facts: yes

  tasks:
    - name: Информация о сервере базы данных
      ansible.builtin.debug:
        msg:
          - "Сервер БД настроен: {{ ansible_hostname }}"
          - "IP адрес: {{ ansible_default_ipv4.address }}"
          - "Дополнительная настройка БД не требуется в рамках этой лабораторной"
```

### Шаг 4.2. Создание плейбука с использованием `import_role`

```bash
nano ~/lab13-roles/playbooks/import_roles.yml
```

```yaml
---
- name: Настройка серверов с импортом ролей
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    # Статический импорт базовой роли (загружается один раз при старте)
    - name: Импорт базовой роли common
      ansible.builtin.import_role:
        name: common
      tags: [common]

    # Статический импорт роли Nginx для веб-серверов
    - name: Импорт роли Nginx
      ansible.builtin.import_role:
        name: nginx
      vars:
        nginx_listen_port: 80
        nginx_custom_index: true
        nginx_server_name: "{{ inventory_hostname }}"
      when: inventory_hostname in groups['webservers']
      tags: [nginx]

    # Задача для серверов БД
    - name: Настройка сервера БД
      ansible.builtin.debug:
        msg: "Сервер {{ ansible_hostname }} настроен как сервер баз данных (только базовая конфигурация ОС)"
      when: inventory_hostname in groups['databases']
```

### Шаг 4.3. Создание плейбука с использованием `include_role`

```bash
nano ~/lab13-roles/playbooks/include_roles.yml
```

```yaml
---
- name: Настройка серверов с динамическим включением ролей
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    # Динамическое включение базовой роли
    - name: Динамическое включение роли common
      ansible.builtin.include_role:
        name: common
      tags: [common]

    # Динамическое включение с несколькими конфигурациями Nginx
    - name: Динамическое включение роли Nginx для разных сайтов
      ansible.builtin.include_role:
        name: nginx
      vars:
        nginx_listen_port: "{{ item.port }}"
        nginx_server_name: "{{ item.server_name }}"
        nginx_root_directory: "/var/www/{{ item.site_name }}"
        nginx_custom_index: true
      loop:
        - { port: 80, server_name: "site1.local", site_name: "site1" }
        - { port: 8080, server_name: "site2.local", site_name: "site2" }
      when: inventory_hostname in groups['webservers']
      tags: [nginx]

    # Задача для сервера БД
    - name: Информация о сервере БД
      ansible.builtin.debug:
        msg: "Сервер {{ ansible_hostname }} ({{ ansible_default_ipv4.address }}) готов к установке СУБД"
      when: inventory_hostname in groups['databases']
```

### Шаг 4.4. Создание групповых переменных

```bash
nano ~/lab13-roles/group_vars/webservers.yml
```

```yaml
---
# Переменные для группы webservers

# Настройки Nginx
nginx_worker_processes: 2
nginx_worker_connections: 2048
nginx_server_tokens: "off"
nginx_custom_index: true
nginx_root_directory: "/var/www/myapp"

# Часовой пояс
timezone: "Europe/Moscow"
```

```bash
nano ~/lab13-roles/group_vars/databases.yml
```

```yaml
---
# Переменные для группы databases

# Часовой пояс для серверов БД
timezone: "Europe/Moscow"

# Дополнительные пакеты для серверов БД
extra_packages:
  - postgresql-client
  - mysql-client
```

### Шаг 4.5. Создание файла зависимостей для Ansible Galaxy

```bash
nano ~/lab13-roles/requirements.yml
```

```yaml
---
# Зависимости ролей из Ansible Galaxy

roles:
  # Официальная роль geerlingguy (для примера)
  - name: geerlingguy.nginx
    version: 3.1.0
    src: https://github.com/geerlingguy/ansible-role-nginx.git
    scm: git

  # Локальные роли
  - name: common
    src: ./roles/common

  - name: nginx
    src: ./roles/nginx
```

---

## Часть 5. Запуск и тестирование (10 минут)

### Шаг 5.1. Проверка синтаксиса плейбуков

```bash
cd ~/lab13-roles

# Проверка синтаксиса
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/import_roles.yml --syntax-check
ansible-playbook playbooks/include_roles.yml --syntax-check
```

### Шаг 5.2. Просмотр списка задач в плейбуке

```bash
# Просмотр всех задач (import_role видны, include_role - нет)
echo "=== Секция roles (site.yml) ==="
ansible-playbook playbooks/site.yml --list-tasks

echo -e "\n=== import_role (import_roles.yml) ==="
ansible-playbook playbooks/import_roles.yml --list-tasks

echo -e "\n=== include_role (include_roles.yml) ==="
ansible-playbook playbooks/include_roles.yml --list-tasks
```

### Шаг 5.3. Запуск основного плейбука

```bash
# Режим проверки (dry-run)
ansible-playbook playbooks/site.yml --check

# Реальный запуск
ansible-playbook playbooks/site.yml -v
```

### Шаг 5.4. Запуск с тегами

```bash
# Запуск только базовой настройки
ansible-playbook playbooks/site.yml --tags common

# Запуск только роли Nginx
ansible-playbook playbooks/site.yml --tags nginx

# Запуск только установки (без конфигурации)
ansible-playbook playbooks/import_roles.yml --tags installation

# Запуск с пропуском конфигурации
ansible-playbook playbooks/import_roles.yml --skip-tags configuration
```

### Шаг 5.5. Проверка результата

```bash
# Проверка статуса Nginx на web-сервере
ansible webservers -m systemd -a "name=nginx state=started"

# Проверка, что веб-сервер отвечает
ansible webservers -m uri -a "url=http://localhost status_code=200"

# Проверка содержимого сайта
ansible webservers -m uri -a "url=http://localhost return_content=yes"

# Проверка создания файла с информацией о настройке
ansible all -m command -a "cat /opt/ansible/provision_info.txt"

# Проверка установки базовых пакетов
ansible all -m command -a "dpkg -l | grep -E 'htop|git|curl'"
```

### Шаг 5.6. Проверка конфигурации Nginx

```bash
# Проверка конфигурации Nginx
ansible webservers -m command -a "nginx -t"

# Просмотр конфига Nginx
ansible webservers -m command -a "cat /etc/nginx/nginx.conf | head -20"

# Просмотр виртуального хоста
ansible webservers -m command -a "cat /etc/nginx/sites-available/default"
```

---

