# Лабораторная работа №9: Циклы, блоки и условные конструкции

## Цель работы
Научиться использовать условные конструкции (`when`), циклы (`loop`) и блоки (`block`/`rescue`/`always`) для создания гибких и отказоустойчивых плейбуков Ansible.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля).

---

## Часть 1. Подготовка рабочей среды (5 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab9-loops-conditions
cd ~/lab9-loops-conditions
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab9-loops-conditions/inventory.ini
```

```ini
[webservers]
web-server ansible_host=ipserver1

[databases]
db-server ansible_host=ipserver2

[all:children]
webservers
databases

[all:vars]
ansible_user=ansible
ansible_python_interpreter=/usr/bin/python3
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab9-loops-conditions/ansible.cfg
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
 (20 минут)

### Шаг 2.1. Playbook с условными конструкциями

```bash
nano ~/lab9-loops-conditions/01-conditions.yml
```

```yaml
---
- name: Условные конструкции в Ansible
  hosts: all
  gather_facts: yes
  
  vars:
    environment: "production"
    enable_monitoring: true
    install_debug_tools: false
    app_port: 8080
  
  tasks:
    # ===== ПРОСТЫЕ УСЛОВИЯ =====
    - name: Проверка семейства ОС
      ansible.builtin.debug:
        msg: "Это система семейства {{ ansible_os_family }}"
    
    - name: Специальное сообщение для Debian
      ansible.builtin.debug:
        msg: "Отлично! Debian - надёжная система"
      when: ansible_os_family == "Debian"
    
    # ===== ПРОВЕРКА РЕСУРСОВ =====
    - name: Проверка памяти (предупреждение)
      ansible.builtin.debug:
        msg: "⚠️ ВНИМАНИЕ! Мало памяти: {{ ansible_memtotal_mb }} MB (рекомендуется 2048 MB)"
      when: ansible_memtotal_mb < 2048
    
    - name: Проверка количества ядер
      ansible.builtin.debug:
        msg: "✅ Хорошая конфигурация: {{ ansible_processor_vcpus }} ядер"
      when: ansible_processor_vcpus >= 2
    
    # ===== ПРОВЕРКА ПЕРЕМЕННЫХ =====
    - name: Установка мониторинга
      ansible.builtin.debug:
        msg: "🟢 Устанавливаю систему мониторинга"
      when: enable_monitoring | bool
    
    - name: Установка отладочных утилит
      ansible.builtin.debug:
        msg: "🐛 Устанавливаю отладочные утилиты"
      when: install_debug_tools | bool
    
    - name: Настройка порта для production
      ansible.builtin.debug:
        msg: "Производственное окружение, порт: {{ app_port }}"
      when: environment == "production"
    
    - name: Настройка для разработки
      ansible.builtin.debug:
        msg: "Режим разработки, включено подробное логирование"
      when: environment == "development"
    
    # ===== КОМБИНИРОВАННЫЕ УСЛОВИЯ (AND) =====
    - name: Установка кэширования (Debian И память > 1GB)
      ansible.builtin.debug:
        msg: "Устанавливаю кэширование (большая память)"
      when: 
        - ansible_os_family == "Debian"
        - ansible_memtotal_mb > 1024
    
    # ===== ЛОГИЧЕСКОЕ ИЛИ (OR) =====
    - name: Универсальная установка (Debian или RedHat)
      ansible.builtin.debug:
        msg: "Система совместима с нашим ПО"
      when: 
        - ansible_os_family == "Debian" or ansible_os_family == "RedHat"
    
    # ===== ОТРИЦАНИЕ (NOT) =====
    - name: Не production окружение
      ansible.builtin.debug:
        msg: "Включено подробное логирование (не production)"
      when: environment != "production"
    
    - name: Реальный запуск (не check mode)
      ansible.builtin.debug:
        msg: "Выполняется реальный запуск (не симуляция)"
      when: not ansible_check_mode
    
    # ===== ПРОВЕРКА ГРУПП =====
    - name: Действия только на веб-сервере
      ansible.builtin.debug:
        msg: "Выполняю настройку веб-сервера {{ inventory_hostname }}"
      when: "'webservers' in group_names"
    
    - name: Действия только на БД-сервере
      ansible.builtin.debug:
        msg: "Выполняю настройку базы данных на {{ inventory_hostname }}"
      when: "'databases' in group_names"
    
    # ===== ПРОВЕРКА СУЩЕСТВОВАНИЯ ФАЙЛОВ =====
    - name: Проверка существования файла
      ansible.builtin.stat:
        path: /etc/nginx/nginx.conf
      register: nginx_config
    
    - name: Сообщение о наличии nginx
      ansible.builtin.debug:
        msg: "Nginx установлен и настроен"
      when: nginx_config.stat.exists
    
    - name: Сообщение о отсутствии nginx
      ansible.builtin.debug:
        msg: "Nginx не установлен"
      when: not nginx_config.stat.exists
    
    # ===== ПРОВЕРКА ВЕРСИЙ =====
    - name: Использование нового API для свежих версий
      ansible.builtin.debug:
        msg: "Использую новое API (версия >= 20.04)"
      when: ansible_distribution_version is version('20.04', '>=')
```

### Шаг 2.2. Выполнение

```bash
# Обычный запуск
ansible-playbook 01-conditions.yml

# Запуск в check mode для демонстрации not ansible_check_mode
ansible-playbook 01-conditions.yml --check

# Запуск с переопределением переменных
ansible-playbook 01-conditions.yml -e "environment=development install_debug_tools=true"
```

---

## Часть 3. Циклы (Loops) (25 минут)

### Шаг 3.1. Playbook с циклами

```bash
nano ~/lab9-loops-conditions/02-loops.yml
```

```yaml
---
- name: Циклы в Ansible
  hosts: all
  become: yes
  
  vars:
    packages_to_install:
      - curl
      - wget
      - git
      - vim
      - htop
      - net-tools
    
    users_to_create:
      - name: developer
        uid: 2001
        groups: sudo
      - name: app_user
        uid: 2002
        groups: www-data
      - name: backup_user
        uid: 2003
        groups: backup
    
    directories:
      - /opt/app/logs
      - /opt/app/config
      - /opt/app/data
      - /opt/app/backups
    
    app_configs:
      - src: nginx.conf
        dest: /etc/nginx/nginx.conf
        mode: '0644'
      - src: php.ini
        dest: /etc/php/8.2/cli/php.ini
        mode: '0644'
      - src: app.env
        dest: /opt/app/.env
        mode: '0600'
  
  tasks:
    # ===== ПРОСТОЙ ЦИКЛ ПО СПИСКУ =====
    - name: Установка пакетов
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop: "{{ packages_to_install }}"
      loop_control:
        label: "{{ item }}"
    
    # ===== СОЗДАНИЕ ДИРЕКТОРИЙ ЧЕРЕЗ ЦИКЛ =====
    - name: Создание структуры директорий
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
      loop: "{{ directories }}"
      loop_control:
        label: "Создаю {{ item }}"
    
    # ===== ЦИКЛ ПО СЛОВАРЮ =====
    - name: Создание пользователей с разными параметрами
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: "{{ item.groups }}"
        shell: /bin/bash
        state: present
      loop: "{{ users_to_create }}"
      loop_control:
        label: "{{ item.name }} (UID: {{ item.uid }})"
    
    # ===== ЦИКЛ С ИНДЕКСАМИ =====
    - name: Вывод списка с индексами
      ansible.builtin.debug:
        msg: "Сервер {{ idx }}: {{ item }}"
      loop: "{{ groups['all'] }}"
      loop_control:
        index_var: idx
    
    # ===== ЦИКЛ ДЛЯ КОПИРОВАНИЯ ФАЙЛОВ =====
    - name: Копирование конфигурационных файлов
      ansible.builtin.copy:
        src: "files/{{ item.src }}"
        dest: "{{ item.dest }}"
        mode: "{{ item.mode }}"
      loop: "{{ app_configs }}"
      loop_control:
        label: "{{ item.dest }}"
    
    # ===== ЦИКЛ С УСЛОВИЕМ ВНУТРИ =====
    - name: Условная установка пакетов (только для Debian)
      ansible.builtin.apt:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - apache2
        - postgresql
      when: ansible_os_family == "Debian"
      loop_control:
        label: "{{ item }} (Debian)"
    
    # ===== ВЛОЖЕННЫЕ ЦИКЛЫ (CROSS PRODUCT) =====
    - name: Создание комбинаций окружений и сервисов
      ansible.builtin.debug:
        msg: "Конфигурация для {{ env }}/{{ service }}.yml"
      loop: "{{ ['dev', 'staging', 'prod'] | product(['web', 'api', 'db']) | list }}"
      vars:
        env: "{{ item[0] }}"
        service: "{{ item[1] }}"
      loop_control:
        label: "{{ env }}-{{ service }}"
    
    # ===== ДИАПАЗОНЫ ЧИСЕЛ =====
    - name: Создание тестовых файлов
      ansible.builtin.file:
        path: "/tmp/test_file_{{ item }}.txt"
        state: touch
      loop: "{{ range(1, 6) | list }}"
      loop_control:
        label: "Файл {{ item }}"
    
    # ===== ИСПОЛЬЗОВАНИЕ FILTER ДЛЯ ФИЛЬТРАЦИИ СПИСКА =====
    - name: Установка только веб-пакетов
      ansible.builtin.debug:
        msg: "Устанавливаю веб-пакет: {{ item }}"
      loop: "{{ ['nginx', 'apache2', 'mysql', 'postgresql'] | difference(['mysql', 'postgresql']) }}"
    
    # ===== ПАРАЛЛЕЛЬНЫЕ ЦИКЛЫ =====
    - name: Параллельная установка пакетов
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
      loop:
        - curl
        - wget
        - git
      loop_control:
        parallel: 3
```

### Шаг 3.2. Создание тестовых файлов для копирования

```bash
mkdir -p ~/lab9-loops-conditions/files

echo "server { listen 80; }" > ~/lab9-loops-conditions/files/nginx.conf
echo "memory_limit = 256M" > ~/lab9-loops-conditions/files/php.ini
echo "APP_ENV=production" > ~/lab9-loops-conditions/files/app.env
```

### Шаг 3.3. Выполнение

```bash
ansible-playbook 02-loops.yml
```

---

## Часть 4. Блоки (Blocks) с обработкой ошибок (25 минут)

### Шаг 4.1. Playbook с блоками

```bash
nano ~/lab9-loops-conditions/03-blocks.yml
```

```yaml
---
- name: Блоки и обработка ошибок в Ansible
  hosts: web-server
  become: yes
  
  tasks:
    # ===== ПРОСТОЙ БЛОК С ОБЩИМ УСЛОВИЕМ =====
    - name: Настройка Debian системы
      block:
        - name: Обновление кэша
          ansible.builtin.apt:
            update_cache: yes
        
        - name: Установка базовых пакетов
          ansible.builtin.apt:
            name:
              - curl
              - wget
              - git
            state: present
      
      when: ansible_os_family == "Debian"
    
    # ===== БЛОК С RESCUE (ОБРАБОТКА ОШИБОК) =====
    - name: Развёртывание приложения с откатом
      block:
        - name: Остановка сервиса
          ansible.builtin.service:
            name: nginx
            state: stopped
        
        - name: Копирование новой версии (симулируем ошибку)
          ansible.builtin.command: /bin/false
          register: deploy_result
          # Этот шаг вызовет ошибку
        
        - name: Запуск сервиса
          ansible.builtin.service:
            name: nginx
            state: started
      
      rescue:
        - name: Откат изменений
          ansible.builtin.debug:
            msg: "Ошибка! Выполняю откат изменений..."
        
        - name: Запуск старой версии сервиса
          ansible.builtin.service:
            name: nginx
            state: started
        
        - name: Отправка уведомления об ошибке
          ansible.builtin.debug:
            msg: "Отправляю алерт: Деплой не удался на {{ inventory_hostname }}"
      
      always:
        - name: Очистка временных файлов (выполнится всегда)
          ansible.builtin.debug:
            msg: "Очищаю временные файлы..."
    
    # ===== БЛОК С ИГНОРИРОВАНИЕМ ОШИБОК =====
    - name: Блок с ignore_errors
      block:
        - name: Рискованная операция 1
          ansible.builtin.command: /bin/false
          ignore_errors: yes
        
        - name: Рискованная операция 2
          ansible.builtin.command: /bin/true
      
      rescue:
        - name: Этот блок не выполнится из-за ignore_errors
          ansible.builtin.debug:
            msg: "Это сообщение не появится"
    
    # ===== ВЛОЖЕННЫЕ БЛОКИ =====
    - name: Внешний блок
      block:
        - name: Внешняя задача 1
          ansible.builtin.debug:
            msg: "Внешний блок - задача 1"
        
        - name: Внутренний блок
          block:
            - name: Внутренняя задача 1
              ansible.builtin.debug:
                msg: "Внутренний блок - задача 1"
            
            - name: Внутренняя задача 2
              ansible.builtin.debug:
                msg: "Внутренний блок - задача 2"
          
          when: ansible_os_family == "Debian"
        
        - name: Внешняя задача 2
          ansible.builtin.debug:
            msg: "Внешний блок - задача 2"
      
      when: inventory_hostname != "localhost"
    
    # ===== РЕАЛЬНЫЙ ПРИМЕР: НАСТРОЙКА С ОТКАТОМ =====
    - name: Настройка веб-сервера с откатом при ошибке
      block:
        - name: Сохранение текущей конфигурации
          ansible.builtin.copy:
            src: /etc/nginx/nginx.conf
            dest: /backup/nginx.conf.bak
            remote_src: yes
        
        - name: Копирование новой конфигурации
          ansible.builtin.copy:
            content: |
              user www-data;
              worker_processes auto;
              # Неправильная директива (вызовет ошибку)
              invalid_directive test;
            dest: /etc/nginx/nginx.conf
            mode: '0644'
      
      rescue:
        - name: Восстановление конфигурации из бэкапа
          ansible.builtin.copy:
            src: /backup/nginx.conf.bak
            dest: /etc/nginx/nginx.conf
            remote_src: yes
        
        - name: Отправка уведомления
          ansible.builtin.debug:
            msg: "Ошибка настройки nginx! Конфигурация восстановлена."
      
      always:
        - name: Проверка конфигурации nginx
          ansible.builtin.command: nginx -t
          register: nginx_test
          ignore_errors: yes
          changed_when: false
        
        - name: Результат проверки
          ansible.builtin.debug:
            msg: "Результат проверки nginx: {{ nginx_test.rc == 0 and 'OK' or 'ОШИБКА' }}"
```

### Шаг 4.2. Выполнение

```bash
ansible-playbook 03-blocks.yml
```

---

