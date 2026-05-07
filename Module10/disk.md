# Лабораторная работа №10: Управление дисками, LVM и монтированием

## Цель работы
Научиться управлять дисками, разделами и файловыми системами с помощью Ansible: создавать разделы, настраивать LVM (Logical Volume Manager), форматировать и монтировать тома.

## Окружение
- **1 управляющий узел (Control Node)** — Linux Debian 12
- **2 управляемых хоста** — Linux Debian 12 (web-server, db-server)

**Требования:** Все три хоста уже настроены для работы с Ansible (пользователь ansible, SSH-ключи, sudo без пароля). На каждом managed хосте должен быть добавлен **дополнительный виртуальный диск** (например, `/dev/sdb`).

---

## Часть 1. Подготовка рабочей среды (10 минут)

### Шаг 1.1. Создание директории для лабораторной

```bash
mkdir -p ~/lab10-storage
cd ~/lab10-storage
```

### Шаг 1.2. Создание файла инвентаря

```bash
nano ~/lab10-storage/inventory.ini
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
```

### Шаг 1.3. Создание конфигурационного файла

```bash
nano ~/lab10-storage/ansible.cfg
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

### Шаг 1.4. Проверка наличия дополнительного диска

```bash
# Проверка на web-server
ssh ansible@192.168.1.11 "lsblk"

# Проверка на db-server
ssh ansible@192.168.1.12 "lsblk"
```

**Ожидаемый вывод (должен быть диск sdb или vdb):**
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   20G  0 disk
├─sda1   8:1    0   19G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0    5G  0 disk
```

---

## Часть 2. Управление разделами (модуль `parted`) (20 минут)

### Шаг 2.1. Playbook для создания разделов на диске

```bash
nano ~/lab10-storage/01-create-partitions.yml
```

```yaml
---
- name: Создание разделов на диске /dev/sdb
  hosts: all
  gather_facts: yes
  
  tasks:
    # ===== ПРОВЕРКА НАЛИЧИЯ ДИСКА =====
    - name: Проверка существования диска /dev/sdb
      ansible.builtin.stat:
        path: /dev/sdb
      register: disk_status
    
    - name: Остановка, если диск не найден
      ansible.builtin.fail:
        msg: "Диск /dev/sdb не найден на {{ inventory_hostname }}!"
      when: not disk_status.stat.exists
    
    # ===== ОЧИСТКА СТАРЫХ РАЗДЕЛОВ (опционально) =====
    - name: Удаление существующих разделов на /dev/sdb
      ansible.builtin.parted:
        device: /dev/sdb
        state: absent
      when: disk_status.stat.exists
    
    # ===== СОЗДАНИЕ ТАБЛИЦЫ GPT =====
    - name: Создание GPT таблицы разделов
      ansible.builtin.parted:
        device: /dev/sdb
        label: gpt
        state: present
    
    # ===== СОЗДАНИЕ ПЕРВОГО РАЗДЕЛА (2GB, для данных) =====
    - name: Создание раздела 1 на 2GB
      ansible.builtin.parted:
        device: /dev/sdb
        number: 1
        part_start: 0%
        part_end: 2GiB
        state: present
    
    # ===== СОЗДАНИЕ ВТОРОГО РАЗДЕЛА (оставшееся место, LVM) =====
    - name: Создание раздела 2 для LVM
      ansible.builtin.parted:
        device: /dev/sdb
        number: 2
        part_start: 2GiB
        part_end: 100%
        flags: [ lvm ]
        state: present
    
    # ===== ПРОВЕРКА СОЗДАННЫХ РАЗДЕЛОВ =====
    - name: Получение информации о разделах
      ansible.builtin.parted:
        device: /dev/sdb
        unit: MiB
      register: partition_info
    
    - name: Вывод информации о разделах
      ansible.builtin.debug:
        msg: "Разделы на /dev/sdb: {{ partition_info.partitions }}"
```

### Шаг 2.2. Выполнение playbook

```bash
ansible-playbook 01-create-partitions.yml

# Проверка результата
ssh ansible@192.168.1.11 "lsblk /dev/sdb"
```

**Ожидаемый вывод:**
```
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sdb      8:16   0   5G  0 disk
├─sdb1   8:17   0   2G  0 part
└─sdb2   8:18   0   3G  0 part
```

---

## Часть 3. Форматирование разделов (модуль `filesystem`) (15 минут)

### Шаг 3.1. Playbook для форматирования

```bash
nano ~/lab10-storage/02-format-filesystems.yml
```

```yaml
---
- name: Форматирование разделов в файловые системы
  hosts: all
  
  tasks:
    # ===== ФОРМАТИРОВАНИЕ ПЕРВОГО РАЗДЕЛА (ext4) =====
    - name: Форматирование /dev/sdb1 в ext4
      ansible.builtin.filesystem:
        dev: /dev/sdb1
        fstype: ext4
        force: no      # Не форматировать, если уже есть ФС
    
    # ===== ФОРМАТИРОВАНИЕ ВТОРОГО РАЗДЕЛА (xfs для LVM) =====
    - name: Форматирование /dev/sdb2 (будет использован в LVM)
      ansible.builtin.filesystem:
        dev: /dev/sdb2
        fstype: xfs
        force: no
    
    # ===== ПРОВЕРКА ФОРМАТИРОВАНИЯ =====
    - name: Получение информации о файловых системах
      ansible.builtin.shell: "blkid /dev/sdb1 /dev/sdb2"
      register: blkid_info
      changed_when: false
    
    - name: Вывод информации
      ansible.builtin.debug:
        msg: "{{ blkid_info.stdout_lines }}"
```

### Шаг 3.2. Выполнение

```bash
ansible-playbook 02-format-filesystems.yml

# Проверка
ssh ansible@192.168.1.11 "sudo blkid /dev/sdb1 /dev/sdb2"
```

---

## Часть 4. Управление LVM (20 минут)

### Шаг 4.1. Playbook для настройки LVM

```bash
nano ~/lab10-storage/03-lvm-setup.yml
```

```yaml
---
- name: Настройка LVM (группы томов и логические тома)
  hosts: all
  
  tasks:
    # ===== СОЗДАНИЕ ГРУППЫ ТОМОВ (VG) =====
    - name: Создание группы томов vg_data из /dev/sdb2
      ansible.builtin.lvg:
        vg: vg_data
        pvs: /dev/sdb2
        pesize: 4
        state: present
    
    # ===== СОЗДАНИЕ ЛОГИЧЕСКОГО ТОМА ДЛЯ ПРИЛОЖЕНИЯ =====
    - name: Создание логического тома lv_app (1GB)
      ansible.builtin.lvol:
        vg: vg_data
        lv: lv_app
        size: 1G
        state: present
    
    # ===== СОЗДАНИЕ ЛОГИЧЕСКОГО ТОМА ДЛЯ БАЗЫ ДАННЫХ =====
    - name: Создание логического тома lv_db (1GB)
      ansible.builtin.lvol:
        vg: vg_data
        lv: lv_db
        size: 1G
        state: present
    
    # ===== ИСПОЛЬЗОВАНИЕ ОСТАВШЕГОСЯ МЕСТА =====
    - name: Создание логического тома lv_data (оставшееся место)
      ansible.builtin.lvol:
        vg: vg_data
        lv: lv_data
        size: 100%FREE
        state: present
    
    # ===== ПОЛУЧЕНИЕ ИНФОРМАЦИИ О VG =====
    - name: Получение информации о группе томов
      ansible.builtin.command: vgdisplay vg_data
      register: vg_info
      changed_when: false
    
    - name: Вывод информации о VG
      ansible.builtin.debug:
        msg: "{{ vg_info.stdout_lines }}"
    
    # ===== ПОЛУЧЕНИЕ ИНФОРМАЦИИ О LV =====
    - name: Список логических томов
      ansible.builtin.shell: lvs vg_data
      register: lv_list
      changed_when: false
    
    - name: Вывод списка LV
      ansible.builtin.debug:
        msg: "{{ lv_list.stdout_lines }}"
```

### Шаг 4.2. Выполнение

```bash
ansible-playbook 03-lvm-setup.yml

# Проверка
ssh ansible@192.168.1.11 "sudo vgdisplay; sudo lvs"
```

**Ожидаемый вывод `lvs`:**
```
  LV     VG      Attr       LSize
  lv_app vg_data -wi-a----- 1.00g
  lv_db  vg_data -wi-a----- 1.00g
  lv_data vg_data -wi-a----- 1.00g
```

### Шаг 4.3. Форматирование логических томов

```bash
nano ~/lab10-storage/04-format-lvols.yml
```

```yaml
---
- name: Форматирование логических томов
  hosts: all
  
  tasks:
    # ===== ФОРМАТИРОВАНИЕ LV_APP =====
    - name: Форматирование lv_app в ext4
      ansible.builtin.filesystem:
        dev: /dev/mapper/vg_data-lv_app
        fstype: ext4
        force: no
    
    # ===== ФОРМАТИРОВАНИЕ LV_DB =====
    - name: Форматирование lv_db в xfs
      ansible.builtin.filesystem:
        dev: /dev/mapper/vg_data-lv_db
        fstype: xfs
        force: no
    
    # ===== ФОРМАТИРОВАНИЕ LV_DATA =====
    - name: Форматирование lv_data в ext4
      ansible.builtin.filesystem:
        dev: /dev/mapper/vg_data-lv_data
        fstype: ext4
        force: no
    
    # ===== ПРОВЕРКА =====
    - name: Проверка файловых систем на LV
      ansible.builtin.shell: "blkid /dev/mapper/vg_data-lv_*"
      register: blkid_lvs
      changed_when: false
    
    - name: Результат форматирования
      ansible.builtin.debug:
        msg: "{{ blkid_lvs.stdout_lines }}"
```

```bash
ansible-playbook 04-format-lvols.yml
```

---

## Часть 5. Монтирование разделов и томов (20 минут)

### Шаг 5.1. Playbook для создания точек монтирования и монтирования

```bash
nano ~/lab10-storage/05-mount-volumes.yml
```

```yaml
---
- name: Монтирование разделов и логических томов
  hosts: all
  
  tasks:
    # ===== СОЗДАНИЕ ТОЧЕК МОНТИРОВАНИЯ =====
    - name: Создание точек монтирования
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /mnt/data_ext4
        - /mnt/app
        - /mnt/database
        - /mnt/storage
    
    # ===== МОНТИРОВАНИЕ /DEV/SDB1 (EXT4) =====
    - name: Монтирование /dev/sdb1 в /mnt/data_ext4
      ansible.builtin.mount:
        path: /mnt/data_ext4
        src: /dev/sdb1
        fstype: ext4
        opts: defaults,noatime
        state: mounted
    
    # ===== МОНТИРОВАНИЕ LV_APP =====
    - name: Монтирование логического тома lv_app в /mnt/app
      ansible.builtin.mount:
        path: /mnt/app
        src: /dev/mapper/vg_data-lv_app
        fstype: ext4
        opts: defaults
        state: mounted
    
    # ===== МОНТИРОВАНИЕ LV_DB =====
    - name: Монтирование логического тома lv_db в /mnt/database
      ansible.builtin.mount:
        path: /mnt/database
        src: /dev/mapper/vg_data-lv_db
        fstype: xfs
        opts: defaults
        state: mounted
    
    # ===== МОНТИРОВАНИЕ LV_DATA =====
    - name: Монтирование логического тома lv_data в /mnt/storage
      ansible.builtin.mount:
        path: /mnt/storage
        src: /dev/mapper/vg_data-lv_data
        fstype: ext4
        opts: defaults
        state: mounted
    
    # ===== ПРОВЕРКА МОНТИРОВАНИЯ =====
    - name: Проверка смонтированных файловых систем
      ansible.builtin.shell: "df -h | grep -E '/mnt/|/dev/sdb'"
      register: mount_check
      changed_when: false
    
    - name: Вывод информации о монтировании
      ansible.builtin.debug:
        msg: "{{ mount_check.stdout_lines }}"
    
    # ===== СОЗДАНИЕ ТЕСТОВЫХ ФАЙЛОВ =====
    - name: Создание тестового файла в /mnt/app
      ansible.builtin.copy:
        content: "Тестовый файл приложения\nСоздан: {{ ansible_date_time.date }}"
        dest: /mnt/app/test.txt
        mode: '0644'
    
    - name: Создание тестового файла в /mnt/database
      ansible.builtin.copy:
        content: "Тестовые данные базы данных"
        dest: /mnt/database/data.sql
        mode: '0644'
```

### Шаг 5.2. Выполнение

```bash
ansible-playbook 05-mount-volumes.yml

# Проверка на серверах
ssh ansible@192.168.1.11 "df -h | grep -E '/mnt|sdb'"
ssh ansible@192.168.1.11 "ls -la /mnt/*/"
```

---

## Часть 6. Расширение логических томов (15 минут)

### Шаг 6.1. Playbook для расширения LV

```bash
nano ~/lab10-storage/06-extend-lvm.yml
```

```yaml
---
- name: Расширение логических томов
  hosts: all
  
  tasks:
    # ===== ПОЛУЧЕНИЕ ТЕКУЩЕГО РАЗМЕРА LV_APP =====
    - name: Получение текущего размера lv_app
      ansible.builtin.shell: "lvs vg_data --noheadings -o lv_name,lv_size | grep lv_app | awk '{print $2}'"
      register: current_size
      changed_when: false
    
    - name: Текущий размер lv_app
      ansible.builtin.debug:
        msg: "Текущий размер lv_app: {{ current_size.stdout }}"
    
    # ===== РАСШИРЕНИЕ LV_APP ДО 2GB =====
    - name: Расширение lv_app до 2GB
      ansible.builtin.lvol:
        vg: vg_data
        lv: lv_app
        size: 2G
        resizefs: yes      # Автоматическое расширение файловой системы
        state: present
    
    # ===== ПРОВЕРКА НОВОГО РАЗМЕРА =====
    - name: Проверка нового размера
      ansible.builtin.shell: "lvs vg_data --noheadings -o lv_name,lv_size | grep lv_app"
      register: new_size
      changed_when: false
    
    - name: Вывод нового размера
      ansible.builtin.debug:
        msg: "Новый размер lv_app: {{ new_size.stdout }}"
    
    # ===== ПРОВЕРКА ФАЙЛОВОЙ СИСТЕМЫ =====
    - name: Проверка размера ФС
      ansible.builtin.shell: "df -h /mnt/app | tail -1"
      register: fs_size
      changed_when: false
    
    - name: Размер ФС после расширения
      ansible.builtin.debug:
        msg: "Файловая система: {{ fs_size.stdout }}"
```

### Шаг 6.2. Выполнение

```bash
ansible-playbook 06-extend-lvm.yml

# Проверка
ssh ansible@192.168.1.11 "lvs vg_data && df -h /mnt/app"
```

---

4. Освоили монтирование разделов и томов с добавлением в fstab
5. Научились расширять логические тома с автоматическим расширением ФС
6. Создали комплексный playbook для полной настройки дискового пространства
