# Лабораторная работа №6: Переменные в Ansible

## Цель работы
Научиться создавать и использовать переменные в Ansible в различных местах: в плейбуках, инвентаризации и внешних файлах, а также понимать приоритеты переменных.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

**Выполните на Control Node от пользователя ansible:**

```bash
mkdir -p ~/lab6-variables/{group_vars,host_vars,vars,templates}
cd ~/lab6-variables
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab6-variables/inventory.ini
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

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab6-variables/ansible.cfg
```

**Содержимое файла:**

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

## Часть 2. Синтаксис переменных 

### Шаг 2.1. Создание playbook для демонстрации синтаксиса

Создайте файл `01-syntax.yml`:

```bash
nano ~/lab6-variables/01-syntax.yml
```

**Содержимое файла:**

```yaml
---
- name: Демонстрация синтаксиса переменных
  hosts: all
  
  vars:
    # Простые типы переменных
    string_var: "Hello World"
    number_var: 42
    float_var: 3.14
    boolean_true: true
    boolean_false: false
    null_var: null
    
    # Список (массив)
    users_list:
      - alice
      - bob
      - charlie
    
    # Словарь (объект)
    database_config:
      host: localhost
      port: 5432
      name: mydb
      user: admin
      password: secret
    
    # Многострочная строка
    multiline_string: |
      Это первая строка
      Это вторая строка
      Это третья строка
  
  tasks:
    - name: Вывод простых переменных
      ansible.builtin.debug:
        msg: |
          === ПРОСТЫЕ ПЕРЕМЕННЫЕ ===
          string_var: {{ string_var }}
          number_var: {{ number_var }}
          float_var: {{ float_var }}
          boolean_true: {{ boolean_true }}
          boolean_false: {{ boolean_false }}
          null_var: {{ null_var | default('NULL') }}
    
    - name: Вывод элементов списка
      ansible.builtin.debug:
        msg: |
          === РАБОТА СО СПИСКОМ ===
          Весь список: {{ users_list }}
          Первый элемент: {{ users_list[0] }}
          Последний элемент: {{ users_list[-1] }}
          Диапазон [0:2]: {{ users_list[0:2] }}
    
    - name: Цикл по списку
      ansible.builtin.debug:
        msg: "Пользователь: {{ item }}"
      loop: "{{ users_list }}"
    
    - name: Вывод элементов словаря
      ansible.builtin.debug:
        msg: |
          === РАБОТА СО СЛОВАРЁМ ===
          host: {{ database_config.host }}
          port: {{ database_config.port }}
          name: {{ database_config['name'] }}
          user: {{ database_config.user }}
          password: {{ database_config.password | default('empty') }}
    
    - name: Многострочная строка
      ansible.builtin.debug:
        msg: |
          === МНОГОСТРОЧНАЯ СТРОКА ===
          {{ multiline_string }}
```

### Шаг 2.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 01-syntax.yml --syntax-check

# Выполнение
ansible-playbook 01-syntax.yml
```

---

## Часть 3. Определение переменных в рабочих книгах 

### Шаг 3.1. Playbook с переменными на разных уровнях

Создайте файл `02-playbook-vars.yml`:

```bash
nano ~/lab6-variables/02-playbook-vars.yml
```

**Содержимое файла:**

```yaml
---
- name: Переменные на уровне play
  hosts: all
  vars:
    play_level_var: "Переменная уровня play"
    app_name: "mywebapp"
    app_version: "1.0.0"
  
  tasks:
    # Переменная в задаче
    - name: Задача с локальной переменной
      vars:
        task_level_var: "Переменная уровня задачи"
      ansible.builtin.debug:
        msg: |
          === ПЕРЕМЕННЫЕ ===
          play_level_var: {{ play_level_var }}
          task_level_var: {{ task_level_var }}
          app_name: {{ app_name }}
          app_version: {{ app_version }}
    
    # Регистрация результата
    - name: Сохранение результата команды
      ansible.builtin.shell: hostname
      register: hostname_result
    
    - name: Использование зарегистрированной переменной
      ansible.builtin.debug:
        msg: "Имя хоста из команды: {{ hostname_result.stdout }}"
    
    # set_fact - создание переменной на лету
    - name: Создание переменной через set_fact
      ansible.builtin.set_fact:
        combined_message: "{{ app_name }} ({{ app_version }}) на {{ hostname_result.stdout }}"
    
    - name: Использование set_fact переменной
      ansible.builtin.debug:
        msg: "{{ combined_message }}"
    
    # Фильтры с переменными
    - name: Применение фильтров
      ansible.builtin.debug:
        msg: |
          === ФИЛЬТРЫ ===
          app_name в верхнем регистре: {{ app_name | upper }}
          app_name в нижнем регистре: {{ app_name | lower }}
          app_version с default: {{ undefined_var | default('НЕ ОПРЕДЕЛЕНА') }}
```

### Шаг 3.2. Выполнение playbook

```bash
ansible-playbook 02-playbook-vars.yml
```

---

## Часть 4. Определение переменных в инвентаризации 

### Шаг 4.1. Создание структуры для переменных инвентаря

```bash
mkdir -p ~/lab6-variables/inventories/prod/group_vars
mkdir -p ~/lab6-variables/inventories/prod/host_vars
mkdir -p ~/lab6-variables/inventories/staging/group_vars
mkdir -p ~/lab6-variables/inventories/staging/host_vars
```

### Шаг 4.2. Создание инвентаря для разных окружений

**Production инвентарь:**

```bash
nano ~/lab6-variables/inventories/prod/hosts.ini
```

```ini
[webservers]
web-prod ansible_host=192.168.1.11

[databases]
db-prod ansible_host=192.168.1.12

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
environment=production
```

**Staging инвентарь:**

```bash
nano ~/lab6-variables/inventories/staging/hosts.ini
```

```ini
[webservers]
web-stg ansible_host=192.168.1.11

[databases]
db-stg ansible_host=192.168.1.12

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
environment=staging
```

### Шаг 4.3. Создание переменных для групп и хостов

**group_vars для всех хостов (production):**

```bash
nano ~/lab6-variables/inventories/prod/group_vars/all.yml
```

```yaml
---
# Переменные для всех хостов в production
ntp_server: pool.ntp.org
timezone: UTC
admin_email: admin@production.com
backup_enabled: true
log_level: ERROR
```

**group_vars для веб-серверов (production):**

```bash
nano ~/lab6-variables/inventories/prod/group_vars/webservers.yml
```

```yaml
---
# Переменные для веб-серверов в production
web_package: nginx
web_port: 80
web_ssl_port: 443
web_root: /var/www/html
worker_processes: auto
max_clients: 1024
```

**group_vars для БД-серверов (production):**

```bash
nano ~/lab6-variables/inventories/prod/group_vars/databases.yml
```

```yaml
---
# Переменные для серверов БД в production
db_package: mariadb-server
db_port: 3306
db_root_password: "prod_secure_password"
db_max_connections: 200
```

**host_vars для конкретного хоста web-prod:**

```bash
nano ~/lab6-variables/inventories/prod/host_vars/web-prod.yml
```

```yaml
---
# Переменные только для web-prod (переопределяют групповые)
web_port: 8080  # переопределяем порт для этого хоста
custom_domain: special.prod.example.com
```

**group_vars для staging (упрощённые настройки):**

```bash
nano ~/lab6-variables/inventories/staging/group_vars/all.yml
```

```yaml
---
# Переменные для всех хостов в staging
ntp_server: pool.ntp.org
timezone: UTC
admin_email: admin@staging.com
backup_enabled: false
log_level: DEBUG
debug_mode: true
```

### Шаг 4.4. Playbook для проверки переменных инвентаря

Создайте файл `03-inventory-vars.yml`:

```bash
nano ~/lab6-variables/03-inventory-vars.yml
```

**Содержимое файла:**

```yaml
---
- name: Проверка переменных из инвентаря
  hosts: all
  gather_facts: no
  
  tasks:
    - name: Вывод переменных окружения
      ansible.builtin.debug:
        msg: |
          ========================================
          ХОСТ: {{ inventory_hostname }}
          ОКРУЖЕНИЕ: {{ environment | default('не определено') }}
          ========================================
          Глобальные переменные:
          - ntp_server: {{ ntp_server | default('N/A') }}
          - timezone: {{ timezone | default('N/A') }}
          - admin_email: {{ admin_email | default('N/A') }}
          - backup_enabled: {{ backup_enabled | default('N/A') }}
          - log_level: {{ log_level | default('N/A') }}
          ========================================
          Групповые переменные (если есть):
          - web_package: {{ web_package | default('N/A') }}
          - web_port: {{ web_port | default('N/A') }}
          - db_package: {{ db_package | default('N/A') }}
          - db_port: {{ db_port | default('N/A') }}
          ========================================
          Хостовые переменные:
          - custom_domain: {{ custom_domain | default('N/A') }}
          ========================================
```

### Шаг 4.5. Проверка переменных инвентаря

```bash
# Проверка production окружения
ansible-playbook -i inventories/prod/hosts.ini 03-inventory-vars.yml

# Проверка staging окружения
ansible-playbook -i inventories/staging/hosts.ini 03-inventory-vars.yml

# Просмотр всех переменных инвентаря
ansible-inventory -i inventories/prod/hosts.ini --graph
ansible-inventory -i inventories/prod/hosts.ini --host web-prod
```

---

## Часть 5. Определение переменных во внешних файлах 

### Шаг 5.1. Создание файлов с переменными

**Файл с общими переменными:**

```bash
nano ~/lab6-variables/vars/common.yml
```

```yaml
---
# Общие переменные
common_packages:
  - curl
  - wget
  - git
  - vim
  - htop
  - net-tools

ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
  - 2.pool.ntp.org

default_timezone: UTC
default_user: ansible
```

**Файл с переменными приложения:**

```bash
nano ~/lab6-variables/vars/app.yml
```

```yaml
---
# Переменные приложения
app_name: mywebapp
app_version: 2.1.0
app_port: 8080
app_user: myapp
app_group: myapp
app_dir: /opt/{{ app_name }}
```

**Файл с переменными для базы данных:**

```bash
nano ~/lab6-variables/vars/database.yml
```

```yaml
---
# Переменные базы данных
database:
  type: postgresql
  version: 15
  host: localhost
  port: 5432
  name: myapp_db
  user: db_user
  password: "{{ vault_db_password | default('changeme') }}"
  pool_size: 10
```

**Файл с секретами (для демонстрации):**

```bash
nano ~/lab6-variables/vars/secrets.yml
```

```yaml
---
# Секретные переменные (в реальности шифровать через ansible-vault)
api_key: "secret-api-key-12345"
db_password: "secure_db_password"
admin_password: "admin_pass"
```

### Шаг 5.2. Создание playbook с подключением внешних файлов

Создайте файл `04-external-vars.yml`:

```bash
nano ~/lab6-variables/04-external-vars.yml
```

**Содержимое файла:**

```yaml
---
- name: Использование переменных из внешних файлов
  hosts: all
  vars_files:
    - vars/common.yml
    - vars/app.yml
    - vars/database.yml
    - vars/secrets.yml
  
  vars:
    # Локальные переменные переопределяют внешние
    app_port: 9090  # переопределяем порт
  
  tasks:
    - name: Вывод переменных из common.yml
      ansible.builtin.debug:
        msg: |
          === ИЗ common.yml ===
          common_packages: {{ common_packages }}
          ntp_servers: {{ ntp_servers }}
          default_timezone: {{ default_timezone }}
          default_user: {{ default_user }}
    
    - name: Вывод переменных из app.yml
      ansible.builtin.debug:
        msg: |
          === ИЗ app.yml ===
          app_name: {{ app_name }}
          app_version: {{ app_version }}
          app_port (переопределён): {{ app_port }}
          app_user: {{ app_user }}
          app_dir: {{ app_dir }}
    
    - name: Вывод переменных из database.yml
      ansible.builtin.debug:
        msg: |
          === ИЗ database.yml ===
          database.type: {{ database.type }}
          database.version: {{ database.version }}
          database.host: {{ database.host }}
          database.port: {{ database.port }}
          database.name: {{ database.name }}
          database.user: {{ database.user }}
          database.pool_size: {{ database.pool_size }}
    
    - name: Вывод секретных переменных
      ansible.builtin.debug:
        msg: |
          === СЕКРЕТЫ (НЕ ИСПОЛЬЗУЙТЕ ТАК В РЕАЛЬНОМ КОДЕ!) ===
          api_key: {{ api_key }}
          db_password: {{ db_password }}
          admin_password: {{ admin_password }}
    
    - name: Демонстрация шаблона с переменными
      ansible.builtin.copy:
        content: |
          # Конфигурационный файл
          APP_NAME={{ app_name }}
          APP_VERSION={{ app_version }}
          APP_PORT={{ app_port }}
          DB_HOST={{ database.host }}
          DB_PORT={{ database.port }}
          DB_NAME={{ database.name }}
          DB_USER={{ database.user }}
          DB_PASSWORD={{ database.password }}
        dest: "/tmp/{{ inventory_hostname }}_config.txt"
        mode: '0644'
    
    - name: Проверка созданного файла
      ansible.builtin.shell: cat /tmp/{{ inventory_hostname }}_config.txt
      register: config_content
    
    - name: Вывод содержимого конфига
      ansible.builtin.debug:
        msg: "{{ config_content.stdout }}"
```

### Шаг 5.3. Выполнение playbook

```bash
ansible-playbook 04-external-vars.yml
```

---

## Часть 6. Приоритет переменных (20 минут)

### Шаг 6.1. Создание playbook для демонстрации приоритетов

Создайте файл `05-priority.yml`:

```bash
nano ~/lab6-variables/05-priority.yml
```

**Содержимое файла:**

```yaml
---
- name: Демонстрация приоритетов переменных
  hosts: all
  
  vars:
    # Переменная уровня play
    test_var: "PLAY - значение из play"
    priority_demo: "PLAY - значение по умолчанию"
  
  vars_files:
    - vars/common.yml
  
  tasks:
    - name: Задача с локальной переменной
      vars:
        test_var: "TASK - значение из задачи"
        priority_demo: "TASK - локальное значение"
      
      ansible.builtin.debug:
        msg: |
          === ПРИОРИТЕТ ПЕРЕМЕННЫХ ===
          test_var (задача переопределяет play): {{ test_var }}
          priority_demo (задача): {{ priority_demo }}
          common_packages (из external file): {{ common_packages }}
    
    - name: Создание переменной через set_fact
      ansible.builtin.set_fact:
        test_var: "SET_FACT - создано во время выполнения"
      
    - name: Переменная после set_fact
      ansible.builtin.debug:
        msg: "test_var после set_fact: {{ test_var }}"
    
    - name: Переменная с фильтром default
      ansible.builtin.debug:
        msg: "undefined_var: {{ undefined_var | default('DEFAULT - значение по умолчанию') }}"
```

### Шаг 6.2. Запуск с разными уровнями приоритета

```bash
# Базовый запуск
ansible-playbook 05-priority.yml

# Запуск с передачей переменной через командную строку (высший приоритет)
ansible-playbook 05-priority.yml -e "priority_demo=EXTRA_VARS - из командной строки"

# С несколькими переменными
ansible-playbook 05-priority.yml -e "test_var=EXTRA_VARS test_var2=value2"

# Из JSON файла
echo '{"priority_demo": "EXTRA_VARS - из JSON файла"}' > /tmp/extra_vars.json
ansible-playbook 05-priority.yml -e @/tmp/extra_vars.json
```

---

## Часть 7. Комплексный пример (20 минут)

### Шаг 7.1. Шаблон для генерации конфигурации

Создайте шаблон `templates/app_config.j2`:

```bash
nano ~/lab6-variables/templates/app_config.j2
```

```jinja2
# ============================================
# Конфигурация приложения {{ app_name }}
# Сгенерировано Ansible {{ ansible_date_time.date }}
# Хост: {{ ansible_hostname }}
# Окружение: {{ environment | default('development') }}
# ============================================

[server]
host = {{ ansible_default_ipv4.address | default('127.0.0.1') }}
port = {{ app_port | default(8080) }}
workers = {{ workers | default(4) }}
timeout = {{ timeout | default(30) }}

[database]
type = {{ database.type | default('postgresql') }}
host = {{ database.host | default('localhost') }}
port = {{ database.port | default(5432) }}
name = {{ database.name | default('myapp') }}
user = {{ database.user | default('app_user') }}
password = {{ database.password | default('changeme') }}
pool_size = {{ database.pool_size | default(10) }}

[logging]
level = {{ log_level | default('INFO') }}
file = {{ log_file | default('/var/log/{{ app_name }}/app.log') }}
format = "{{ log_format | default('%(asctime)s - %(name)s - %(levelname)s - %(message)s') }}"

[features]
debug = {{ debug_mode | default(false) }}
caching = {{ caching_enabled | default(true) }}
metrics = {{ metrics_enabled | default(false) }}

# Дополнительные настройки из внешних файлов
{% for key, value in extra_config.items() %}
{{ key }} = {{ value }}
{% endfor %}
```



6. Освоили фильтры и специальные переменные

**Готовность:** Вы готовы к созданию гибких, переиспользуемых плейбуков с использованием переменных.
