# Лабораторная работа №5: Комплексное управление сервером

## Цель работы
Научиться автоматизировать базовую конфигурацию сервера с помощью Ansible: управление пользователями, пакетами, службами и межсетевым экраном.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12 (ansible-controller)
- **2 управляемых хоста (Managed Nodes)** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

**Выполните на Control Node от пользователя ansible:**

```bash
mkdir -p ~/lab5-config
cd ~/lab5-config
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab5-config/inventory.ini
```

**Содержимое файла:**

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
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab5-config/ansible.cfg
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

## Часть 2. Управление пользователями и группами 

### Шаг 2.1. Создание playbook для управления пользователями

Создайте файл `01-users.yml`:

```bash
nano ~/lab5-config/01-users.yml
```

**Содержимое файла:**

```yaml
---
- name: Управление пользователями и группами
  hosts: all
  
  tasks:
    # ========== ГРУППЫ ==========
    - name: Создание группы администраторов
      ansible.builtin.group:
        name: admin
        gid: 5000
        state: present
    
    - name: Создание группы для разработчиков
      ansible.builtin.group:
        name: developers
        gid: 5001
        state: present
    
    - name: Создание группы для приложения
      ansible.builtin.group:
        name: myapp
        system: yes
        state: present
    
    # ========== ПОЛЬЗОВАТЕЛИ ==========
    - name: Создание администратора alice
      ansible.builtin.user:
        name: alice
        uid: 5000
        group: admin
        groups: sudo
        append: yes
        home: /home/alice
        shell: /bin/bash
        state: present
        create_home: yes
    
    - name: Создание разработчика bob
      ansible.builtin.user:
        name: bob
        uid: 5001
        group: developers
        home: /home/bob
        shell: /bin/bash
        state: present
        create_home: yes
    
    - name: Создание системного пользователя для приложения
      ansible.builtin.user:
        name: myapp
        uid: 900
        group: myapp
        system: yes
        shell: /sbin/nologin
        home: /opt/myapp
        create_home: yes
        state: present
    
    # ========== SSH-КЛЮЧИ ==========
    - name: Создание .ssh директории для alice
      ansible.builtin.file:
        path: /home/alice/.ssh
        state: directory
        owner: alice
        group: admin
        mode: '0700'
    
    - name: Добавление SSH-ключа alice
      ansible.builtin.authorized_key:
        user: alice
        state: present
        key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC3... alice@workstation"
    
    # ========== НАСТРОЙКА SUDO ==========
    - name: Настройка sudo для группы admin
      ansible.builtin.copy:
        content: |
          # Группа admin может выполнять любые команды
          %admin ALL=(ALL) ALL
        dest: /etc/sudoers.d/admin-group
        mode: '0440'
        validate: 'visudo -cf %s'
```

### Шаг 2.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 01-users.yml --syntax-check

# Просмотр целевых хостов
ansible-playbook 01-users.yml --list-hosts

# Режим симуляции
ansible-playbook 01-users.yml --check

# Реальное выполнение
ansible-playbook 01-users.yml
```

### Шаг 2.3. Проверка результатов

```bash
# Проверка создания пользователей на web-server
ssh ansible@192.168.1.11 "id alice; id bob; id myapp"

# Проверка групп
ssh ansible@192.168.1.11 "getent group admin; getent group developers; getent group myapp"

# Проверка sudo прав
ssh ansible@192.168.1.12 "cat /etc/sudoers.d/admin-group"
```

---

## Часть 3. Управление пакетами и репозиториями (20 минут)

### Шаг 3.1. Создание playbook для управления пакетами

Создайте файл `02-packages.yml`:

```bash
nano ~/lab5-config/02-packages.yml
```

**Содержимое файла:**

```yaml
---
- name: Управление пакетами и репозиториями
  hosts: all
  
  tasks:
    # ========== ОБНОВЛЕНИЕ СИСТЕМЫ ==========
    - name: Обновление кэша пакетов
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600
    
    # ========== БАЗОВЫЕ ПАКЕТЫ ДЛЯ ВСЕХ СЕРВЕРОВ ==========
    - name: Установка общих пакетов
      ansible.builtin.apt:
        name:
          - curl
          - wget
          - git
          - vim
          - htop
          - net-tools
          - ca-certificates
          - gnupg
          - lsb-release
        state: present
    
    # ========== ПАКЕТЫ ДЛЯ ВЕБ-СЕРВЕРА ==========
    - name: Установка веб-пакетов
      ansible.builtin.apt:
        name:
          - nginx
          - php-fpm
          - php-mysql
          - php-curl
          - php-gd
          - mariadb-client
        state: present
      when: "'webservers' in group_names"
    
    # ========== ПАКЕТЫ ДЛЯ СЕРВЕРА БД ==========
    - name: Установка пакетов БД
      ansible.builtin.apt:
        name:
          - mariadb-server
          - mariadb-client
          - python3-pymysql
        state: present
      when: "'databases' in group_names"
    
    # ========== ДОБАВЛЕНИЕ РЕПОЗИТОРИЯ DOCKER (только для веб-сервера) ==========
    - name: Добавление GPG-ключа Docker
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/debian/gpg
        state: present
      when: "'webservers' in group_names"
    
    - name: Добавление репозитория Docker
      ansible.builtin.apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_distribution_release }} stable"
        state: present
        update_cache: yes
      when: "'webservers' in group_names"
    
    - name: Установка Docker
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present
      when: "'webservers' in group_names"
    
    # ========== УДАЛЕНИЕ НЕНУЖНЫХ ПАКЕТОВ ==========
    - name: Удаление ненужного ПО
      ansible.builtin.apt:
        name:
          - telnet
          - ftp
        state: absent
```

### Шаг 3.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 02-packages.yml --syntax-check

# Режим симуляции
ansible-playbook 02-packages.yml --check

# Реальное выполнение
ansible-playbook 02-packages.yml
```

### Шаг 3.3. Проверка результатов

```bash
# Проверка установленных пакетов на web-server
ssh ansible@192.168.1.11 "dpkg -l | grep -E 'nginx|php|docker'"

# Проверка установленных пакетов на db-server
ssh ansible@192.168.1.12 "dpkg -l | grep mariadb"

# Проверка Docker
ssh ansible@192.168.1.11 "docker --version"
```

---

## Часть 4. Управление службами 

### Шаг 4.1. Создание playbook для управления службами

Создайте файл `03-services.yml`:

```bash
nano ~/lab5-config/03-services.yml
```

**Содержимое файла:**

```yaml
---
- name: Управление службами
  hosts: all
  
  tasks:
    # ========== ВЕБ-СЕРВЕР ==========
    - name: Запуск и включение nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
      when: "'webservers' in group_names"
    
    - name: Запуск и включение php-fpm
      ansible.builtin.service:
        name: "php{{ ansible_distribution_version | regex_replace('\\..*', '') }}-fpm"
        state: started
        enabled: yes
      when: "'webservers' in group_names"
      ignore_errors: yes
    
    - name: Запуск и включение Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes
      when: "'webservers' in group_names"
    
    # ========== СЕРВЕР БД ==========
    - name: Запуск и включение MariaDB
      ansible.builtin.service:
        name: mariadb
        state: started
        enabled: yes
      when: "'databases' in group_names"
    
    # ========== ОБЩИЕ СЛУЖБЫ ==========
    - name: Запуск и включение SSH
      ansible.builtin.service:
        name: sshd
        state: started
        enabled: yes
    
    # ========== ПРОВЕРКА СТАТУСА ==========
    - name: Проверка статуса SSH
      ansible.builtin.service_facts:
    
    - name: Вывод статуса SSH
      ansible.builtin.debug:
        msg: "SSH статус: {{ ansible_facts.services['ssh.service'].state }}"
```

### Шаг 4.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 03-services.yml --syntax-check

# Режим симуляции
ansible-playbook 03-services.yml --check

# Реальное выполнение
ansible-playbook 03-services.yml
```

### Шаг 4.3. Проверка результатов

```bash
# Проверка статуса служб на web-server
ssh ansible@192.168.1.11 "systemctl status nginx --no-pager | head -5"
ssh ansible@192.168.1.11 "systemctl status docker --no-pager | head -5"

# Проверка статуса служб на db-server
ssh ansible@192.168.1.12 "systemctl status mariadb --no-pager | head -5"

# Проверка статуса SSH на обоих серверах
ansible all -m shell -a "systemctl is-active ssh"
```

---

## Часть 5. Управление межсетевым экраном 

### Шаг 5.1. Создание playbook для настройки UFW

Создайте файл `04-firewall.yml`:

```bash
nano ~/lab5-config/04-firewall.yml
```

**Содержимое файла:**

```yaml
---
- name: Настройка межсетевого экрана
  hosts: all
  
  tasks:
    # ========== СНАЧАЛА РАЗРЕШАЕМ SSH! ==========
    - name: Разрешение SSH (важно!)
      ansible.builtin.ufw:
        rule: allow
        port: '22'
        proto: tcp
        comment: "Allow SSH"
    
    # ========== ОБЩИЕ ПРАВИЛА ==========
    - name: Разрешение HTTP
      ansible.builtin.ufw:
        rule: allow
        port: '80'
        proto: tcp
        comment: "Allow HTTP"
    
    - name: Разрешение HTTPS
      ansible.builtin.ufw:
        rule: allow
        port: '443'
        proto: tcp
        comment: "Allow HTTPS"
    
    # ========== ПРАВИЛА ДЛЯ ВЕБ-СЕРВЕРА ==========
    - name: Разрешение порта 8080 (для тестов)
      ansible.builtin.ufw:
        rule: allow
        port: '8080'
        proto: tcp
        comment: "Test port"
      when: "'webservers' in group_names"
    
    # ========== ПРАВИЛА ДЛЯ СЕРВЕРА БД ==========
    - name: Разрешение MySQL (только с веб-сервера)
      ansible.builtin.ufw:
        rule: allow
        port: '3306'
        proto: tcp
        from_ip: '192.168.1.11'
        comment: "Allow MySQL from web-server"
      when: "'databases' in group_names"
    
    # ========== ОГРАНИЧЕНИЕ SSH ==========
    - name: Ограничение SSH (защита от брутфорса)
      ansible.builtin.ufw:
        rule: limit
        port: '22'
        proto: tcp
        comment: "Limit SSH connections"
    
    # ========== ЗАПРЕТ ПО УМОЛЧАНИЮ ==========
    - name: Включение UFW с политикой deny
      ansible.builtin.ufw:
        state: enabled
        policy: deny
        direction: incoming
```

**Важно:** Последовательность операций в этом playbook имеет значение:
1. Сначала добавляются разрешающие правила
2. В конце включается UFW с политикой запрета

### Шаг 5.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 04-firewall.yml --syntax-check

# Режим симуляции (не полностью работает с UFW, но проверит синтаксис)
ansible-playbook 04-firewall.yml --check

# Реальное выполнение
ansible-playbook 04-firewall.yml
```

### Шаг 5.3. Проверка результатов

```bash
# Проверка статуса UFW на web-server
ssh ansible@192.168.1.11 "sudo ufw status verbose"

# Проверка статуса UFW на db-server
ssh ansible@192.168.1.12 "sudo ufw status verbose"

# Проверка политики
ansible all -m shell -a "sudo ufw status | grep -E 'Default:|Status:'"
```

---



5. Создали комплексную конфигурацию сервера

**Готовность:** Вы освоили все основные аспекты базовой конфигурации сервера с помощью Ansible.
