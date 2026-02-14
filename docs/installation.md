# Инструкция по запуску установки LINSTOR

## Предварительные требования

1. **Ansible** версии 2.9.0 или выше (рекомендуется последняя версия)
   ```bash
   ansible --version
   ```
   **Примечание:** Минимальная версия - 2.9.0+, но рекомендуется использовать последнюю стабильную версию (например, 2.20.2+). Ваша версия 2.20.2 полностью совместима! ✅

2. **Python библиотека netaddr**
   ```bash
   pip install netaddr
   ```

3. **SSH доступ** ко всем целевым хостам без пароля (по ключу)

4. **Учетная запись LINBIT Portal** на https://my.linbit.com

## Шаг 1: Настройка инвентаря

Используйте файл из `clusters/` (например, `clusters/linstor.ini`) или создайте свой. Укажите IP-адреса или hostname ваших серверов:

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.21
192.168.1.22
192.168.1.23

[linstor_storage_pool]
192.168.1.21
192.168.1.22
192.168.1.23

[admin]
192.168.1.11

[linstor_cluster:children]
controller
satellite

[drbd]
sdb
sdc
```

**Примечания:**
- Узел может быть одновременно в группах `controller` и `satellite` — это создаст Combined узел
- Узлы в группе `linstor_storage_pool` будут предоставлять блочное хранилище (LVM thin pool)
- Группа `[admin]` — узел, на котором устанавливается LINSTOR GUI (если используется роль gui)
- Секция `[drbd]` — список дисков для storage pool (sdb, sdc и т.д.)

## Шаг 2: Настройка переменных

Отредактируйте файл `group_vars/all.yaml`:

```yaml
---
# Ansible variables
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ansible_become: yes

# LINSTOR variables
drbd_backing_disk: /dev/sdb
drbd_replication_network: 192.168.100.0/24

# LINBIT portal variables
lb_user: ""
lb_pass: ""
lb_con_id: ""
lb_clu_id: ""
```

### Параметры:

- **ansible_user** — пользователь для SSH подключения
- **ansible_ssh_private_key_file** — путь к приватному SSH ключу
- **ansible_become** — повышение привилегий (в playbook используется `become: true`)
- **drbd_backing_disk** — неиспользуемый блочный диск для LVM (например, `/dev/sdb`)
  - Если диска нет, не добавляйте узел в `linstor_storage_pool` — будет создан только file-thin pool
- **drbd_replication_network** - сеть для репликации DRBD в формате CIDR (рекомендуется отдельная сеть)
- **lb_user, lb_pass** - учетные данные для LINBIT Portal
- **lb_con_id** - ID контракта в LINBIT Portal
- **lb_clu_id** - ID кластера в LINBIT Portal

## Шаг 3: Проверка подключения

Проверьте, что Ansible может подключиться ко всем хостам:

```bash
ansible all -i clusters/linstor.ini -m ping
```

Должны увидеть `SUCCESS` для всех хостов. Замените `linstor.ini` на ваш файл инвентаря при необходимости.

## Шаг 4: Запуск установки

### Базовый запуск (использует переменные из group_vars/all.yaml):

```bash
ansible-playbook ubuntu.yaml
```

### Запуск с указанием инвентаря:

```bash
ansible-playbook -i clusters/linstor.ini ubuntu.yaml
```

### Запуск с передачей учетных данных LINBIT через командную строку:

Если не хотите хранить пароли в файле:

```bash
ansible-playbook ubuntu.yaml \
  -e lb_user="ваш_логин" \
  -e lb_pass="ваш_пароль" \
  -e lb_con_id="1234" \
  -e lb_clu_id="4321"
```

### Запуск с ограничением на определенные хосты:

```bash
# Только controller
ansible-playbook ubuntu.yaml --limit controller

# Только satellite
ansible-playbook ubuntu.yaml --limit satellite

# Конкретный хост
ansible-playbook ubuntu.yaml --limit 192.168.1.11
```

### Запуск с тегами (выполнение отдельных ролей):

```bash
# Только установка controller
ansible-playbook ubuntu.yaml --tags controller

# Только установка satellite
ansible-playbook ubuntu.yaml --tags satellite

# Только создание storage pool
ansible-playbook ubuntu.yaml --tags storage-pool
```

### Проверка без выполнения (dry-run):

```bash
ansible-playbook ubuntu.yaml --check
```

### Подробный вывод (verbose):

```bash
ansible-playbook ubuntu.yaml -v    # -v, -vv, -vvv для большей детализации
```

## Шаг 5: Проверка установки

После успешного выполнения playbook, подключитесь к controller узлу и проверьте:

```bash
# Проверка узлов
linstor node list

# Проверка storage pools
linstor storage-pool list

# Проверка ресурсов
linstor resource list
```

## Создание тестового ресурса

```bash
# Создание resource definition
linstor resource-definition create test-res-0

# Создание volume definition
linstor volume-definition create test-res-0 100MiB

# Создание ресурса на узле
linstor resource create \
  $(linstor sp list | head -n4 | tail -n1 | cut -d"|" -f3 | sed 's/ //g') \
  test-res-0 --storage-pool lvm-thin

# Проверка
linstor resource list
```

## Устранение проблем

### Ошибка подключения SSH

```bash
# Проверьте SSH ключ
ssh -i ~/.ssh/ansible_key ansible@192.168.1.11

# Проверьте, что ключ добавлен в ssh-agent
ssh-add ~/.ssh/ansible_key
```

### Ошибка регистрации в LINBIT Portal

- Проверьте правильность учетных данных
- Убедитесь, что у вас есть активный контракт
- Проверьте доступность https://my.linbit.com

### Ошибка установки пакетов

- Убедитесь, что узлы имеют доступ к интернету
- Проверьте, что репозитории LINBIT настроены корректно после регистрации
- Проверьте версию ОС (должна быть поддерживаемая)

### Проблемы с firewall

Playbook автоматически настраивает firewall, но если есть проблемы:

```bash
# RHEL/CentOS
firewall-cmd --list-ports

# Ubuntu
ufw status
```

## Полезные команды

### Проверка конфигурации Ansible:

```bash
ansible-config dump
```

### Проверка синтаксиса playbook:

```bash
ansible-playbook ubuntu.yaml --syntax-check
```

### Просмотр фактов о хостах:

```bash
ansible all -i hosts.ini -m setup
```

### Выполнение команды на всех хостах:

```bash
ansible all -i hosts.ini -a "systemctl status linstor-controller"
```

## Примеры использования

### Установка только на новых узлах:

```bash
# Добавьте новые узлы в hosts.ini
# Затем запустите с ограничением
ansible-playbook ubuntu.yaml --limit new_nodes
```

### Обновление только controller:

```bash
ansible-playbook ubuntu.yaml --tags controller --limit controller
```

### Пересоздание storage pool:

```bash
ansible-playbook ubuntu.yaml --tags storage-pool
```

