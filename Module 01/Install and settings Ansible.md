# Лабораторная работа: Настройка и проверка окружения Ansible

## Цель работы
Научиться устанавливать Ansible, настраивать управляющий узел и управляемые хосты, проверять работоспособность окружения.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста (Managed Nodes)** — Linux Debian 12

---

## 📋 Предварительные требования

### На всех трёх машинах:
- Установлен Debian 12
- Настроены сетевые интерфейсы (машины видят друг друга)
- Есть доступ в интернет для установки пакетов

### Схема сети (пример):
Включить все машины. 
Узанть их ip адреса и записать себе в блокнот. IP адресация динамическая, срок аренды 1 неделя.
```
Control Node: (hostname: ansible-controller)
Managed Node1: (hostname: server1)
Managed Node2: (hostname: server2)
```

---

## Часть 1. Подготовка окружения 

### Шаг 1.1. Создание пользователя для Ansible

**Выполнить на ВСЕХ трёх машинах (control + 2 managed):**

```bash
# Переключиться на root
sudo su -

# Создать пользователя ansible
useradd -m -s /bin/bash ansible

# Установить пароль (на время лабораторной)
passwd ansible
# Введите пароль: 1 (парольной политики нет)

# Добавить пользователя в группу sudo
usermod -aG sudo ansible

# Настроить sudo без пароля (для автоматизации)
echo "ansible ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

### Шаг 1.2. Проверка создания пользователя

```bash
# Проверить, что пользователь создан
id ansible

# Переключиться на пользователя ansible
su - ansible

# Проверить sudo
sudo whoami
# Должно вывести: root
```

---

## Часть 2. Установка Ansible

### Шаг 2.1. Установка на Control Node

**Выполнить ТОЛЬКО на управляющем узле :**

```bash
# Обновление списка пакетов
sudo apt update

# Установка Ansible
sudo apt install -y ansible

# Проверка установки
ansible --version
```

**Ожидаемый вывод:**
```
ansible [core 2.x.x]
  config file = /etc/ansible/ansible.cfg
  ...
```

---

## Часть 3. Настройка SSH-доступа 

### Шаг 3.1. Генерация SSH-ключа на Control Node

**Выполнить на Control Node от пользователя ansible:**

```bash
# Переключиться на пользователя ansible
su - ansible

# Генерация SSH-ключа (без пароля для автоматизации)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Проверка создания ключей
ls -la ~/.ssh/
# Должны быть: id_rsa (приватный) и id_rsa.pub (публичный)
```

### Шаг 3.2. Копирование публичного ключа на Managed Nodes

**Выполнить на Control Node от пользователя ansible:**

```bash
# Копирование ключа на Managed Node 1 (ip server1)
ssh-copy-id ansible@ip server1
# При запросе пароля: 1

# Копирование ключа на Managed Node 2 (ip server2)
ssh-copy-id ansible@ip server2
# При запросе пароля: 1
```

### Шаг 3.3. Проверка SSH-подключения

**Выполнить на Control Node от пользователя ansible:**

```bash
# Проверка подключения к Node1
ssh ansible@ip server1 hostname
# Должно вывести: server1

# Проверка подключения к Node2
ssh ansible@ip server2 hostname
# Должно вывести: server2
```

---

## Часть 4. Создание инвентаря 

### Шаг 4.1. Создание директории для лабораторной

**Выполнить на Control Node от пользователя ansible:**

```bash
# Создание рабочей директории
mkdir ~/ansible-lab
cd ~/ansible-lab
```

### Шаг 4.2. Создание файла инвентаря

**Создайте файл `inventory.ini`:**

```bash
nano ~/ansible-lab/inventory.ini
```

**Содержимое файла:**

```ini
# Лабораторная работа: Настройка и проверка окружения

# Группа веб-серверов
[webservers]
server1 ansible_host=ip server1 ansible_user=ansible

# Группа баз данных
[databases]
server2 ansible_host=ip server2 ansible_user=ansible

# Группа всех серверов
[all:children]
webservers
databases

# Общие переменные для всех хостов
[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### Шаг 4.3. Проверка инвентаря

```bash
# Просмотр всех хостов в инвентаре
ansible all -i inventory.ini --list-hosts

# Просмотр хостов группы webservers
ansible webservers -i inventory.ini --list-hosts

# Просмотр хостов группы databases
ansible databases -i inventory.ini --list-hosts
```

**Ожидаемый вывод:**
```
  hosts (2):
    server1
    server2
```

---

## Часть 5. Создание конфигурационного файла Ansible

### Шаг 5.1. Создание ansible.cfg

**Создайте файл `ansible.cfg` в рабочей директории:**

```bash
nano ~/ansible-lab/ansible.cfg
```

**Содержимое файла:**

```ini
[defaults]
# Путь к файлу инвентаря
inventory = ./inventory.ini

# Отключить проверку SSH ключей (для лабораторной)
host_key_checking = False

# Пользователь по умолчанию для подключения
remote_user = ansible

# Таймаут соединения (секунды)
timeout = 10

# Количество параллельных процессов
forks = 5

# Включить цветной вывод
force_color = 1

# Путь к директории с ролями
roles_path = ./roles

# Отключить сбор фактов, если не нужны
gathering = smart

[ssh_connection]
# Ускорение SSH-соединений
pipelining = True
```

### Шаг 5.2. Проверка конфигурации

```bash
# Проверка, какой конфиг используется
ansible --version | grep "config file"

# Должно показать: config file = /home/ansible/ansible-lab/ansible.cfg
```

---

## Часть 6. Проверка окружения 

### Шаг 6.1. Базовая проверка ping

```bash
cd ~/ansible-lab

# Проверка доступности всех хостов
ansible all -m ping
```

**Ожидаемый вывод:**
```
server1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
server2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

### Шаг 6.2. Сбор информации о системе (facts)

```bash
# Сбор фактов со всех хостов
ansible all -m setup

# Сбор только операционной системы
ansible all -m setup -a "filter=ansible_distribution*"

# Сбор информации о сети
ansible all -m setup -a "filter=ansible_*_address"
```

### Шаг 6.3. Выполнение ad-hoc команд

```bash
# Проверка uptime серверов
ansible all -m shell -a "uptime"

# Проверка версии ядра
ansible all -m shell -a "uname -a"

# Проверка свободного места на диске
ansible all -m shell -a "df -h /"

# Проверка использования памяти
ansible all -m shell -a "free -h"

# Проверка имени хоста
ansible all -m shell -a "hostname"
```

### Шаг 6.4. Проверка работы с повышенными привилегиями (sudo)

```bash
# Проверка, что пользователь ansible может выполнять sudo
ansible all -b -m shell -a "whoami"
# Должно вывести: root

# Просмотр системных логов (требует sudo)
ansible all -b -m shell -a "tail -n 5 /var/log/syslog"
```

---

## Часть 7. Первый плейбук 

### Шаг 7.1. Создание простого плейбука

**Создайте файл `test-playbook.yml`:**

```bash
nano ~/ansible-lab/test-playbook.yml
```

**Содержимое плейбука:**

```yaml
---
- name: Лабораторная работа - проверка окружения
  hosts: all
  become: no  # без sudo для начала
  
  tasks:
    - name: 1. Вывод приветствия
      debug:
        msg: "Привет с сервера {{ ansible_hostname }}!"
    
    - name: 2. Проверка доступности Python
      command: python3 --version
      register: python_version
    
    - name: 3. Вывод версии Python
      debug:
        msg: "Python version: {{ python_version.stdout }}"
    
    - name: 4. Проверка дискового пространства
      shell: df -h / | tail -n 1
      register: disk_usage
    
    - name: 5. Вывод информации о диске
      debug:
        msg: "Disk usage: {{ disk_usage.stdout }}"
    
    - name: 6. Создание временного файла
      copy:
        content: |
          Лабораторная работа Ansible
          Сервер: {{ ansible_hostname }}
          Дата: {{ ansible_date_time.date }}
        dest: /tmp/ansible_test.txt
        mode: '0644'
      become: yes
    
    - name: 7. Проверка создания файла
      stat:
        path: /tmp/ansible_test.txt
      register: file_info
    
    - name: 8. Вывод информации о файле
      debug:
        msg: "Файл создан, размер: {{ file_info.stat.size }} байт"
    
    - name: 9. Завершающее сообщение
      debug:
        msg: "Проверка окружения на {{ ansible_hostname }} выполнена успешно!"
```

### Шаг 7.2. Запуск плейбука

```bash
cd ~/ansible-lab

# Запуск плейбука
ansible-playbook test-playbook.yml
```

### Шаг 7.3. Проверка результатов на Managed Nodes

**На Managed Node 1 (server1):**

```bash
# Проверка созданного файла
cat /tmp/ansible_test.txt
```

**На Managed Node 2 (server2):**

```bash
# Проверка созданного файла
cat /tmp/ansible_test.txt
```

---

## 📝 Задания для отчета

1. **Скриншоты выполнения:**
   - Установки Ansible (`ansible --version`)
   - Создания инвентаря (`ansible all --list-hosts`)
   - Проверки ping (`ansible all -m ping`)
   - Выполнения ad-hoc команд
   - Результата работы плейбука

2. **Ответы на вопросы:**
   - Зачем нужен безагентный подход Ansible?
   - Почему в тестовой среде отключена проверка host keys?
   - Что означает параметр `become: yes` в плейбуке?
   - Какую информацию содержат факты (facts) Ansible?

3. **Дополнительное задание:**
   - Добавьте в инвентарь переменную для каждого хоста (например, `app_port=8080` для web-server и `db_port=5432` для db-server)
   - Напишите ad-hoc команду для установки пакета `htop` на оба managed узла
   - Создайте плейбук, который устанавливает `nginx` на группу `webservers`

---

## 🔧 Возможные проблемы и решения

| Проблема | Решение |
|----------|---------|
| `Host key verification failed` | Выполнить `ssh-keyscan ip  >> ~/.ssh/known_hosts` или настроить `host_key_checking = False` |
| `Permission denied` при sudo | Проверить настройки sudoers: `ansible ALL=(ALL) NOPASSWD: ALL` |
| `Python not found` | Установить Python: `sudo apt install python3` |
| `Connection timeout` | Проверить сетевую связность: `ping 192.168.1.11` |
| `User ansible not exists` | Создать пользователя на всех машинах (пункт 1.1) |

---

## 🎯 Результат лабораторной

После выполнения всех шагов вы получите:
1. **Полностью настроенное окружение** для работы с Ansible
2. **Понимание базовых концепций** инвентаря, конфигурации и ad-hoc команд
3. **Первый рабочий плейбук** для проверки системы
4. **Готовность** к выполнению следующих лабораторных работ по Ansible
