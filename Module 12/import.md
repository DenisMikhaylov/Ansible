# Лабораторная работа №12: Комплексные проекты автоматизации

## Цель работы
Научиться использовать обработчики, динамическое/статическое включение задач и шифрование секретов Ansible Vault для создания комплексных проектов автоматизации на трёх хостах Debian 12.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **3 управляемых хоста** — Linux Debian 12 (web-server, db-server, proxy-server)

**Требования:** Все четыре хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание структуры директорий для лабораторной

```bash
mkdir -p ~/lab12-complex/{playbooks,tasks,templates,handlers,group_vars/all,vault_files}
cd ~/lab12-complex
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab12-complex/inventory.ini
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
service_name=nginx
app_port=8080

[databases:vars]
db_type=postgresql
db_port=5432

```

### Шаг 1.3. Создание конфигурационного файла Ansible

```bash
nano ~/lab12-complex/ansible.cfg
```

```ini
[defaults]
inventory = ./inventory.ini
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

## Часть 2. Использование уведомлений и обработчиков (Handlers)

### Шаг 2.1. Создание обработчиков для сервисов

```bash
nano ~/lab12-complex/handlers/main.yml
```

```yaml
---
# Основные обработчики для сервисов
- name: перезапустить nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
    daemon_reload: yes

- name: перезагрузить nginx
  ansible.builtin.systemd:
    name: nginx
    state: reloaded

- name: проверить конфиг nginx
  ansible.builtin.command: nginx -t
  register: nginx_test_result
  changed_when: false
  listen: "validate nginx"

```

### Шаг 2.2. Создание плейбука с обработчиками

```bash
nano ~/lab12-complex/playbooks/web_with_handlers.yml
```

```yaml
---
- name: Установка и настройка Nginx с использованием обработчиков
  hosts: webservers
  become: yes
  gather_facts: yes

  handlers:
    - name: перезапустить nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted

    - name: проверить конфиг nginx
      ansible.builtin.command: nginx -t
      register: nginx_test
      changed_when: false

  tasks:
    - name: Установка Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes
      notify: перезапустить nginx

    - name: Создание тестовой страницы
      ansible.builtin.copy:
        content: |
          <html>
          <h1>Server {{ ansible_hostname }}</h1>
          <p>IP: {{ ansible_default_ipv4.address }}</p>
          <p>Time: {{ ansible_date_time.iso8601 }}</p>
          </html>
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: перезапустить nginx

    - name: Настройка конфигурации сайта из шаблона
      ansible.builtin.template:
        src: templates/nginx_site.conf.j2
        dest: /etc/nginx/sites-available/default
        backup: yes
      notify:
        - проверить конфиг nginx
        - перезапустить nginx
```

### Шаг 2.3. Создание шаблона для Nginx

```bash
nano ~/lab12-complex/templates/nginx_site.conf.j2
```

```nginx
server {
    listen {{ app_port | default(80) }};
    server_name {{ ansible_fqdn | default('_') }};
    
    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location /status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }

    access_log /var/log/nginx/{{ app_name | default('site') }}.log;
    error_log /var/log/nginx/{{ app_name | default('site') }}_error.log;
}
```

### Шаг 2.4. Запуск и проверка обработчиков

```bash
# Первый запуск - установка и настройка
ansible-playbook playbooks/web_with_handlers.yml

# Второй запуск - без изменений (обработчики не сработают)
ansible-playbook playbooks/web_with_handlers.yml

# Проверка статуса сервиса
ansible webservers -m systemd -a "name=nginx state=started"
```

### Шаг 2.5. Эксперимент с принудительным запуском обработчиков

```bash
nano ~/lab12-complex/playbooks/flush_handlers_test.yml
```

```yaml
---
- name: Тестирование flush_handlers
  hosts: webservers
  become: yes

  handlers:
    - name: перезапустить nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted

  tasks:
    - name: Изменить конфигурацию Nginx
      ansible.builtin.lineinfile:
        path: /etc/nginx/sites-available/default
        regexp: '^\s*listen\s+'
        line: '    listen 8080 default_server;'
      notify: перезапустить nginx

    - name: Принудительный запуск обработчиков СЕЙЧАС
      ansible.builtin.meta: flush_handlers

    - name: Проверка, что Nginx слушает новый порт
      ansible.builtin.wait_for:
        port: 8080
        timeout: 10
        state: started

    - name: Проверка работоспособности
      ansible.builtin.uri:
        url: http://localhost:8080
        status_code: 200
      register: result

    - name: Вывод результата проверки
      ansible.builtin.debug:
        msg: "Nginx успешно перезагружен и отвечает на порту 8080"
```

```bash
ansible-playbook playbooks/flush_handlers_test.yml
```

---

## Часть 3. Включение и импорт задач и плейбуков 

### Шаг 3.1. Создание подключаемых файлов задач

**Базовые задачи для Debian 12:**

```bash
nano ~/lab12-complex/tasks/base_setup.yml
```

```yaml
---
- name: Обновление пакетов
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Установка базовых утилит
  ansible.builtin.apt:
    name:
      - curl
      - wget
      - htop
      - git
      - vim
      - net-tools
      - ca-certificates
    state: present

- name: Установка часового пояса
  ansible.builtin.timezone:
    name: Europe/Moscow

- name: Настройка локалей
  ansible.builtin.locale_gen:
    name: "{{ item }}"
    state: present
  loop:
    - ru_RU.UTF-8
    - en_US.UTF-8
```

**Настройка безопасности:**

```bash
nano ~/lab12-complex/tasks/security.yml
```

```yaml
---
- name: Отключение входа по паролю для root
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'
    backup: yes
  notify: перезапустить ssh

- name: Установка Fail2ban
  ansible.builtin.apt:
    name: fail2ban
    state: present

- name: Запуск Fail2ban
  ansible.builtin.systemd:
    name: fail2ban
    state: started
    enabled: yes

- name: Базовая настройка UFW
  ansible.builtin.ufw:
    rule: "{{ item.rule }}"
    port: "{{ item.port }}"
    proto: "{{ item.proto }}"
  loop:
    - { rule: 'allow', port: '22', proto: 'tcp' }
    - { rule: 'allow', port: '80', proto: 'tcp' }
    - { rule: 'allow', port: '443', proto: 'tcp' }

- name: Включение UFW
  ansible.builtin.ufw:
    state: enabled
    policy: deny
```

**Создание пользователей:**

```bash
nano ~/lab12-complex/tasks/create_users.yml
```

```yaml
---
- name: Создание пользователя {{ user_item.name }}
  ansible.builtin.user:
    name: "{{ user_item.name }}"
    shell: "{{ user_item.shell | default('/bin/bash') }}"
    groups: "{{ user_item.groups | default('') }}"
    append: yes
    state: present
    create_home: yes

- name: Добавление SSH ключа для {{ user_item.name }}
  ansible.builtin.authorized_key:
    user: "{{ user_item.name }}"
    state: present
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

- name: Добавление пользователя в sudoers
  ansible.builtin.lineinfile:
    path: /etc/sudoers.d/{{ user_item.name }}
    line: "{{ user_item.name }} ALL=(ALL) NOPASSWD:ALL"
    create: yes
    mode: '0440'
  when: user_item.sudo | default(false)
```

### Шаг 3.2. Демонстрация разницы между import и include

```bash
nano ~/lab12-complex/playbooks/include_vs_import.yml
```

```yaml
---
- name: ДЕМОНСТРАЦИЯ: статический импорт (import_tasks)
  hosts: all
  gather_facts: no
  
  tasks:
    - name: Статический импорт - загружается ДО выполнения
      ansible.builtin.import_tasks: tasks/base_setup.yml
      # when условие применяется ко ВСЕМ задачам внутри
      when: ansible_os_family == "Debian"

- name: ДЕМОНСТРАЦИЯ: динамическое включение (include_tasks)
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Динамическое включение с циклом
      ansible.builtin.include_tasks: tasks/create_users.yml
      loop:
        - { name: "deploy_user", shell: "/bin/bash", sudo: true }
        - { name: "monitoring", shell: "/bin/bash", sudo: false }
        - { name: "backup_user", shell: "/bin/sh", sudo: false }
      vars:
        user_item: "{{ item }}"
      when: inventory_hostname in groups['webservers'] or inventory_hostname in groups['databases']
```

### Шаг 3.3. Создание плейбука для web-сервера с импортами

```bash
nano ~/lab12-complex/playbooks/webserver_full.yml
```

```yaml
---
- name: Полная настройка WEB сервера
  hosts: webservers
  become: yes
  gather_facts: yes

  handlers:
    - name: перезапустить ssh
      ansible.builtin.systemd:
        name: ssh
        state: restarted

  tasks:
    # Статический импорт - всегда должен выполняться
    - name: Базовая настройка системы
      ansible.builtin.import_tasks: tasks/base_setup.yml
      tags: [base, always]

    # Динамическое включение с условием
    - name: Настройка безопасности
      ansible.builtin.include_tasks: tasks/security.yml
      when: ansible_distribution == "Debian" and ansible_distribution_major_version == "12"
      tags: security

    # Динамическое включение с данными из inventory
    - name: Создание пользователей
      ansible.builtin.include_tasks: tasks/create_users.yml
      loop:
        - { name: "webadmin", sudo: true }
        - { name: "www-data", sudo: false }
      vars:
        user_item: "{{ item }}"

    # Основная настройка web-сервера
    - name: Установка Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
      notify: перезапустить nginx

    - name: Копирование конфигурации
      ansible.builtin.template:
        src: templates/nginx_site.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: перезапустить nginx

  handlers:
    - name: перезапустить nginx
      ansible.builtin.systemd:
        name: nginx
        state: restarted
```

### Шаг 3.4. Создание главного плейбука из нескольких файлов

```bash
nano ~/lab12-complex/site.yml
```

```yaml
---
# Статический импорт плейбуков - все загружаются заранее
- import_playbook: playbooks/webserver_full.yml
- import_playbook: playbooks/database_setup.yml
- import_playbook: playbooks/proxy_setup.yml
```

```bash
nano ~/lab12-complex/playbooks/database_setup.yml
```

```yaml
---
- name: Настройка сервера БД
  hosts: databases
  become: yes

  tasks:
    - name: Установка PostgreSQL
      ansible.builtin.apt:
        name:
          - postgresql
          - postgresql-contrib
        state: present

    - name: Запуск PostgreSQL
      ansible.builtin.systemd:
        name: postgresql
        state: started
        enabled: yes
```

```bash
nano ~/lab12-complex/playbooks/proxy_setup.yml
```

```yaml
---
- name: Настройка прокси-сервера
  hosts: proxies
  become: yes

  tasks:
    - name: Установка HAProxy
      ansible.builtin.apt:
        name: haproxy
        state: present

    - name: Настройка HAProxy
      ansible.builtin.template:
        src: templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
      notify: перезапустить haproxy

  handlers:
    - name: перезапустить haproxy
      ansible.builtin.systemd:
        name: haproxy
        state: restarted
```

### Шаг 3.5. Запуск и отладка импортов

```bash
# Просмотр списка задач (import_tasks видны, include_tasks - нет)
ansible-playbook site.yml --list-tasks

# Запуск с определёнными тегами
ansible-playbook playbooks/webserver_full.yml --tags security --check

# Проверка синтаксиса
ansible-playbook site.yml --syntax-check

# Запуск с дополнительной детализацией
ansible-playbook site.yml -v
```

---

## Часть 4. Шифрование с Ansible Vault (20 минут)

### Шаг 4.1. Создание зашифрованных файлов с секретами

```bash
# Создание зашифрованного файла (потребуется ввести пароль)
ansible-vault create ~/lab12-complex/group_vars/all/secrets.yml
```

Введите пароль (например: `lab12password`)

Добавьте содержимое:

```yaml
---
# Зашифрованные секреты для всех хостов
db_root_password: "SuperStrongRootPassword123!"
db_app_password: "AppUserPassword456!"
api_key: "sk_test_4eC39HqLyjWDarjtT1zdp7dc"
jwt_secret: "production_jwt_secret_key_2024"
ssl_cert_password: "CertPass789!"
monitoring_token: "monit_token_a1b2c3d4e5f6"
```

### Шаг 4.2. Шифрование отдельных строк

```bash
# Создание файла с обычными переменными и зашифрованными строками
nano ~/lab12-complex/group_vars/all/common.yml
```

```yaml
---
# Обычные (незашифрованные) переменные
app_env: production
app_version: "1.2.3"
log_level: INFO

# Зашифрованная строка - пароль приложения
db_user_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          34333264303732386236383430333133363361653338636233316366333963613734633839383839
          3538626261386464326535303262363337333436636139320a636130383565336130613737323362
          65313731646565313433636466616534323638313830313066386634316366663564356161336238
          3766616430323433650a343566386338363232383966343737353465663332373130363632633134
          6331

# Зашифрованная строка с меткой окружения
monitoring_api_key: !vault |
          $ANSIBLE_VAULT;1.2;AES256;production
          38656166356564306232663864356565316364353637333433396432623661353061323132343034
          6238646230323035333037343637353764376166303134390a626463353132343138356537313262
          64323133386661303363326632643065356239343738613530646130656436303065386533633134
          3563306636343562350a323765633936393466376165313761366634346139326437653562346438
          6339
```

### Шаг 4.3. Создание плейбука с использованием зашифрованных секретов

```bash
nano ~/lab12-complex/playbooks/deploy_with_secrets.yml
```

```yaml
---
- name: Развертывание приложения с зашифрованными секретами
  hosts: webservers,databases
  become: yes
  vars_files:
    - ../group_vars/all/common.yml
    - ../group_vars/all/secrets.yml

  tasks:
    - name: Информация о конфигурации (без вывода секретов)
      ansible.builtin.debug:
        msg:
          - "Окружение: {{ app_env }}"
          - "Версия приложения: {{ app_version }}"
          - "Уровень логирования: {{ log_level }}"
      no_log: false  # Обычные переменные можно показывать

    - name: Создание конфигурационного файла приложения из шаблона
      ansible.builtin.template:
        src: ../templates/app_config.env.j2
        dest: /opt/app/config.env
        mode: '0600'
        owner: "{{ ansible_user }}"
      no_log: true  # Скрываем содержимое (там будут пароли)

    - name: Настройка мониторинга с API ключом
      ansible.builtin.get_url:
        url: "https://api.monitoring.com/agent?token={{ monitoring_token }}"
        dest: /usr/local/bin/monitoring_agent
        mode: '0755'
      no_log: true
      when: inventory_hostname in groups['webservers']

    - name: Создание .env файла для Docker
      ansible.builtin.copy:
        content: |
          DB_PASSWORD={{ db_app_password }}
          API_KEY={{ api_key }}
          JWT_SECRET={{ jwt_secret }}
          MONITORING_TOKEN={{ monitoring_token }}
        dest: /opt/docker/.env
        mode: '0600'
      no_log: true

  post_tasks:
    - name: Проверка наличия конфигурационных файлов
      ansible.builtin.stat:
        path: /opt/app/config.env
      register: config_file

    - name: Результат проверки
      ansible.builtin.debug:
        msg: "Конфигурационный файл успешно создан"
      when: config_file.stat.exists
```

### Шаг 4.4. Создание шаблона конфигурации

```bash
nano ~/lab12-complex/templates/app_config.env.j2
```

```bash
# Ansible сгенерированный конфигурационный файл
# Создан: {{ ansible_date_time.iso8601 }}
# Хост: {{ ansible_hostname }}
# Окружение: {{ app_env }}

DATABASE_URL=postgresql://{{ app_user | default('app') }}:{{ db_app_password }}@{{ db_host | default('localhost') }}:{{ db_port | default(5432) }}/{{ db_name | default('appdb') }}
API_ENDPOINT=https://api.example.com/v1
API_KEY={{ api_key }}
JWT_SECRET={{ jwt_secret }}
LOG_LEVEL={{ log_level }}

MONITORING_ENABLED=true
MONITORING_TOKEN={{ monitoring_token }}

# Секретные настройки
ENCRYPTION_KEY={{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}
```

### Шаг 4.5. Работа с зашифрованными файлами

```bash
# Просмотр зашифрованного файла
ansible-vault view ~/lab12-complex/group_vars/all/secrets.yml
# Введите пароль: lab12password

# Редактирование зашифрованного файла
ansible-vault edit ~/lab12-complex/group_vars/all/secrets.yml

# Смена пароля
ansible-vault rekey ~/lab12-complex/group_vars/all/secrets.yml

# Создание зашифрованной строки для вставки
ansible-vault encrypt_string --name 'new_secret' 'MyNewSecretValue' --ask-vault-pass

# Расшифровка в новый файл (осторожно!)
ansible-vault decrypt ~/lab12-complex/group_vars/all/secrets.yml --output=decrypted_secrets.yml

# Зашифровка существующего файла
echo "backup_password: secret123" > ~/lab12-complex/group_vars/all/backup.yml
ansible-vault encrypt ~/lab12-complex/group_vars/all/backup.yml
```

### Шаг 4.6. Запуск плейбука с Vault

```bash
# Интерактивный ввод пароля
ansible-playbook playbooks/deploy_with_secrets.yml --ask-vault-pass

# Использование файла с паролем
echo "lab12password" > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt

ansible-playbook playbooks/deploy_with_secrets.yml --vault-password-file ~/.vault_pass.txt

# Использование переменной окружения
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass.txt
ansible-playbook playbooks/deploy_with_secrets.yml

# Для нескольких Vault ID (разные пароли для разных файлов)
ansible-playbook playbooks/deploy_with_secrets.yml \
  --vault-id all@~/.vault_pass.txt \
  --vault-id secrets@~/.vault_secrets.txt
```

### Шаг 4.7. Проверка безопасности

```bash
# Проверка, что секреты не попадают в логи
ansible-playbook playbooks/deploy_with_secrets.yml --vault-password-file ~/.vault_pass.txt -v 2>&1 | grep -i "password\|secret\|token"

# Должно быть пусто или только сообщения о том, что вывод скрыт (no_log)
```

---



---

**Лабораторная работа выполнена!** 🎉
