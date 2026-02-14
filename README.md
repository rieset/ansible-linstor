# LINSTOR Ansible Playbook

Автоматизированное развертывание кластера LINSTOR® с помощью Ansible для Ubuntu 24.04+.

## Описание

Этот playbook автоматически настраивает и развертывает полнофункциональный кластер LINSTOR, включая:

- Проверку системных требований (Ubuntu 24.04+)
- Установку и настройку DRBD и LVM
- Установку и настройку LINSTOR Controller
- Установку и настройку LINSTOR Satellite узлов
- Инициализацию дисков и создание storage pools (lvm-thin)
- Настройку I/O scheduler для дисков DRBD
- Установку и доступ к LINSTOR GUI

## Системные требования

- **ОС**: Ubuntu 24.04 или новее (или совместимые варианты)
- **Ansible**: 2.9.0+ (рекомендуется последняя версия, например 2.20.2+)
- **Python**: python-netaddr должен быть установлен
- **SSH**: Passwordless SSH доступ ко всем целевым системам
- **Диски**: Минимум один дополнительный диск для storage pool (указывается в секции `[drbd]` в inventory)
- **Сеть**: Отдельная сеть для репликации DRBD (рекомендуется, но не обязательно)

## Быстрый старт

### 1. Настройка инвентаря

Используйте существующий файл конфигурации из директории `clusters/` или создайте новый файл инвентаря для вашего кластера.

**Пример структуры файла инвентаря:**

```ini
[controller]
192.168.1.11
192.168.1.12
192.168.1.13

[satellite]
192.168.1.11
192.168.1.12
192.168.1.13

[linstor_storage_pool]
192.168.1.11
192.168.1.12
192.168.1.13

[admin]
192.168.1.11

[linstor_cluster:children]
controller
satellite

[drbd]
sdb
sdc
```

**Группы:**
- `[controller]` - узлы контроллера LINSTOR
- `[satellite]` - спутниковые узлы (могут быть Combined, если также в группе controller)
- `[linstor_storage_pool]` - узлы, которые предоставляют storage pool
- `[admin]` - узел для установки LINSTOR GUI
- `[drbd]` - список дисков для DRBD (sdb, sdc и т.д.)

**Создание нового файла инвентаря:**

Создайте новый файл в директории `clusters/` (например, `clusters/my-cluster.ini`) и заполните его по образцу выше, указав IP адреса ваших узлов и диски.

### 2. Настройка переменных

Отредактируйте `group_vars/all.yaml`:

```yaml
---
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ansible_become: yes

drbd_backing_disk: /dev/sdb
drbd_replication_network: 192.168.100.0/24
```

**Параметры:**
- `ansible_user` - пользователь для SSH подключения
- `ansible_ssh_private_key_file` - путь к SSH приватному ключу
- `drbd_backing_disk` - диск по умолчанию для DRBD (если не указан в секции [drbd])
- `drbd_replication_network` - сеть для репликации DRBD в формате CIDR

### 3. Запуск установки

```bash
ansible-playbook -i clusters/<your-cluster>.ini ubuntu.yaml
```

Замените `<your-cluster>.ini` на имя вашего файла инвентаря (например, `clusters/linstor-52.ini`).

## Процесс установки

Playbook выполняет следующие этапы:

1. **Создание временной директории Ansible** - создает `/tmp/ansible-tmp` на всех узлах
2. **Проверка системы** - проверяет версию Ubuntu (требуется 24.04+)
3. **Проверка дисков** - проверяет наличие дисков из секции `[drbd]` в inventory
4. **Инициализация дисков** - создает PV, VG, thin pool и настраивает I/O scheduler
5. **Установка пакетов** - устанавливает DRBD, LVM и LINSTOR компоненты
6. **Настройка Controller** - запускает и настраивает LINSTOR Controller
7. **Настройка Satellite** - запускает и настраивает LINSTOR Satellite узлы
8. **Регистрация узлов** - регистрирует satellite узлы в кластере
9. **Создание storage pools** - создает lvm-thin storage pools на узлах
10. **Установка GUI** - устанавливает LINSTOR GUI (встроен в контроллер)

Подробное описание процесса см. в [docs/installation_process.md](docs/installation_process.md).

## Доступ к LINSTOR GUI

После успешной установки GUI доступен через веб-браузер:

```
http://<controller-ip>:3370/ui/
```

**Учетные данные по умолчанию:**
- Username: `admin`
- Password: `admin`

Рекомендуется изменить пароль после первого входа.

## Проверка установки

Подключитесь к любому контроллеру и выполните:

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

# Создание volume definition (100 MiB)
linstor volume-definition create test-res-0 100MiB

# Создание ресурса на узле (используйте имя узла из linstor node list)
linstor resource create <node-name> test-res-0 --storage-pool lvm-thin

# Проверка
linstor resource list
```

## Структура проекта

```
linstor-ansible/
├── ubuntu.yaml                  # Главный playbook (Ubuntu)
├── clusters/                    # Директория с inventory файлами
│   ├── linstor-52.ini          # Пример: инвентарь хостов для кластера
│   └── <your-cluster>.ini      # Ваш файл инвентаря
├── group_vars/
│   └── all.yaml                # Глобальные переменные
├── docs/                        # Документация
│   ├── installation_process.md # Процесс установки
│   ├── troubleshooting.md      # Устранение проблем
│   └── utilities/              # Вспомогательные playbooks
└── roles/
    ├── commons/                 # Общие роли
    │   ├── os-checker/         # Проверка ОС и дисков
    │   └── pre-install/        # Предустановка (пакеты, диски)
    └── linstor/                # Роли LINSTOR
        ├── controller/         # Контроллер LINSTOR
        ├── satellite/          # Спутниковые узлы
        ├── storage-pool/       # Создание пулов хранения
        └── gui/                # LINSTOR GUI
```

## Установленные пакеты

Playbook автоматически устанавливает следующие пакеты из репозитория LINBIT PPA:

- `drbd-utils` - утилиты DRBD
- `drbd-dkms` - модуль ядра DRBD (DKMS)
- `lvm2` - Logical Volume Manager
- `linstor-controller` - контроллер LINSTOR
- `linstor-satellite` - спутниковый узел LINSTOR
- `linstor-client` - клиент LINSTOR
- `linstor-gui` - веб-интерфейс (встроен в контроллер)

## Типы узлов

### Controller
Узел только с контроллером LINSTOR. Управляет кластером, но не хранит данные.

### Satellite
Узел только со спутником LINSTOR. Хранит данные, но не управляет кластером.

### Combined
Узел с контроллером и спутником одновременно. Может управлять кластером и хранить данные.

Чтобы создать Combined узел, добавьте его в обе группы `[controller]` и `[satellite]` в inventory.

## Особенности

- ✅ **Поддержка Ubuntu 24.04+**: Автоматическая проверка версии ОС
- ✅ **Автоматическая установка**: Установка последних версий из репозиториев LINBIT
- ✅ **Гибкая конфигурация**: Поддержка различных типов узлов
- ✅ **Инициализация дисков**: Автоматическое создание PV, VG, thin pool
- ✅ **Оптимизация I/O**: Настройка scheduler для дисков DRBD
- ✅ **Встроенный GUI**: Веб-интерфейс доступен через контроллер

## Устранение проблем

См. [docs/troubleshooting.md](docs/troubleshooting.md) для решения распространенных проблем.

### Быстрая проверка

```bash
# Проверка подключения
ansible all -i clusters/<your-cluster>.ini -m ping

# Проверка версии Ubuntu
ansible all -i clusters/<your-cluster>.ini -m shell -a "lsb_release -a"

# Проверка доступных дисков
ansible linstor_storage_pool -i clusters/<your-cluster>.ini -m shell -a "lsblk -o NAME,SIZE,MODEL,SERIAL,ROTA,TYPE"
```

Замените `<your-cluster>.ini` на имя вашего файла инвентаря.

## Дополнительная документация

- [Процесс установки](docs/installation_process.md) - подробное описание этапов установки
- [Устранение проблем](docs/troubleshooting.md) - решение распространенных проблем
- [Официальная документация LINSTOR](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/)

## Справочная информация

Для получения дополнительной информации о LINSTOR - например, инструкции по интеграции с Kubernetes, OpenStack, Docker или другими системами - обратитесь к [документации LINBIT по LINSTOR](https://linbit.com/drbd-user-guide/linstor-guide-1_0-en/).

## Лицензия

Этот проект распространяется под лицензией MIT.

Copyright (c) 2026 Albert Iblyaminov

См. файл [LICENSE](LICENSE) для подробной информации.

**Примечание:** LINSTOR является продуктом LINBIT и имеет свою собственную лицензию.
