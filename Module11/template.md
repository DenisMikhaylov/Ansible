# Лабораторная работа №11: Шаблоны Jinja2

## Цель работы
Научиться создавать динамические конфигурационные файлы с помощью шаблонов Jinja2 в Ansible: использовать переменные, фильтры, тесты и управляющие структуры (условия и циклы).

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab11-templates/{templates,files}
cd ~/lab11-templates
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab11-templates/inventory.ini
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

[webservers:vars]
app_port=8080
app_name=webapp

[databases:vars]
db_port=5432
db_name=mydb
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab11-templates/ansible.cfg
```

```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
remote_user = ansible
become = yes
become_method = sudo
become_user = root

[ssh_connection]
pipelining = True
```

---

## Часть 2. Синтаксис шаблонов Jinja2 (20 минут)

### Шаг 2.1. Создание простого шаблона

```bash
nano ~/lab11-templates/templates/simple_config.j2
```

```jinja2
{# Это комментарий в шаблоне Jinja2 - он не попадёт в итоговый файл #}
# ============================================
# Конфигурационный файл
# ============================================
# Сгенерировано: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
# Хост: {{ inventory_hostname }}
# Пользователь: {{ ansible_user }}
# ============================================

# Простые переменные
app_name = {{ app_name | default('myapp') }}
app_port = {{ app_port | default(8080) }}
environment = {{ environment | default('development') }}

# Переменные из словаря
database_host = {{ database.host | default('localhost') }}
database_port = {{ database.port | default(5432) }}
database_name = {{ database.name | default('mydb') }}

# Переменные из hostvars (другие хосты)
{# This shows how we can access variables from other hosts #}
web_server_ip = {{ hostvars['web-server']['ansible_default_ipv4']['address'] | default('unknown') }}
db_server_ip = {{ hostvars['db-server']['ansible_default_ipv4']['address'] | default('unknown') }}

# Системные факты
os_family = {{ ansible_os_family }}
distribution = {{ ansible_distribution }}
kernel = {{ ansible_kernel }}
cpu_cores = {{ ansible_processor_vcpus }}
memory_mb = {{ ansible_memtotal_mb }}
```

### Шаг 2.2. Playbook для использования шаблона

```bash
nano ~/lab11-templates/01-simple-template.yml
```

```yaml
---
- name: Использование простого шаблона Jinja2
  hosts: all
  gather_facts: yes
  
  vars:
    environment: "production"
    database:
      host: "{{ ansible_default_ipv4.address }}"
      port: 5432
      name: "{{ db_name | default('mydb') }}"
  
  tasks:
    - name: Создание директории для конфигов
      ansible.builtin.file:
        path: /opt/template_test
        state: directory
        mode: '0755'
    
    - name: Генерация конфигурации из шаблона
      ansible.builtin.template:
        src: templates/simple_config.j2
        dest: "/opt/template_test/{{ inventory_hostname }}_config.txt"
        mode: '0644'
        backup: yes
    
    - name: Просмотр сгенерированного файла
      ansible.builtin.shell: cat /opt/template_test/{{ inventory_hostname }}_config.txt
      register: config_content
      changed_when: false
    
    - name: Вывод содержимого
      ansible.builtin.debug:
        msg: "{{ config_content.stdout_lines }}"
```

### Шаг 2.3. Выполнение

```bash
ansible-playbook 01-simple-template.yml

# Проверка на серверах
ssh ansible@192.168.1.11 "cat /opt/template_test/web-server_config.txt"
ssh ansible@192.168.1.12 "cat /opt/template_test/db-server_config.txt"
```

---

## Часть 3. Фильтры в шаблонах (25 минут)

### Шаг 3.1. Создание шаблона с фильтрами

```bash
nano ~/lab11-templates/templates/filters_demo.j2
```

```jinja2
# ============================================
# ДЕМОНСТРАЦИЯ ФИЛЬТРОВ JINJA2
# ============================================

{# ----- СТРОКОВЫЕ ФИЛЬТРЫ ----- #}
[strings]
original = "{{ app_name | default('mywebapp') }}"
upper_case = "{{ (app_name | default('mywebapp')) | upper }}"
lower_case = "{{ (app_name | default('mywebapp')) | lower }}"
capitalize = "{{ (app_name | default('mywebapp')) | capitalize }}"
title_case = "{{ (app_name | default('mywebapp')) | title }}"
trim_example = "  {{ (app_name | default('mywebapp')) }}  " | trim
replace_test = "hello world" | replace("world", "ansible")

{# ----- ЧИСЛОВЫЕ ФИЛЬТРЫ ----- #}
[numbers]
absolute = {{ -42 | abs }}
rounded = {{ 3.14159 | round(2) }}
integer = {{ "123" | int }}
float_number = {{ "3.14" | float }}

{# ----- СПИСОЧНЫЕ ФИЛЬТРЫ ----- #}
[lists]
{% set sample_list = [3, 1, 4, 1, 5, 9, 2, 6, 5] %}
original_list = {{ sample_list }}
sorted_list = {{ sample_list | sort }}
unique_values = {{ sample_list | unique | sort }}
list_length = {{ sample_list | length }}
first_item = {{ sample_list | first }}
last_item = {{ sample_list | last }}
joined_string = {{ sample_list | join(', ') }}

{# ----- ФИЛЬТРЫ ДЛЯ РАБОТЫ С ПУТЯМИ ----- #}
[paths]
full_path = "/etc/nginx/nginx.conf"
basename = {{ "/etc/nginx/nginx.conf" | basename }}
dirname = {{ "/etc/nginx/nginx.conf" | dirname }}

{# ----- ФИЛЬТР DEFAULT ----- #}
[default_values]
defined_var = {{ defined_var | default('значение по умолчанию') }}
undefined_var = {{ undefined_var | default('эта переменная не определена') }}

{# ----- ЦЕПОЧКИ ФИЛЬТРОВ ----- #}
[chained_filters]
{% set servers = ['web01', 'db01', 'cache01'] %}
server_list = {{ servers | join(', ') | upper }}
first_server_capitalized = {{ servers | first | capitalize }}
```

### Шаг 3.2. Playbook для демонстрации фильтров

```bash
nano ~/lab11-templates/02-filters-demo.yml
```

```yaml
---
- name: Демонстрация фильтров Jinja2
  hosts: all
  gather_facts: yes
  
  vars:
    app_name: "mywebapp"
    defined_var: "это определённая переменная"
  
  tasks:
    - name: Генерация файла с фильтрами
      ansible.builtin.template:
        src: templates/filters_demo.j2
        dest: /tmp/filters_demo.txt
        mode: '0644'
    
    - name: Просмотр результата
      ansible.builtin.shell: cat /tmp/filters_demo.txt
      register: filters_output
      changed_when: false
    
    - name: Вывод
      ansible.builtin.debug:
        msg: "{{ filters_output.stdout_lines }}"
```

### Шаг 3.3. Выполнение

```bash
ansible-playbook 02-filters-demo.yml

# Просмотр результата
ssh ansible@192.168.1.11 "cat /tmp/filters_demo.txt"
```

---

## Часть 4. Тесты в шаблонах (15 минут)

### Шаг 4.1. Создание шаблона с тестами

```bash
nano ~/lab11-templates/templates/tests_demo.j2
```

```jinja2
# ============================================
# ДЕМОНСТРАЦИЯ ТЕСТОВ JINJA2
# ============================================

{# ----- ПРОВЕРКА ОПРЕДЕЛЕНИЯ ПЕРЕМЕННЫХ ----- #}
[variable_checks]
{% if database_password is defined %}
database_password_defined = true
database_password = {{ database_password }}
{% else %}
database_password_defined = false
database_password = "не задан"
{% endif %}

{% if undefined_var is undefined %}
undefined_var_status = "переменная НЕ определена"
{% endif %}

{# ----- ПРОВЕРКА ТИПОВ ----- #}
[type_checks]
{% set test_number = 42 %}
{% set test_string = "hello" %}
{% set test_list = [1, 2, 3] %}

is_number = {{ test_number is number }}
is_string = {{ test_string is string }}
is_sequence = {{ test_list is sequence }}
is_mapping = {{ test_list is mapping }}

{# ----- ПРОВЕРКА ЧИСЕЛ ----- #}
[number_checks]
{% for i in range(1, 11) %}
number_{{ i }}_is_even = {{ i is even }}
number_{{ i }}_is_odd = {{ i is odd }}
{% endfor %}

{# ----- ПРОВЕРКА СТРОК ----- #}
[string_checks]
{% set test = "HELLO" %}
is_upper = {{ test is upper }}
is_lower = {{ test is lower }}

{# ----- ПРОВЕРКА НА NONE ----- #}
[none_checks]
{% set null_var = none %}
is_none = {{ null_var is none }}
is_not_none = {{ null_var is not none }}
```

### Шаг 4.2. Playbook для тестов

```bash
nano ~/lab11-templates/03-tests-demo.yml
```

```yaml
---
- name: Демонстрация тестов Jinja2
  hosts: all
  
  vars:
    database_password: "secure_password_123"
  
  tasks:
    - name: Генерация файла с тестами
      ansible.builtin.template:
        src: templates/tests_demo.j2
        dest: /tmp/tests_demo.txt
        mode: '0644'
    
    - name: Просмотр результата
      ansible.builtin.shell: cat /tmp/tests_demo.txt
      register: tests_output
      changed_when: false
    
    - name: Вывод
      ansible.builtin.debug:
        msg: "{{ tests_output.stdout_lines }}"
```

### Шаг 4.3. Выполнение

```bash
ansible-playbook 03-tests-demo.yml
```

---

## Часть 5. Управляющие структуры (условия и циклы) (25 минут)

### Шаг 5.1. Создание шаблона с условиями и циклами

```bash
nano ~/lab11-templates/templates/control_structures.j2
```

```jinja2
# ============================================
# УПРАВЛЯЮЩИЕ СТРУКТУРЫ В JINJA2
# ============================================

{# ----- УСЛОВИЯ (IF/ELIF/ELSE) ----- #}
[conditional_config]
environment = {{ environment | default('development') }}

{% if environment == "production" %}
log_level = ERROR
debug_mode = false
backup_enabled = true
monitoring_enabled = true
{% elif environment == "staging" %}
log_level = WARNING
debug_mode = true
backup_enabled = true
monitoring_enabled = true
{% else %}
log_level = DEBUG
debug_mode = true
backup_enabled = false
monitoring_enabled = false
{% endif %}

{# ----- ЦИКЛЫ ПО СПИСКАМ ----- #}
[server_list]
{% set servers = groups['all'] %}
total_servers = {{ servers | length }}

servers:
{% for server in servers %}
  - name: {{ server }}
    ip: {{ hostvars[server]['ansible_default_ipv4']['address'] }}
    {% if server in groups['webservers'] %}role: web{% endif %}
    {% if server in groups['databases'] %}role: database{% endif %}
{% endfor %}

{# ----- ЦИКЛ С ИНДЕКСАМИ (LOOP.INDEX) ----- #}
[numbered_list]
{% for server in servers %}
{{ loop.index }}. {{ server }} ({{ hostvars[server]['ansible_default_ipv4']['address'] }})
{% endfor %}

{# ----- ЦИКЛ ПО СЛОВАРЮ ----- #}
[config_parameters]
{% set params = {
    'timeout': 30,
    'retries': 5,
    'cache_size': 256,
    'max_connections': 1000
} %}

{% for key, value in params.items() %}
{{ key }} = {{ value }}
{% endfor %}

{# ----- ЦИКЛ С УСЛОВИЕМ (ФИЛЬТРАЦИЯ) ----- #}
[active_servers]
{% set active_servers = groups['all'] | select('in', enabled_hosts | default([])) | list %}
{% if active_servers | length > 0 %}
active: {{ active_servers | join(', ') }}
{% else %}
active: none
{% endif %}

{# ----- ВЛОЖЕННЫЕ ЦИКЛЫ ----- #}
[matrix_config]
{% set environments = ['dev', 'staging', 'prod'] %}
{% set services = ['web', 'api', 'db'] %}

{% for env in environments %}
[{{ env }}]
{% for service in services %}
  {{ service }}_enabled = {% if env == 'prod' %}true{% else %}false{% endif %}
{% endfor %}
{% endfor %}

{# ----- ОПЕРАТОР SET (ПРИСВАИВАНИЕ) ----- #}
[calculated_values]
{% set total_memory = ansible_memtotal_mb // 1024 %}
{% set recommended_workers = ansible_processor_vcpus * 2 %}
{% set app_port = custom_port | default(8080) %}

total_ram_gb = {{ total_memory }}
recommended_workers = {{ recommended_workers }}
final_port = {{ app_port }}
```

### Шаг 5.2. Playbook для управляющих структур

```bash
nano ~/lab11-templates/04-control-structures.yml
```

```yaml
---
- name: Демонстрация управляющих структур Jinja2
  hosts: all
  gather_facts: yes
  
  vars:
    environment: "production"
    enabled_hosts:
      - web-server
      - db-server
    custom_port: 9090
  
  tasks:
    - name: Генерация конфигурации с управляющими структурами
      ansible.builtin.template:
        src: templates/control_structures.j2
        dest: /tmp/control_structures.txt
        mode: '0644'
    
    - name: Просмотр результата
      ansible.builtin.shell: cat /tmp/control_structures.txt
      register: control_output
      changed_when: false
    
    - name: Вывод
      ansible.builtin.debug:
        msg: "{{ control_output.stdout_lines }}"
```

### Шаг 5.3. Выполнение

```bash
ansible-playbook 04-control-structures.yml

# Запуск с другим окружением
ansible-playbook 04-control-structures.yml -e "environment=development"
```

---

## Часть 6. Комплексный пример: Конфигурация nginx (25 минут)

### Шаг 6.1. Создание шаблона nginx

```bash
nano ~/lab11-templates/templates/nginx.conf.j2
```

```jinja2
# ============================================
# Конфигурация Nginx
# Сгенерировано Ansible {{ ansible_date_time.date }}
# Хост: {{ inventory_hostname }}
# ============================================

user {{ nginx_user | default('www-data') }};
worker_processes {{ nginx_worker_processes | default('auto') }};
pid /run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections | default(1024) }};
    multi_accept on;
    use epoll;
}

http {
    # Базовые настройки
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout | default(65) }};
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Форматы логов
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log {{ nginx_log_level | default('warn') }};

    # Сжатие
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level {{ nginx_gzip_comp_level | default(6) }};
    gzip_types text/plain text/css text/xml text/javascript 
               application/json application/javascript application/xml+rss 
               application/rss+xml image/svg+xml;

    # Upstream бэкенды
    {% if groups['webservers'] | length > 1 %}
    upstream backend {
        {% for server in groups['webservers'] %}
        server {{ hostvars[server]['ansible_default_ipv4']['address'] }}:{{ app_port | default(8080) }} weight={{ loop.index }};
        {% endfor %}
        keepalive 32;
    }
    {% endif %}

    # Основной сервер
    server {
        listen {{ nginx_port | default(80) }};
        server_name {{ ansible_fqdn | default(ansible_hostname) }};

        root {{ nginx_root | default('/var/www/html') }};
        index index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        location /status {
            stub_status on;
            access_log off;
            {% if monitoring_allowed_ips is defined %}
            {% for ip in monitoring_allowed_ips %}
            allow {{ ip }};
            {% endfor %}
            deny all;
            {% else %}
            allow 127.0.0.1;
            deny all;
            {% endif %}
        }

        {% if ssl_enabled | default(false) %}
        # SSL конфигурация
        listen 443 ssl http2;
        ssl_certificate /etc/nginx/ssl/{{ ansible_hostname }}.crt;
        ssl_certificate_key /etc/nginx/ssl/{{ ansible_hostname }}.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        {% endif %}

        # Обработка ошибок
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
    }

    {% for vhost in virtual_hosts | default([]) %}
    # Виртуальный хост: {{ vhost.name }}
    server {
        listen {{ vhost.port | default(80) }};
        server_name {{ vhost.server_name }};
        root {{ vhost.document_root }};
        
        {% if vhost.ssl is defined %}
        listen 443 ssl;
        ssl_certificate {{ vhost.ssl.cert }};
        ssl_certificate_key {{ vhost.ssl.key }};
        {% endif %}
        
        location / {
            try_files $uri $uri/ =404;
        }
    }
    {% endfor %}
}
```

### Шаг 6.2. Playbook для настройки nginx

```bash
nano ~/lab11-templates/05-nginx-template.yml
```

```yaml
---
- name: Настройка Nginx с использованием шаблона
  hosts: webservers
  gather_facts: yes
  
  vars:
    nginx_port: 80
    nginx_worker_processes: "auto"
    nginx_worker_connections: 4096
    nginx_keepalive_timeout: 120
    nginx_gzip_comp_level: 6
    nginx_log_level: "warn"
    monitoring_allowed_ips:
      - "192.168.1.0/24"
      - "10.0.0.0/8"
    virtual_hosts:
      - name: "example"
        port: 80
        server_name: "example.com"
        document_root: "/var/www/example"
      - name: "test"
        port: 8080
        server_name: "test.local"
        document_root: "/var/www/test"
  
  tasks:
    - name: Установка nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes
    
    - name: Создание директорий для виртуальных хостов
      ansible.builtin.file:
        path: "{{ item.document_root }}"
        state: directory
        mode: '0755'
      loop: "{{ virtual_hosts }}"
    
    - name: Генерация конфигурации nginx из шаблона
      ansible.builtin.template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: '0644'
        backup: yes
      notify: reload nginx
    
    - name: Проверка конфигурации nginx
      ansible.builtin.command: nginx -t
      register: nginx_test
      changed_when: false
    
    - name: Результат проверки
      ansible.builtin.debug:
        msg: "Nginx config test: {{ nginx_test.rc == 0 and 'OK' or 'FAILED' }}"
  
  handlers:
    - name: reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

### Шаг 6.3. Выполнение

```bash
ansible-playbook 05-nginx-template.yml

# Проверка конфигурации
ssh ansible@192.168.1.11 "nginx -t && cat /etc/nginx/nginx.conf | head -30"
```

---



| Цикл (for) | `{% for item in list %}...{% endfor %}` | `{% for host in groups['all'] %}` |
| Присваивание (set) | `{% set var = value %}` | `{% set port = 8080 %}` |
| Фильтр | `{{ var \| filter }}` | `
