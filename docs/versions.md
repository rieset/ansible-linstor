# Места определения версий в проекте

Этот документ содержит полный список всех мест, где определяются версии пакетов, ОС и зависимостей.

## 1. Версии операционных систем

### readme.md
**Файл:** `readme.md`  
**Строка:** 22  
**Текущее значение:**
```markdown
Target systems are RHEL 7/8/9 or Ubuntu 22.04, 24.04, 25.04+ (or compatible variants).
```
**Статус:** ✅ Обновлено

### Проверка версии ОС
**Файл:** `roles/commons/os-checker/tasks/main.yaml`  
**Строки:** 4-8  
**Описание:** Определяет версию ОС из `/etc/os-release`, но не использует её для ограничений - только для определения семейства ОС (RedHat/Debian)

```yaml
- name: get os_version from /etc/os-release
  when: ansible_os_family is not defined
  raw: "grep '^VERSION_ID=' /etc/os-release | sed s'/VERSION_ID=//'"
  register: os_version
  changed_when: False
```

## 2. Версия Ansible

### readme.md
**Файл:** `readme.md`  
**Строка:** 19  
**Текущее значение:**
```markdown
Deployment environment must have Ansible `2.9.0+` and `python-netaddr`.
```
**Статус:** ✅ Обновлено (было 2.7.0+)

## 3. Версии пакетов LINSTOR

### ⚠️ ВАЖНО: Версии пакетов НЕ зафиксированы явно

Пакеты устанавливаются **без указания версий**, что означает установку последних доступных версий из репозиториев LINBIT.

### Определение списка пакетов для RHEL/CentOS

**Файл:** `roles/linstor/controller/meta/main.yaml`  
**Строки:** 4-6  
**Пакеты:**
```yaml
lb_rpm_pkgs: ["kmod-drbd", "drbd", "linstor-controller", "linstor-satellite", "linstor-client", "python-linstor"]
```

**Файл:** `roles/linstor/satellite/meta/main.yaml`  
**Строки:** 4-6  
**Пакеты:**
```yaml
lb_rpm_pkgs: ["kmod-drbd", "drbd", "linstor-controller", "linstor-client", "linstor-satellite", "python-linstor"]
```

### Определение списка пакетов для Ubuntu/Debian

**Файл:** `roles/linstor/controller/meta/main.yaml`  
**Строки:** 4-6  
**Пакеты:**
```yaml
lb_deb_pkgs: ["drbd-dkms", "drbd-utils", "linstor-controller", "linstor-satellite", "linstor-client", "python-linstor"]
```

**Файл:** `roles/linstor/satellite/meta/main.yaml`  
**Строки:** 4-6  
**Пакеты:**
```yaml
lb_deb_pkgs: ["drbd-dkms", "drbd-utils", "linstor-controller", "linstor-client", "linstor-satellite", "python-linstor"]
```

### Установка пакетов

**Файл:** `roles/commons/pre-install/tasks/pkg.yaml`  
**Строки:** 12-24  

**Для RHEL:**
```yaml
- name: install LINBIT packages (RHEL)
  when: ansible_os_family == "RedHat"
  yum:
    name: "{{ item }}"
    update_cache: yes
  with_items: "{{ lb_rpm_pkgs }}"
```

**Для Ubuntu:**
```yaml
- name: install LINBIT packages (Ubuntu)
  when: ansible_os_family == "Debian"
  apt:
    name: "{{ item }}"
    update_cache: yes
  with_items: "{{ lb_deb_pkgs }}"
```

**Примечание:** Модули `yum` и `apt` без указания версии устанавливают последнюю доступную версию из репозитория.

## 4. Версия linbit-manage-node.py

**Файл:** `roles/commons/pre-install/tasks/pkg.yaml`  
**Строки:** 2-7  

```yaml
- name: fetch the latest linbit-manage-node.py
  get_url:
    url: "https://my.linbit.com/linbit-manage-node.py"
    dest: "/root/linbit-manage-node.py"
    mode: "0640"
    force: "yes"
```

**Примечание:** Скрипт всегда загружается с `force: yes`, что означает загрузку последней версии при каждом запуске.

## 5. Как добавить явное указание версий пакетов

Если необходимо зафиксировать конкретные версии пакетов, нужно изменить файл `roles/commons/pre-install/tasks/pkg.yaml`:

### Для RHEL (yum):
```yaml
- name: install LINBIT packages (RHEL)
  when: ansible_os_family == "RedHat"
  yum:
    name: "{{ item.name }}"
    version: "{{ item.version }}"
    update_cache: yes
  with_items: "{{ lb_rpm_pkgs_with_versions }}"
```

И определить переменную в `group_vars/all.yaml` или в meta роли:
```yaml
lb_rpm_pkgs_with_versions:
  - { name: "kmod-drbd", version: "9.2.0" }
  - { name: "drbd", version: "9.2.0" }
  - { name: "linstor-controller", version: "1.25.0" }
  # и т.д.
```

### Для Ubuntu (apt):
```yaml
- name: install LINBIT packages (Ubuntu)
  when: ansible_os_family == "Debian"
  apt:
    name: "{{ item.name }}={{ item.version }}"
    update_cache: yes
  with_items: "{{ lb_deb_pkgs_with_versions }}"
```

## 6. Сводная таблица

| Компонент | Файл | Строка | Текущее значение | Статус |
|-----------|------|--------|------------------|--------|
| Ubuntu версии | readme.md | 22 | Ubuntu 22.04, 24.04, 25.04+ | ✅ Обновлено |
| RHEL версии | readme.md | 22 | RHEL 7/8/9 | ✅ Актуально |
| Ansible версия | readme.md | 19 | Ansible 2.9.0+ | ✅ Обновлено |
| Пакеты LINSTOR | meta/main.yaml | 4-6 | Без версий (последние) | ⚠️ Требует проверки |
| linbit-manage-node.py | pkg.yaml | 2-7 | Всегда последняя | ✅ Актуально |

## 7. Рекомендации

1. **Для production:** Рассмотреть возможность фиксации версий пакетов LINSTOR для обеспечения стабильности
2. **Для development:** Оставить установку последних версий для получения новых функций и исправлений
3. **Мониторинг:** Регулярно проверять доступность пакетов для новых версий Ubuntu в репозиториях LINBIT
4. **Тестирование:** Перед обновлением версий ОС тестировать playbook на тестовом окружении

