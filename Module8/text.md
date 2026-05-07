# Лабораторная работа №8: Работа с текстом в Ansible

## Цель работы
Научиться управлять текстовыми файлами с помощью Ansible: добавлять, изменять, удалять строки и блоки текста, использовать регулярные выражения для поиска и замены.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab8-text
cd ~/lab8-text
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab8-text/inventory.ini
```

```ini
[webservers]
web-server ansible_host=ip server1

[databases]
db-server ansible_host=ipserver 2

[all:children]
webservers
databases

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab8-text/ansible.cfg
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

### Шаг 1.4. Создание тестовых файлов на удалённых хостах

```bash
# Создание тестового файла на web-server
ssh ansible@192.168.1.11 "sudo tee /etc/test_config.txt << 'EOF'
# Это конфигурационный файл
# Настройки сервера

server_name=localhost
port=8080
debug_mode=false

# Дополнительные настройки
max_connections=100
timeout=30

# Закомментированная настройка
# cache_size=256
EOF"

# Создание тестового файла на db-server
ssh ansible@192.168.1.12 "sudo tee /etc/db_config.ini << 'EOF'
[database]
host=127.0.0.1
port=5432
name=mydb
user=admin
password=secret

[pool]
size=10
timeout=60
EOF"
```

---

## Часть 2. Регулярные выражения

### Шаг 2.1. Playbook для изучения регулярных выражений

```bash
nano ~/lab8-text/01-regex-basics.yml
```

```yaml
---
- name: Изучение регулярных выражений
  hosts: all
  gather_facts: no
  
  tasks:
    # Поиск строк, начинающихся с #
    - name: Найти все закомментированные строки
      ansible.builtin.shell: grep '^#' /etc/test_config.txt || true
      register: comments
      changed_when: false
      
    
    - name: Вывод закомментированных строк
      ansible.builtin.debug:
        msg: "Закомментированные строки:\n{{ comments.stdout_lines }}"
     
    
    # Поиск строк, не начинающихся с #
    - name: Найти все активные настройки
      ansible.builtin.shell: grep '^[^#]' /etc/test_config.txt | grep -v '^$' || true
      register: active
      changed_when: false
     
    
    - name: Вывод активных настроек
      ansible.builtin.debug:
        msg: "Активные настройки:\n{{ active.stdout_lines }}"
     
    
    # Поиск строк с числами
    - name: Найти все строки с числами
      ansible.builtin.shell: grep -E '[0-9]+' /etc/db_config.ini || true
      register: numbers
      changed_when: false
      
    
    - name: Вывод строк с числами
      ansible.builtin.debug:
        msg: "Строки с числами:\n{{ numbers.stdout_lines }}"
     
    
    # Поиск строк с ключ=значение
    - name: Найти все параметры (ключ=значение)
      ansible.builtin.shell: grep -E '^[a-z_]+=' /etc/db_config.ini | grep -v '^#' || true
      register: params
      changed_when: false
      
    
    - name: Вывод параметров
      ansible.builtin.debug:
        msg: "Параметры (ключ=значение):\n{{ params.stdout_lines }}"
      
```

### Шаг 2.2. Выполнение

```bash
ansible-playbook 01-regex-basics.yml
```

---

## Часть 3. Управление строками текста (модуль `lineinfile`) 
### Шаг 3.1. Playbook для управления строками

```bash
nano ~/lab8-text/02-lineinfile.yml
```

```yaml
---
- name: Управление строками с помощью lineinfile
  hosts: web-server
  become: yes
  
  tasks:
    # ===== ДОБАВЛЕНИЕ СТРОК =====
    - name: Добавить DNS сервер в resolv.conf
      ansible.builtin.lineinfile:
        path: /etc/resolv.conf
        line: 'nameserver 8.8.8.8'
        state: present
        create: yes
    
    - name: Добавить строку в конец файла
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        line: '# Добавлено Ansible'
        insertafter: EOF
        state: present
    
    - name: Добавить строку в начало файла
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        line: '# === НАЧАЛО КОНФИГУРАЦИИ ==='
        insertbefore: BOF
        state: present
    
    # ===== ЗАМЕНА СТРОК =====
    - name: Изменить порт с 8080 на 9090
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^port='
        line: 'port=9090'
        backup: yes
    
    - name: Включить debug режим
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^debug_mode='
        line: 'debug_mode=true'
    
    - name: Изменить max_connections
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^max_connections='
        line: 'max_connections=200'
    
    # ===== РАСКОММЕНТИРОВАНИЕ СТРОК =====
    - name: Раскомментировать cache_size
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^#\s*cache_size='
        line: 'cache_size=512'
        backrefs: yes
    
    # ===== ВСТАВКА В ОПРЕДЕЛЁННОЕ МЕСТО =====
    - name: Добавить строку после определённой
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        line: 'new_param=value'
        insertafter: '^server_name='
        state: present
    
    - name: Добавить строку перед определённой
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        line: '# Пользовательские настройки'
        insertbefore: '^max_connections'
        state: present
    
    # ===== УДАЛЕНИЕ СТРОК =====
    - name: Удалить строки с TODO
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: 'TODO:'
        state: absent
    
    - name: Удалить пустые строки
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^$'
        state: absent
    
    # ===== ОБРАТНЫЕ ССЫЛКИ (backrefs) =====
    - name: Заменить формат параметров (key=value -> key = value)
      ansible.builtin.lineinfile:
        path: /etc/test_config.txt
        regexp: '^([a-z_]+)=([^=]+)$'
        line: '\1 = \2'
        backrefs: yes
    
    # ===== ПРОВЕРКА ИЗМЕНЕНИЙ =====
    - name: Показать итоговый файл
      ansible.builtin.shell: cat /etc/test_config.txt
      register: final_content
      changed_when: false
    
    - name: Вывод содержимого
      ansible.builtin.debug:
        msg: "{{ final_content.stdout_lines }}"
```

### Шаг 3.2. Выполнение

```bash
ansible-playbook 02-lineinfile.yml

# Просмотр изменённого файла
ssh ansible@192.168.1.11 "cat /etc/test_config.txt"
```

---

## Часть 4. Управление блоками текста (модуль `blockinfile`) 

### Шаг 4.1. Playbook для управления блоками

```bash
nano ~/lab8-text/03-blockinfile.yml
```

```yaml
---
- name: Управление блоками текста с помощью blockinfile
  hosts: web-server
  become: yes
  
  tasks:
    # ===== ДОБАВЛЕНИЕ БЛОКА =====
    - name: Добавить блок с настройками кэша
      ansible.builtin.blockinfile:
        path: /etc/nginx/nginx.conf
        block: |
          # Настройки кэширования
          proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:10m;
          proxy_cache_key "$scheme$request_method$host$request_uri";
          proxy_cache_valid 200 302 60m;
          proxy_cache_valid 404 1m;
        marker: "# {mark} ANSIBLE CACHE CONFIG"
        create: yes
        backup: yes
     
    
    # ===== ДОБАВЛЕНИЕ БЛОКА В КОНЕЦ ФАЙЛА =====
    - name: Добавить блок мониторинга
      ansible.builtin.blockinfile:
        path: /etc/nginx/nginx.conf
        block: |
          # Настройки мониторинга
          stub_status on;
          access_log /var/log/nginx/status.log;
        marker: "## {mark} MONITORING CONFIG ##"
        insertafter: EOF
    
    # ===== ДОБАВЛЕНИЕ БЛОКА В НАЧАЛО ФАЙЛА =====
    - name: Добавить блок с глобальными настройками
      ansible.builtin.blockinfile:
        path: /etc/nginx/nginx.conf
        block: |
          user www-data;
          worker_processes auto;
          pid /run/nginx.pid;
        marker: "### {mark} GLOBAL CONFIG ###"
        insertbefore: BOF
    
    # ===== ОБНОВЛЕНИЕ СУЩЕСТВУЮЩЕГО БЛОКА =====
    - name: Обновить настройки кэша
      ansible.builtin.blockinfile:
        path: /etc/nginx/nginx.conf
        block: |
          # Обновлённые настройки кэширования
          proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=mycache:20m;
          proxy_cache_key "$scheme$request_method$host$request_uri";
          proxy_cache_valid 200 302 120m;
          proxy_cache_valid 404 2m;
          proxy_cache_min_uses 3;
        marker: "# {mark} ANSIBLE CACHE CONFIG"
    
    # ===== ДОБАВЛЕНИЕ БЛОКА ПОСЛЕ СТРОКИ =====
    - name: Добавить блок безопасности после определённой строки
      ansible.builtin.blockinfile:
        path: /etc/nginx/nginx.conf
        block: |
          # Настройки безопасности
          server_tokens off;
          add_header X-Frame-Options "SAMEORIGIN";
          add_header X-Content-Type-Options nosniff;
        marker: "# {mark} SECURITY CONFIG"
        insertafter: '^http {'
    
    # ===== ПРОВЕРКА ФАЙЛА NGINX =====
    - name: Проверить конфигурацию nginx
      ansible.builtin.command: nginx -t
      register: nginx_test
      changed_when: false
      ignore_errors: yes
    
    - name: Результат проверки
      ansible.builtin.debug:
        msg: "{{ nginx_test.stdout_lines }}"
    
    # ===== ДЕМОНСТРАЦИЯ НА ТЕСТОВОМ ФАЙЛЕ =====
    - name: Создать тестовый файл
      ansible.builtin.copy:
        content: "# Пустой файл для демонстрации"
        dest: /tmp/test_blocks.txt
        mode: '0644'
    
    - name: Добавить первый блок
      ansible.builtin.blockinfile:
        path: /tmp/test_blocks.txt
        block: |
          ПЕРВЫЙ БЛОК
          Содержимое первого блока
        marker: "<!-- {mark} BLOCK1 -->"
    
    - name: Добавить второй блок
      ansible.builtin.blockinfile:
        path: /tmp/test_blocks.txt
        block: |
          ВТОРОЙ БЛОК
          Содержимое второго блока
        marker: "<!-- {mark} BLOCK2 -->"
    
    - name: Показать результат
      ansible.builtin.shell: cat /tmp/test_blocks.txt
      register: blocks_result
      changed_when: false
    
    - name: Содержимое тестового файла
      ansible.builtin.debug:
        msg: "{{ blocks_result.stdout_lines }}"
```

### Шаг 4.2. Выполнение

```bash
ansible-playbook 03-blockinfile.yml

# Просмотр изменённого конфига nginx
ssh ansible@192.168.1.11 "cat /etc/nginx/nginx.conf | head -50"
```

---

## Часть 5. Массовая замена (модуль `replace`)

### Шаг 5.1. Playbook для массовой замены текста

```bash
nano ~/lab8-text/04-replace.yml
```

```yaml
---
- name: Массовая замена текста с помощью replace
  hosts: db-server
  become: yes
  
  tasks:
    # ===== ПРОСТАЯ ЗАМЕНА =====
    - name: Заменить localhost на реальный IP
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: '127\.0\.0\.1'
        replace: '{{ ansible_default_ipv4.address }}'
        backup: yes
    
    - name: Заменить пароль на зашифрованный
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: 'password=secret'
        replace: 'password=ENC(HASHED_PASSWORD)'
    
    # ===== ЗАМЕНА С ИСПОЛЬЗОВАНИЕМ ГРУПП =====
    - name: Заменить формат (key=value -> key : value)
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: '^([a-z_]+)=([^=]+)$'
        replace: '\1: \2'
    
    # ===== ЗАМЕНА НЕСКОЛЬКИХ ВХОЖДЕНИЙ =====
    - name: Изменить все timeout значения
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: 'timeout=(\d+)'
        replace: 'timeout=\1_version2'
    
    # ===== УДАЛЕНИЕ ЛИШНИХ СТРОК =====
    - name: Удалить комментарии
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: '^#.*$'
        replace: ''
    
    - name: Нормализовать пустые строки (убрать множественные)
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: '\n\s*\n'
        replace: '\n'
    
    # ===== ОБРАТНЫЕ ССЫЛКИ =====
    - name: Преобразовать параметры в camelCase
      ansible.builtin.replace:
        path: /etc/db_config.ini
        regexp: '([a-z])_([a-z])'
        replace: '\1\U\2'
    
    # ===== ПРОВЕРКА РЕЗУЛЬТАТА =====
    - name: Показать итоговый файл
      ansible.builtin.shell: cat /etc/db_config.ini
      register: final_config
      changed_when: false
    
    - name: Вывод результата
      ansible.builtin.debug:
        msg: "{{ final_config.stdout_lines }}"
```

### Шаг 5.2. Выполнение

```bash
ansible-playbook 04-replace.yml

# Просмотр изменённого файла
ssh ansible@192.168.1.12 "cat /etc/db_config.ini"
```

