# Лабораторная работа №12
## Комплексные проекты автоматизации Ansible
### Стенд: 3 виртуальные машины Debian 12 (например, server1, server2, server3)

---

## Цель работы
Научиться использовать обработчики, динамическое/статическое включение задач и шифрование секретов в Ansible на примере трёх Debian 12.

---

## Подготовка стенда

```bash
# Проверка доступности хостов
ansible all -m ping

# Инвентарь (inventory.ini)
[web]
debian-web ansible_host=192.168.1.10 

[db]
debian-db ansible_host=192.168.1.20 



[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

---

## Часть 1. Уведомления и обработчики (Handlers)

### Задание: Настроить Nginx с перезапуском только при изменении конфигов

**1.1. Создайте структуру каталогов:**

```bash
mkdir -p lab12/handlers
cd lab12
```

**1.2. Создайте плейбук `nginx_with_handlers.yml`:**

```yaml
---
- name: Установка и настройка Nginx с обработчиками
  hosts: web
  become: yes
  gather_facts: yes

  tasks:
    - name: Установка Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes
      notify: 
        - запустить nginx
        - проверить статус

    - name: Создание кастомной страницы 403
      ansible.builtin.copy:
        content: "<h1>Доступ запрещен</h1>"
        dest: /var/www/html/403.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: перезагрузить nginx

    - name: Настройка default site (только при изменении)
      ansible.builtin.template:
        src: default.conf.j2
        dest: /etc/nginx/sites-available/default
        backup: yes
      notify: 
        - проверить конфиг
        - перезагрузить nginx

  handlers:
    - name: запустить nginx
      ansible.builtin.systemd:
        name: nginx
        state: started
        enabled: yes

    - name: проверить конфиг
      ansible.builtin.command: nginx -t
      register: nginx_test
      changed_when: false
      listen: "перезагрузить nginx"

    - name: перезагрузить nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
      when: nginx_test.rc == 0

    - name: проверить статус
      ansible.builtin.systemd:
        name: nginx
      register: nginx_status
      changed_when: false
      listen: "запустить nginx"
```

**1.3. Создайте шаблон `templates/default.conf.j2`:**

```nginx
server {
    listen {{ ansible_default_ipv4.port }} default_server;
    listen [::]:80 default_server;
    
    root /var/www/html;
    index index.html index.htm;

    server_name {{ ansible_fqdn }};

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 403 /403.html;
    location = /403.html {
        root /var/www/html;
        internal;
    }
}
```

**1.4. Запустите и проверьте:**

```bash
# Первый запуск - установка и запуск
ansible-playbook -i inventory.ini nginx_with_handlers.yml

# Второй запуск - без изменений (handlers НЕ сработают)
ansible-playbook -i inventory.ini nginx_with_handlers.yml

# Измените конфиг в шаблоне и запустите снова
ansible-playbook -i inventory.ini nginx_with_handlers.yml --diff
```

**1.5. Эксперимент с `flush_handlers`:**

Создайте `nginx_flush_test.yml`:

```yaml
---
- name: Тест flush_handlers
  hosts: web
  become: yes
  
  tasks:
    - name: Изменить конфиг
      ansible.builtin.lineinfile:
        path: /etc/nginx/sites-available/default
        regexp: '^(\s*)try_files'
        line: '\1try_files $uri $uri/ /index.html;'
        backrefs: yes
      notify: перезагрузить nginx
    
    - name: ПРИНУДИТЕЛЬНЫЙ ЗАПУСК HANDLERS (до окончания play)
      ansible.builtin.meta: flush_handlers
    
    - name: Проверка Nginx после перезагрузки
      ansible.builtin.uri:
        url: http://localhost
        return_content: yes
      register: webpage
      until: webpage.status == 200
      retries: 3
      delay: 2
    
    - debug:
        msg: "Nginx успешно перезагружен и отвечает"

  handlers:
    - name: перезагрузить nginx
      ansible.builtin.systemd:
        name: nginx
        state: reloaded
```

---

## Часть 2. Включение и импорт задач

### Задание: Разделить конфигурацию на модули с динамическим и статическим импортом

**2.1. Создайте подключаемые файлы задач:**

`tasks/base_debian12.yml` (базовая настройка Debian 12):
```yaml
---
- name: Обновление apt кэша
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Установка базовых пакетов
  ansible.builtin.apt:
    name:
      - curl
      - wget
      - git
      - vim
      - htop
      - net-tools
    state: present

- name: Настройка локали
  ansible.builtin.locale_gen:
    name: ru_RU.UTF-8
    state: present

- name: Установка часового пояса
  ansible.builtin.timezone:
    name: Europe/Moscow
```

`tasks/security.yml` (безопасность):
```yaml
---
- name: Отключение root SSH (если есть другой пользователь)
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'

- name: Установка fail2ban
  ansible.builtin.apt:
    name: fail2ban
    state: present

- name: Включение fail2ban
  ansible.builtin.systemd:
    name: fail2ban
    state: started
    enabled: yes

- name: Настройка UFW (если используется)
  community.general.ufw:
    rule: allow
    port: "{{ item }}"
    proto: tcp
  loop:
    - '22'
    - '80'
    - '443'
  when: ansible_os_family == "Debian"
```

`tasks/database_setup.yml` (только для БД):
```yaml
---
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

- name: Создание тестовой БД
  become_user: postgres
  ansible.builtin.postgresql_db:
    name: test_lab12
    state: present
  ignore_errors: yes  # Если нет модуля postgresql
```

**2.2. Создайте плейбук с разными типами включения:**

`complex_setup.yml`:
```yaml
---
- name: Комплексная настройка всех серверов
  hosts: all
  become: yes
  
  tasks:
    # СТАТИЧЕСКИЙ импорт (загружается ДО выполнения)
    - name: Базовая настройка (статически)
      ansible.builtin.import_tasks: tasks/base_debian12.yml
      tags: always
    
    # ДИНАМИЧЕСКОЕ включение с условием
    - name: Настройка безопасности (динамически)
      ansible.builtin.include_tasks: tasks/security.yml
      when: ansible_os_family == "Debian"
      tags: security
    
    # ДИНАМИЧЕСКОЕ включение с циклом (пример)
    - name: Создание пользователей
      ansible.builtin.include_tasks: tasks/create_user.yml
      loop:
        - { name: "ansible_admin", shell: "/bin/bash" }
        - { name: "deploy_user", shell: "/bin/bash" }
      vars:
        user_name: "{{ item.name }}"
        user_shell: "{{ item.shell }}"

# ВТОРОЙ PLAY - для базы данных со статическим импортом задач
- name: Настройка БД
  hosts: db
  become: yes
  tasks:
    - name: Статический импорт задач БД
      ansible.builtin.import_tasks: tasks/database_setup.yml
      when: inventory_hostname == "debian-db"
```

**2.3. Создайте подключаемый файл `tasks/create_user.yml`:**

```yaml
---
- name: Создание пользователя {{ user_name }}
  ansible.builtin.user:
    name: "{{ user_name }}"
    shell: "{{ user_shell }}"
    state: present
    create_home: yes

- name: Добавление SSH ключа для {{ user_name }}
  ansible.builtin.authorized_key:
    user: "{{ user_name }}"
    state: present
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
  ignore_errors: yes
```

**2.4. Создайте сборку из нескольких плейбуков (статический импорт):**

`site.yml` (главный плейбук):
```yaml
---
# Статический импорт плейбуков (загружаются все сразу)
- import_playbook: playbooks/common.yml
- import_playbook: playbooks/webservers.yml
- import_playbook: playbooks/databases.yml
- import_playbook: playbooks/proxies.yml
```

Создайте каталог `playbooks/` и файлы:

`playbooks/common.yml`:
```yaml
---
- name: Общая настройка всех хостов
  hosts: all
  become: yes
  tasks:
    - name: Установка ntp
      ansible.builtin.apt:
        name: systemd-timesyncd
        state: present
    - name: Включение синхронизации времени
      ansible.builtin.systemd:
        name: systemd-timesyncd
        state: started
        enabled: yes
```

`playbooks/webservers.yml`:
```yaml
---
- name: Настройка web серверов
  hosts: web
  become: yes
  tasks:
    - name: Установка Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
```

**2.5. Запуск и проверка импортов:**

```bash
# Запуск комплексного плейбука
ansible-playbook -i inventory.ini complex_setup.yml

# Проверка, какие задачи будут выполнены (import_tasks видны)
ansible-playbook -i inventory.ini complex_setup.yml --list-tasks

# Запуск только тегов security
ansible-playbook -i inventory.ini complex_setup.yml --tags security

# Запуск главного плейбука с импортом
ansible-playbook -i inventory.ini site.yml
```

---

## Часть 3. Ansible Vault (шифрование)

### Задание: Зашифровать секреты для PostgreSQL и тестового приложения

**3.1. Создайте зашифрованный файл с паролями:**

```bash
# Способ 1: Создание нового зашифрованного файла
ansible-vault create group_vars/all/vault.yml
# Введите пароль (например: lab12vault)
# Добавьте содержимое:
```

```yaml
# group_vars/all/vault.yml
db_root_password: "SuperSecretRootPass123!"
db_app_password: "AppUserPass456!"
api_key: "sk_live_4eC39HqLyjWDarjtT1zdp7dc"
jwt_secret: "change_this_in_production_$(openssl rand -hex 32)"
```

**3.2. Шифрование отдельных строк в обычном файле:**

Создайте `group_vars/all/vars.yml`:
```yaml
# Обычные переменные
db_host: "localhost"
db_port: 5432
db_name: "lab12_production"
app_user: "ansible_app"

# Зашифрованные строки
db_app_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653236333662633166643938383761303635663035323637303562363665363536353765
          6131393638643261623332356264396335366232333733380a373738363737333438306365306562
          38656265306634633032373938326234313932303964623265636161313937653638356332616432
          3865656637343832640a613163656362306531346565343731643431333330346161656461356564
          6432

admin_password: !vault |
          $ANSIBLE_VAULT;1.2;AES256;lab12_admin
          61323732356262363661306666623630633830663466336265646537346337373435386433393435
          3735653637613662636231623363613564663563336234300a303866636561656538373433386162
          32363136356238353536333234646631383036396261366338353239653531626139316330383632
          3037303861376132650a376464386637666534326232643631336561336265363564343266613639
          3265
```

**3.3. Создайте плейбук с использованием vault-переменных:**

`deploy_app_with_vault.yml`:
```yaml
---
- name: Развертывание приложения с секретами
  hosts: web,db
  become: yes
  vars_files:
    - group_vars/all/vars.yml
    - group_vars/all/vault.yml

  tasks:
    - name: Проверка наличия зашифрованных переменных
      debug:
        msg: 
          - "DB Password exists: {{ db_app_password is defined }}"
          - "API Key exists: {{ api_key is defined }}"
      no_log: true  # Скрываем вывод секретов

    - name: Создание конфига приложения из шаблона
      ansible.builtin.template:
        src: app_config.j2
        dest: /opt/app/config.env
        mode: '0600'
      no_log: true
    
    - name: Создание пользователя PostgreSQL с паролем из vault
      become_user: postgres
      ansible.builtin.postgresql_user:
        name: "{{ app_user }}"
        password: "{{ db_app_password }}"
        state: present
      ignore_errors: yes
      no_log: true
      when: inventory_hostname in groups['db']

- name: Настройка мониторинга с API ключом
  hosts: web
  become: yes
  vars_files:
    - group_vars/all/vault.yml
  
  tasks:
    - name: Установка агента мониторинга
      ansible.builtin.get_url:
        url: "https://api.monitoring.com/agent?key={{ api_key }}"
        dest: /tmp/monitoring_agent.sh
        mode: '0755'
      no_log: true
```

**3.4. Создайте шаблон `templates/app_config.j2`:**

```bash
# Конфигурация приложения
DB_HOST={{ db_host }}
DB_PORT={{ db_port }}
DB_NAME={{ db_name }}
DB_USER={{ app_user }}
DB_PASSWORD={{ db_app_password }}
API_KEY={{ api_key }}
JWT_SECRET={{ jwt_secret }}
ENVIRONMENT=production
DEPLOY_DATE={{ ansible_date_time.iso8601 }}
```

**3.5. Управление зашифрованными файлами:**

```bash
# Просмотр зашифрованного файла
ansible-vault view group_vars/all/vault.yml
# Введите пароль: lab12vault

# Редактирование
ansible-vault edit group_vars/all/vault.yml

# Смена пароля
ansible-vault rekey group_vars/all/vault.yml

# Расшифровка (осторожно, будет открытый текст!)
ansible-vault decrypt group_vars/all/vault.yml --output=decrypted_vault.yml

# Зашифровка существующего файла
ansible-vault encrypt group_vars/all/secrets.yml --ask-vault-pass

# Создание зашифрованной строки для вставки
ansible-vault encrypt_string --name 'new_secret' 'MySecretValue123!' --ask-vault-pass
```

**3.6. Запуск плейбуков с Vault:**

```bash
# Интерактивный ввод пароля
ansible-playbook -i inventory.ini deploy_app_with_vault.yml --ask-vault-pass

# Использование файла с паролем (безопаснее в CI/CD)
echo "lab12vault" > .vault_pass
chmod 600 .vault_pass
ansible-playbook -i inventory.ini deploy_app_with_vault.yml --vault-password-file .vault_pass

# Несколько разных vault-id (если разные пароли для разных файлов)
ansible-playbook -i inventory.ini deploy_app_with_vault.yml \
  --vault-id all@.vault_pass \
  --vault-id db@.vault_pass_db
```

---
