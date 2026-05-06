# Лабораторная работа №7: Использование фактов и волшебных переменных

## Цель работы
Изучить системные факты и волшебные переменные Ansible, научиться собирать информацию о хостах, использовать её для принятия решений и создавать настраиваемые факты.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab7-facts/{templates,facts}
cd ~/lab7-facts
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab7-facts/inventory.ini
```

```ini
[webservers]
web-server ansible_host=ip server1

[databases]
db-server ansible_host=ip serfver2
[all:children]
webservers
databases

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab7-facts/ansible.cfg
```

```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
remote_user = ansible
become = yes
become_method = sudo
become_user = root
gathering = smart

[ssh_connection]
pipelining = True
```

---

## Часть 2. Знакомство с фактами (20 минут)

### Шаг 2.1. Просмотр всех фактов

```bash
# Просмотр всех фактов одного хоста
ansible web-server -m setup

# Сохранение фактов в файл для анализа
ansible web-server -m setup > ~/lab7-facts/all_facts.txt

# Просмотр с фильтром (только сетевые факты)
ansible web-server -m setup -a "filter=*ipv4*"

# Просмотр фактов о памяти
ansible web-server -m setup -a "filter=*mem*"
```

### Шаг 2.2. Создание playbook для изучения фактов

```bash
nano ~/lab7-facts/01-discover-facts.yml
```

```yaml
---
- name: Знакомство с системными фактами
  hosts: all
  gather_facts: yes
  
  tasks:
    # ОС и ядро
    - name: Информация об операционной системе
      ansible.builtin.debug:
        msg: |
          === ОПЕРАЦИОННАЯ СИСТЕМА ===
          Хост: {{ ansible_hostname }}
          Семейство ОС: {{ ansible_os_family }}
          Дистрибутив: {{ ansible_distribution }}
          Версия: {{ ansible_distribution_version }}
          Релиз: {{ ansible_distribution_release }}
          Ядро: {{ ansible_kernel }}
          Архитектура: {{ ansible_architecture }}
    
    # Процессор
    - name: Информация о процессоре
      ansible.builtin.debug:
        msg: |
          === ПРОЦЕССОР ===
          vCPU: {{ ansible_processor_vcpus }}
          Ядер на CPU: {{ ansible_processor_cores }}
          Потоков на ядро: {{ ansible_processor_threads_per_core }}
          Модель: {{ ansible_processor[1] | default('неизвестно') }}
    
    # Память
    - name: Информация о памяти
      ansible.builtin.debug:
        msg: |
          === ПАМЯТЬ ===
          RAM всего: {{ ansible_memtotal_mb }} MB
          RAM свободно: {{ ansible_memfree_mb }} MB
          Swap всего: {{ ansible_swaptotal_mb }} MB
          Swap свободно: {{ ansible_swapfree_mb }} MB
    
    # Сеть
    - name: Информация о сети
      ansible.builtin.debug:
        msg: |
          === СЕТЬ ===
          Основной IP: {{ ansible_default_ipv4.address }}
          Интерфейс: {{ ansible_default_ipv4.interface }}
          Шлюз: {{ ansible_default_ipv4.gateway }}
          Все IP: {{ ansible_all_ipv4_addresses }}
          DNS: {{ ansible_dns.nameservers | default(['нет']) }}
    
    # Диски
    - name: Информация о дисках
      ansible.builtin.debug:
        msg: |
          === ДИСКИ ===
          {% for mount in ansible_mounts %}
          {{ mount.mount }}: {{ mount.size_total | filesizeformat }} 
          (свободно: {{ mount.size_available | filesizeformat }}, 
           используется: {{ (mount.size_total - mount.size_available) | filesizeformat }})
          {% endfor %}
```

### Шаг 2.3. Выполнение playbook

```bash
ansible-playbook 01-discover-facts.yml
```

---

## Часть 3. Использование фактов для принятия решений (20 минут)

### Шаг 3.1. Playbook с кроссплатформенной логикой

```bash
nano ~/lab7-facts/02-conditional-facts.yml
```

```yaml
---
- name: Использование фактов для принятия решений
  hosts: all
  gather_facts: yes
  
  tasks:
    # Кроссплатформенная установка
    - name: Установка веб-сервера для Debian
      ansible.builtin.debug:
        msg: "Debian/Ubuntu: устанавливаю nginx через apt"
      when: ansible_os_family == "Debian"
    
    - name: Установка веб-сервера для RedHat
      ansible.builtin.debug:
        msg: "RedHat/CentOS: устанавливаю httpd через yum/dnf"
      when: ansible_os_family == "RedHat"
    
    # Проверка наличия Docker
    - name: Проверка наличия Docker
      ansible.builtin.command: which docker
      register: docker_check
      failed_when: false
      changed_when: false
    
    - name: Установка Docker, если отсутствует
      ansible.builtin.debug:
        msg: "Docker не установлен, требуется установка"
      when: docker_check.rc != 0
    
    # Проверка ресурсов
    - name: Проверка минимальных требований к RAM
      ansible.builtin.debug:
        msg: "ВНИМАНИЕ: RAM меньше 2GB ({{ ansible_memtotal_mb }} MB)"
      when: ansible_memtotal_mb < 2048
    
    - name: Информация о загрузке CPU
      ansible.builtin.shell: "uptime | awk -F 'load average:' '{print $2}'"
      register: load_avg
      changed_when: false
      
    - name: Вывод нагрузки
      ansible.builtin.debug:
        msg: "Средняя нагрузка за 1/5/15 минут:{{ load_avg.stdout }}"
    
    # Специфичные действия для разных серверов
    - name: Действия только на веб-сервере
      ansible.builtin.debug:
        msg: "Специальная настройка для веб-сервера {{ inventory_hostname }}"
      when: "'webservers' in group_names"
    
    - name: Действия только на сервере БД
      ansible.builtin.debug:
        msg: "Специальная настройка для сервера БД {{ inventory_hostname }}"
      when: "'databases' in group_names"
```

### Шаг 3.2. Выполнение

```bash
ansible-playbook 02-conditional-facts.yml
```

---

## Часть 4. Создание отчёта о системе (20 минут)

### Шаг 4.1. Playbook для генерации системного отчёта

```bash
nano ~/lab7-facts/03-system-report.yml
```

```yaml
---
- name: Генерация отчёта о системе
  hosts: all
  gather_facts: yes
  
  tasks:
    - name: Создание директории для отчётов
      ansible.builtin.file:
        path: /opt/system_reports
        state: directory
        mode: '0755'
    
    - name: Генерация отчёта о системе
      ansible.builtin.copy:
        content: |
          ===============================================
          СИСТЕМНЫЙ ОТЧЁТ
          ===============================================
          Дата генерации: {{ ansible_date_time.date }} {{ ansible_date_time.time }}
          Хост: {{ ansible_hostname }} ({{ ansible_fqdn | default(ansible_hostname) }})
          
          --- ОПЕРАЦИОННАЯ СИСТЕМА ---
          Семейство: {{ ansible_os_family }}
          Дистрибутив: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Кодовое имя: {{ ansible_distribution_release }}
          Ядро: {{ ansible_kernel }}
          Архитектура: {{ ansible_architecture }}
          
          --- ПРОЦЕССОР ---
          vCPU: {{ ansible_processor_vcpus }}
          Ядер: {{ ansible_processor_cores }}
          Модель: {{ ansible_processor[2] | default(ansible_processor[1]) | default('неизвестно') }}
          
          --- ПАМЯТЬ ---
          RAM всего: {{ ansible_memtotal_mb }} MB
          RAM свободно: {{ ansible_memfree_mb }} MB
          Swap всего: {{ ansible_swaptotal_mb }} MB
          Swap свободно: {{ ansible_swapfree_mb }} MB
          
          --- СЕТЬ ---
          Основной IP: {{ ansible_default_ipv4.address }}
          Интерфейс: {{ ansible_default_ipv4.interface }}
          MAC адрес: {{ ansible_default_ipv4.macaddress | default('неизвестно') }}
          Шлюз: {{ ansible_default_ipv4.gateway }}
          
          --- ДИСКИ ---
          {% for mount in ansible_mounts %}
          {{ mount.mount }}:
            - Размер: {{ mount.size_total | filesizeformat }}
            - Свободно: {{ mount.size_available | filesizeformat }}
            - Использовано: {{ (mount.size_total - mount.size_available) | filesizeformat }}
            - Тип: {{ mount.fstype }}
          {% endfor %}
          
          --- ПОЛЬЗОВАТЕЛИ ---
          Всего пользователей: {{ ansible_user_list | length }}
          Пользователи с оболочкой:
          {% for user in ansible_user_list if user.shell not in ['/sbin/nologin', '/bin/false', '/usr/sbin/nologin'] %}
          - {{ user.name }} ({{ user.shell }})
          {% endfor %}
          
          ===============================================
          Отчёт сгенерирован Ansible {{ ansible_version.full }}
          ===============================================
        dest: "/opt/system_reports/{{ inventory_hostname }}_{{ ansible_date_time.date }}.txt"
        mode: '0644'
    
    - name: Вывод пути к отчёту
      ansible.builtin.debug:
        msg: "Отчёт сохранён: /opt/system_reports/{{ inventory_hostname }}_{{ ansible_date_time.date }}.txt"
    
    - name: Чтение отчёта
      ansible.builtin.slurp:
        src: "/opt/system_reports/{{ inventory_hostname }}_{{ ansible_date_time.date }}.txt"
      register: report_content
    
    - name: Показать отчёт
      ansible.builtin.debug:
        msg: "{{ report_content.content | b64decode }}"
```

### Шаг 4.2. Выполнение

```bash
ansible-playbook 03-system-report.yml

# Просмотр созданных отчётов
ssh ansible@192.168.1.11 "cat /opt/system_reports/web-server_*.txt | head -50"
```

---

## Часть 5. Волшебные переменные (20 минут)

### Шаг 5.1. Playbook с волшебными переменными

```bash
nano ~/lab7-facts/04-magic-variables.yml
```

```yaml
---
- name: Изучение волшебных переменных
  hosts: all
  gather_facts: yes
  
  tasks:
    # Переменные инвентаря
    - name: Информация об инвентаре
      ansible.builtin.debug:
        msg: |
          === ИНВЕНТАРЬ ===
          inventory_hostname (из инвентаря): {{ inventory_hostname }}
          inventory_hostname_short: {{ inventory_hostname_short }}
          inventory_dir: {{ inventory_dir }}
          playbook_dir: {{ playbook_dir }}
          groups['all']: {{ groups['all'] }}
          groups['webservers']: {{ groups['webservers'] | default('нет') }}
          group_names (для этого хоста): {{ group_names }}
    
    # Переменные хостов
    - name: Доступ к переменным других хостов через hostvars
      ansible.builtin.debug:
        msg: |
          === ДРУГИЕ ХОСТЫ ===
          {% for host in groups['all'] if host != inventory_hostname %}
          Хост {{ host }}:
            - IP: {{ hostvars[host]['ansible_default_ipv4']['address'] }}
            - ОС: {{ hostvars[host]['ansible_distribution'] }}
          {% endfor %}
    
    # Переменные выполнения
    - name: Параметры выполнения Ansible
      ansible.builtin.debug:
        msg: |
          === ПАРАМЕТРЫ ЗАПУСКА ===
          Версия Ansible: {{ ansible_version.full }}
          Check mode: {{ ansible_check_mode }}
          Diff mode: {{ ansible_diff_mode }}
          Verbosity: {{ ansible_verbosity }}
          Forks: {{ ansible_forks }}
    
    # Сравнение inventory_hostname и ansible_hostname
    - name: Сравнение имён хостов
      ansible.builtin.debug:
        msg: |
          === ИМЕНА ХОСТОВ ===
          inventory_hostname (из инвентаря): {{ inventory_hostname }}
          ansible_hostname (системное): {{ ansible_hostname }}
          Совпадают: {{ inventory_hostname == ansible_hostname }}
```

### Шаг 5.2. Выполнение

```bash
ansible-playbook 04-magic-variables.yml
```

### Шаг 5.3. Запуск с разными параметрами для демонстрации

```bash
# Запуск в check mode
ansible-playbook 04-magic-variables.yml --check

# Запуск с увеличенной детализацией
ansible-playbook 04-magic-variables.yml -v
```

---

## Часть 6. Настраиваемые факты (Custom Facts) (20 минут)

### Шаг 6.1. Создание настраиваемых фактов

**На веб-сервере:**

```bash
ssh ansible@192.168.1.11

# Создание директории для фактов
sudo mkdir -p /etc/ansible/facts.d

# Создание факта в формате INI
sudo tee /etc/ansible/facts.d/app.fact << 'EOF'
[application]
name=web_portal
version=2.1.0
environment=production

[deployment]
date=2024-01-15
user=deploy
EOF

# Создание факта в формате JSON
sudo tee /etc/ansible/facts.d/custom.fact << 'EOF'
{
    "web": {
        "port": 80,
        "ssl_port": 443,
        "document_root": "/var/www/html"
    },
    "features": {
        "cache": true,
        "debug": false,
        "monitoring": true
    }
}
EOF

exit
```

**На сервере БД:**

```bash
ssh ansible@192.168.1.12

sudo mkdir -p /etc/ansible/facts.d

sudo tee /etc/ansible/facts.d/database.fact << 'EOF'
[database]
type=postgresql
version=15
port=5432
max_connections=200

[replication]
role=master
standby_ip=192.168.1.13
EOF

exit
```

### Шаг 6.2. Playbook для чтения настраиваемых фактов

```bash
nano ~/lab7-facts/05-custom-facts.yml
```

```yaml
---
- name: Использование настраиваемых фактов
  hosts: all
  gather_facts: yes     # Важно: custom facts собираются автоматически
  
  tasks:
    - name: Просмотр всех настраиваемых фактов
      ansible.builtin.debug:
        msg: "ansible_local: {{ ansible_local }}"
    
    - name: Факты приложения на веб-сервере
      ansible.builtin.debug:
        msg: |
          === ФАКТЫ ПРИЛОЖЕНИЯ ===
          Имя: {{ ansible_local.app.application.name | default('не определено') }}
          Версия: {{ ansible_local.app.application.version | default('N/A') }}
          Окружение: {{ ansible_local.app.application.environment | default('dev') }}
      when: ansible_local is defined and 'app' in ansible_local
    
    - name: Веб-конфигурация
      ansible.builtin.debug:
        msg: |
          === ВЕБ КОНФИГУРАЦИЯ ===
          Порт: {{ ansible_local.custom.web.port | default(80) }}
          SSL порт: {{ ansible_local.custom.web.ssl_port | default(443) }}
          Document root: {{ ansible_local.custom.web.document_root | default('/var/www') }}
          Кэширование: {{ ansible_local.custom.features.cache | default(false) }}
      when: ansible_local is defined and 'custom' in ansible_local
    
    - name: Факты базы данных
      ansible.builtin.debug:
        msg: |
          === БАЗА ДАННЫХ ===
          Тип: {{ ansible_local.database.database.type | default('sqlite') }}
          Версия: {{ ansible_local.database.database.version | default('N/A') }}
          Порт: {{ ansible_local.database.database.port | default(5432) }}
          Max connections: {{ ansible_local.database.database.max_connections | default(100) }}
          Роль репликации: {{ ansible_local.database.replication.role | default('standalone') }}
      when: ansible_local is defined and 'database' in ansible_local
```

### Шаг 6.3. Выполнение

```bash
ansible-playbook 05-custom-facts.yml
```

---
