# Лабораторная работа №4: Автоматизация управления файлами

## Цель работы
Научиться автоматизировать базовые операции с файлами и директориями с помощью Ansible: создание, копирование, перемещение, удаление и управление правами доступа.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12 (ansible-controller)
- **2 управляемых хоста (Managed Nodes)** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды 

### Шаг 1.1. Создание директории для лабораторной

**Выполните на Control Node от пользователя ansible:**

```bash
mkdir -p ~/lab4-files
cd ~/lab4-files
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab4-files/inventory.ini
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
nano ~/lab4-files/ansible.cfg
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

### Шаг 1.4. Создание директории для файлов

```bash
mkdir -p ~/lab4-files/files
```

---

## Часть 2. Создание файлов и директорий

### Шаг 2.1. Создание playbook для создания директорий

Создайте файл `01-create-directories.yml`:

```bash
nano ~/lab4-files/01-create-directories.yml
```

**Содержимое файла:**

```yaml
---
- name: Создание директорий на всех серверах
  hosts: all
  become: yes
  
  tasks:
    - name: Создание директории /opt/lab4
      ansible.builtin.file:
        path: /opt/lab4
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
    
    - name: Создание директории /opt/lab4/data
      ansible.builtin.file:
        path: /opt/lab4/data
        state: directory
        owner: ansible
        group: ansible
        mode: '0775'
    
    - name: Создание директории /opt/lab4/backup
      ansible.builtin.file:
        path: /opt/lab4/backup
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
    
    - name: Создание директории /opt/lab4/logs
      ansible.builtin.file:
        path: /opt/lab4/logs
        state: directory
        owner: ansible
        group: ansible
        mode: '0755'
```

### Шаг 2.2. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 01-create-directories.yml --syntax-check

# Просмотр целевых хостов
ansible-playbook 01-create-directories.yml --list-hosts

# Режим симуляции
ansible-playbook 01-create-directories.yml --check

# Реальное выполнение
ansible-playbook 01-create-directories.yml
```

### Шаг 2.3. Проверка создания директорий

```bash
# Проверка на web-server
ssh ansible@192.168.1.11 "ls -la /opt/lab4"

# Проверка на db-server
ssh ansible@192.168.1.12 "ls -la /opt/lab4"
```

**Ожидаемый вывод:**
```
total 20
drwxr-xr-x 5 ansible ansible 4096 ... .
drwxr-xr-x 3 root    root    4096 ... ..
drwxrwxr-x 2 ansible ansible 4096 ... data
drwxr-xr-x 2 ansible ansible 4096 ... backup
drwxr-xr-x 2 ansible ansible 4096 ... logs
```

### Шаг 2.4. Создание playbook для создания файлов

Создайте файл `02-create-files.yml`:

```bash
nano ~/lab4-files/02-create-files.yml
```

**Содержимое файла:**

```yaml
---
- name: Создание файлов на всех серверах
  hosts: all
  become: yes
  
  tasks:
    - name: Создание пустого файла README.md
      ansible.builtin.file:
        path: /opt/lab4/README.md
        state: touch
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Создание файла с содержимым info.txt
      ansible.builtin.copy:
        content: |
          ====================================
          Информация о сервере
          ====================================
          Хост: {{ ansible_hostname }}
          ОС: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Ядро: {{ ansible_kernel }}
          CPU: {{ ansible_processor_cores }} ядер
          RAM: {{ ansible_memtotal_mb }} MB
          Дата создания: {{ ansible_date_time.date }}
          ====================================
        dest: /opt/lab4/data/info.txt
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Создание файла конфигурации
      ansible.builtin.copy:
        content: |
          # Конфигурационный файл
          server_name={{ ansible_hostname }}
          environment=lab
          debug_mode=true
          log_level=INFO
        dest: /opt/lab4/config.ini
        owner: ansible
        group: ansible
        mode: '0644'
```

### Шаг 2.5. Выполнение playbook

```bash
# Проверка синтаксиса
ansible-playbook 02-create-files.yml --syntax-check

# Режим симуляции
ansible-playbook 02-create-files.yml --check

# Реальное выполнение
ansible-playbook 02-create-files.yml
```

### Шаг 2.6. Проверка созданных файлов

```bash
# Просмотр всех файлов
ansible all -m shell -a "ls -la /opt/lab4/"

# Просмотр содержимого info.txt
ansible all -m shell -a "cat /opt/lab4/data/info.txt"

# Просмотр конфигурации
ansible all -m shell -a "cat /opt/lab4/config.ini"
```

---

## Часть 3. Копирование файлов (15 минут)

### Шаг 3.1. Создание локальных файлов для копирования

**На Control Node:**

```bash
# Создание тестового HTML файла
cat > ~/lab4-files/files/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Lab4 Test Page</title>
</head>
<body>
    <h1>Автоматизация управления файлами</h1>
    <p>Этот файл был скопирован с помощью Ansible</p>
    <p>Сервер: {{ ansible_hostname }}</p>
</body>
</html>
EOF

# Создание скрипта для мониторинга
cat > ~/lab4-files/files/check_disk.sh << 'EOF'
#!/bin/bash
echo "=== Информация о дисках ==="
df -h | grep -v tmpfs
echo ""
echo "=== Использование диска в /opt/lab4 ==="
du -sh /opt/lab4/*
EOF

chmod +x ~/lab4-files/files/check_disk.sh

# Создание текстового файла
echo "Это тестовый файл для копирования от $(date)" > ~/lab4-files/files/test.txt
```

### Шаг 3.2. Создание playbook для копирования файлов

Создайте файл `03-copy-files.yml`:

```bash
nano ~/lab4-files/03-copy-files.yml
```

**Содержимое файла:**

```yaml
---
- name: Копирование файлов на серверы
  hosts: all
  become: yes
  
  tasks:
    - name: Копирование HTML файла
      ansible.builtin.copy:
        src: files/index.html
        dest: /opt/lab4/data/index.html
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Копирование скрипта мониторинга
      ansible.builtin.copy:
        src: files/check_disk.sh
        dest: /opt/lab4/check_disk.sh
        owner: ansible
        group: ansible
        mode: '0755'
    
    - name: Копирование текстового файла
      ansible.builtin.copy:
        src: files/test.txt
        dest: /opt/lab4/data/test.txt
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Копирование с созданием бэкапа
      ansible.builtin.copy:
        src: files/test.txt
        dest: /opt/lab4/backup/test_backup.txt
        owner: ansible
        group: ansible
        mode: '0644'
        backup: yes
```

### Шаг 3.3. Выполнение playbook

```bash
# Проверка
ansible-playbook 03-copy-files.yml --syntax-check

# Режим симуляции
ansible-playbook 03-copy-files.yml --check

# Реальное выполнение
ansible-playbook 03-copy-files.yml
```

### Шаг 3.4. Проверка скопированных файлов

```bash
# Проверка всех скопированных файлов
ansible all -m shell -a "ls -la /opt/lab4/data/"
ansible all -m shell -a "ls -la /opt/lab4/"

# Выполнение скрипта мониторинга на web-server
ssh ansible@192.168.1.11 "bash /opt/lab4/check_disk.sh"

# Выполнение на db-server
ssh ansible@192.168.1.12 "bash /opt/lab4/check_disk.sh"
```

---

## Часть 4. Перемещение и переименование файлов (15 минут)

### Шаг 4.1. Создание playbook для перемещения файлов

Создайте файл `04-move-files.yml`:

```bash
nano ~/lab4-files/04-move-files.yml
```

**Содержимое файла:**

```yaml
---
- name: Перемещение и переименование файлов
  hosts: all
  become: yes
  
  tasks:
    # Перемещение файла в другую директорию
    - name: Перемещение test.txt в директорию backup
      ansible.builtin.copy:
        src: /opt/lab4/data/test.txt
        dest: /opt/lab4/backup/test.txt
        remote_src: yes
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Удаление исходного файла после перемещения
      ansible.builtin.file:
        path: /opt/lab4/data/test.txt
        state: absent
    
    # Переименование файла
    - name: Переименование config.ini в settings.ini
      ansible.builtin.copy:
        src: /opt/lab4/config.ini
        dest: /opt/lab4/settings.ini
        remote_src: yes
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Удаление старого конфига
      ansible.builtin.file:
        path: /opt/lab4/config.ini
        state: absent
    
    # Перемещение с переименованием
    - name: Перемещение и переименование HTML файла
      ansible.builtin.copy:
        src: /opt/lab4/data/index.html
        dest: /opt/lab4/index.html
        remote_src: yes
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Удаление старого HTML файла
      ansible.builtin.file:
        path: /opt/lab4/data/index.html
        state: absent
    
    # Создание файла в одной директории и перемещение в другую
    - name: Создание временного файла
      ansible.builtin.copy:
        content: "Это временный файл для перемещения"
        dest: /tmp/temp_move.txt
        owner: ansible
        mode: '0644'
    
    - name: Перемещение временного файла в lab4
      ansible.builtin.copy:
        src: /tmp/temp_move.txt
        dest: /opt/lab4/data/moved_file.txt
        remote_src: yes
        owner: ansible
        group: ansible
        mode: '0644'
    
    - name: Удаление временного файла
      ansible.builtin.file:
        path: /tmp/temp_move.txt
        state: absent
```

**Примечание:** В Ansible нет прямого модуля `move`, поэтому перемещение выполняется через `copy` с `remote_src: yes` и последующее удаление исходника.

### Шаг 4.2. Выполнение playbook

```bash
# Проверка
ansible-playbook 04-move-files.yml --syntax-check

# Режим симуляции
ansible-playbook 04-move-files.yml --check

# Реальное выполнение
ansible-playbook 04-move-files.yml
```

### Шаг 4.3. Проверка результатов перемещения

```bash
# Проверка структуры после перемещения
ansible all -m shell -a "find /opt/lab4 -type f -name '*.txt' -o -name '*.ini' -o -name '*.html'"

# Проверка содержимого перемещённых файлов
ansible all -m shell -a "cat /opt/lab4/settings.ini"
ansible all -m shell -a "cat /opt/lab4/index.html"
ansible all -m shell -a "cat /opt/lab4/data/moved_file.txt"
```

---

## Часть 5. Управление правами доступа (15 минут)

### Шаг 5.1. Создание playbook для управления правами

Создайте файл `05-permissions.yml`:

```bash
nano ~/lab4-files/05-permissions.yml
```

**Содержимое файла:**

```yaml
---
- name: Управление правами доступа
  hosts: all
  become: yes
  
  tasks:
    # Изменение прав на файл
    - name: Изменение прав на скрипт (только чтение)
      ansible.builtin.file:
        path: /opt/lab4/check_disk.sh
        mode: '0755'
        owner: root
        group: root
    
    # Изменение прав на директорию
    - name: Изменение прав на директорию data
      ansible.builtin.file:
        path: /opt/lab4/data
        mode: '0750'
        owner: ansible
        group: ansible
    
    # Изменение прав на файл конфигурации
    - name: Защита конфигурационного файла
      ansible.builtin.file:
        path: /opt/lab4/settings.ini
        mode: '0600'
        owner: ansible
        group: ansible
    
    # Изменение владельца директории backup
    - name: Смена владельца backup директории
      ansible.builtin.file:
        path: /opt/lab4/backup
        owner: root
        group: ansible
        mode: '0770'
    
    # Рекурсивное изменение прав
    - name: Рекурсивное изменение прав на всю директорию lab4
      ansible.builtin.file:
        path: /opt/lab4
        owner: ansible
        group: ansible
        recurse: yes
```

### Шаг 5.2. Выполнение playbook

```bash
# Проверка
ansible-playbook 05-permissions.yml --syntax-check

# Реальное выполнение
ansible-playbook 05-permissions.yml
```

### Шаг 5.3. Проверка прав доступа

```bash
# Проверка прав на web-server
ssh ansible@192.168.1.11 "ls -la /opt/lab4/"
ssh ansible@192.168.1.11 "ls -la /opt/lab4/data/"
ssh ansible@192.168.1.11 "ls -la /opt/lab4/settings.ini"

# Проверка на db-server
ssh ansible@192.168.1.12 "ls -la /opt/lab4/"
```

---

## Часть 6. Удаление файлов и директорий (10 минут)

### Шаг 6.1. Создание playbook для удаления

Создайте файл `06-delete-files.yml`:

```bash
nano ~/lab4-files/06-delete-files.yml
```

**Содержимое файла:**

```yaml
---
- name: Удаление файлов и директорий
  hosts: all
  become: yes
  
  tasks:
    # Удаление конкретного файла
    - name: Удаление временного файла
      ansible.builtin.file:
        path: /opt/lab4/data/moved_file.txt
        state: absent
    
    # Удаление файла по условию (проверка существования)
    - name: Проверка существования файла перед удалением
      ansible.builtin.stat:
        path: /opt/lab4/backup/test.txt
      register: test_file
    
    - name: Удаление test.txt если существует
      ansible.builtin.file:
        path: /opt/lab4/backup/test.txt
        state: absent
      when: test_file.stat.exists
    
    # Удаление всех .txt файлов (через shell)
    - name: Удаление всех .txt файлов
      ansible.builtin.shell: find /opt/lab4 -name "*.txt" -type f -delete
      args:
        removes: /opt/lab4
```

### Шаг 6.2. Выполнение playbook

```bash
# Проверка
ansible-playbook 06-delete-files.yml --syntax-check

# Режим симуляции
ansible-playbook 06-delete-files.yml --check

# Реальное выполнение
ansible-playbook 06-delete-files.yml
```



