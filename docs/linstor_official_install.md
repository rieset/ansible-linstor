# Саммари по официальной установке LINSTOR

Референс: [Официальная инструкция LINSTOR](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#s-installation)

## Официальный процесс установки

### 1. Регистрация узла в LINBIT Portal

**Ручная установка:**
```bash
# Загрузка скрипта регистрации
curl -O https://my.linbit.com/linbit-manage-node.py
chmod +x ./linbit-manage-node.py

# Запуск регистрации (требует учетные данные LINBIT Portal)
./linbit-manage-node.py
```

**Что делает скрипт:**
- Автоматически определяет ОС и версию (Ubuntu 22.04/24.04/25.04, RHEL 7/8/9)
- Регистрирует узел в LINBIT Portal по учетным данным
- Настраивает репозитории LINBIT для данной ОС
- Включает необходимые репозитории (drbd-9, linstor и т.д.)

**Адаптация для Ansible:**
- Загрузка через `get_url` модуль
- Запуск через `shell` модуль с переменными окружения
- Выполняется автоматически на всех узлах

### 2. Установка пакетов

**Для RHEL/CentOS:**
```bash
yum install kmod-drbd drbd linstor-controller linstor-satellite linstor-client python-linstor
```

**Для Ubuntu/Debian:**
```bash
apt install drbd-dkms drbd-utils linstor-controller linstor-satellite linstor-client python-linstor
```

**Адаптация для Ansible:**
- Используются модули `yum` и `apt`
- Пакеты устанавливаются без указания версий (последние из репозиториев)
- Условная установка в зависимости от `ansible_os_family`

### 3. Запуск сервисов

**Controller:**
```bash
systemctl enable linstor-controller
systemctl start linstor-controller
```

**Satellite:**
```bash
systemctl enable linstor-satellite
systemctl start linstor-satellite
```

**Адаптация для Ansible:**
- Используется модуль `systemd` с параметрами `enabled: yes` и `state: restarted`

### 4. Инициализация кластера

**Создание первого узла (Controller):**
```bash
linstor node create <node-name> <ip-address> --node-type Controller
```

**Добавление Satellite узлов:**
```bash
linstor node create <node-name> <ip-address> --node-type Satellite
```

**Создание Combined узла:**
```bash
linstor node create <node-name> <ip-address> --node-type Combined
```

**Адаптация для Ansible:**
- Выполняется через `shell` модуль
- Тип узла определяется автоматически по группам в inventory
- Используется фильтр `ipaddr` для выбора IP из сети репликации

## Ключевые отличия Ansible playbook от ручной установки

### Автоматизация
- ✅ Все команды выполняются автоматически на всех узлах
- ✅ Нет необходимости подключаться к каждому узлу вручную
- ✅ Единая конфигурация для всего кластера

### Конфигурация
- ✅ Переменные централизованы в `group_vars/all.yaml`
- ✅ Учетные данные можно передавать через командную строку
- ✅ Гибкая настройка через inventory файлы

### Безопасность
- ✅ Настройка firewall правил автоматически
- ✅ Конфигурация клиента LINSTOR через шаблоны
- ✅ Единообразная настройка на всех узлах

### Storage Pools
- ✅ Автоматическое создание file-thin pool на всех satellite
- ✅ Автоматическое создание lvm-thin pool на узлах с дисками
- ✅ Настройка LVM volume groups и thin pools

## Соответствие официальной инструкции

| Официальный шаг | Ansible playbook | Файл/Роль |
|-----------------|------------------|-----------|
| Регистрация узла | `linbit-manage-node.py` | `roles/commons/pre-install/tasks/pkg.yaml` |
| Установка пакетов | `yum/apt` модули | `roles/commons/pre-install/tasks/pkg.yaml` |
| Запуск Controller | `systemd` модуль | `roles/linstor/controller/tasks/main.yaml` |
| Запуск Satellite | `systemd` модуль | `roles/linstor/satellite/tasks/main.yaml` |
| Инициализация узлов | `linstor node create` | `roles/linstor/controller/tasks/main.yaml`, `roles/linstor/satellite/tasks/main.yaml` |
| Настройка firewall | `firewalld/ufw` модули | `roles/linstor/controller/tasks/main.yaml`, `roles/linstor/satellite/tasks/main.yaml` |
| Создание storage pools | `linstor storage-pool create` | `roles/linstor/satellite/tasks/main.yaml`, `roles/linstor/storage-pool/tasks/main.yaml` |

## Поддерживаемые ОС

Согласно официальной документации и нашему playbook:

- **RHEL:** 7, 8, 9
- **Ubuntu:** 22.04, 24.04, 25.04+
- **CentOS:** через определение как RedHat семейство

## Требования

- Учетная запись LINBIT Portal (https://my.linbit.com)
- Contract ID и Cluster ID из портала
- SSH доступ ко всем узлам
- Python на управляющей машине (для Ansible)

## Полезные ссылки

- [Официальная документация LINSTOR](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/)
- [LINBIT Portal](https://my.linbit.com)
- [Введение в LINSTOR](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/#p-linstor-introduction)

