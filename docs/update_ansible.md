# Обновление Ansible

## Текущая версия

У вас установлен: **Ansible [core 2.17.14]**

Путь установки: `/Users/rie/Library/Python/3.12/bin/ansible`

Это указывает на установку через **pip** (Python package manager).

## Способы обновления

### Способ 1: Обновление через pip3 (РЕКОМЕНДУЕТСЯ)

```bash
# Обновить ansible-core до последней версии
python3 -m pip install --upgrade ansible-core

# Или если установлен ansible (старый пакет)
python3 -m pip install --upgrade ansible
```

### Способ 2: Обновление до конкретной версии

```bash
# Обновить до последней версии 2.17.x
python3 -m pip install --upgrade "ansible-core>=2.17.0"

# Или до конкретной версии
python3 -m pip install --upgrade ansible-core==2.17.15
```

### Способ 3: Переустановка (если есть проблемы)

```bash
# Удалить текущую версию
python3 -m pip uninstall ansible-core

# Установить заново
python3 -m pip install ansible-core
```

## Проверка после обновления

```bash
# Проверить версию
ansible --version

# Проверить, что все работает
ansible --help
```

## Установка зависимостей

Убедитесь, что установлена библиотека `netaddr` (требуется для playbook):

```bash
python3 -m pip install netaddr
```

## Альтернативные способы установки

### Через Homebrew (macOS)

Если хотите использовать Homebrew вместо pip:

```bash
# Установить через Homebrew
brew install ansible

# Обновить через Homebrew
brew upgrade ansible
```

### Через pipx (изолированная установка)

```bash
# Установить pipx (если еще нет)
python3 -m pip install --user pipx

# Установить ansible через pipx
pipx install ansible-core

# Обновить через pipx
pipx upgrade ansible-core
```

## Рекомендации

### Для вашего случая (установка через pip):

```bash
# 1. Обновить ansible-core
python3 -m pip install --upgrade ansible-core

# 2. Убедиться, что netaddr установлен
python3 -m pip install netaddr

# 3. Проверить версию
ansible --version
```

### Проверка совместимости

Ваш playbook требует Ansible `2.9.0+`, а у вас `2.20.2` - это отлично! ✅

**Требования:**
- Минимальная версия: **2.9.0+**
- Ваша версия: **2.20.2** (последняя стабильная)
- Статус: **Полностью совместима** ✅

## Устранение проблем

### ⚠️ Проблема: Конфликт зависимостей (ansible vs ansible-core)

**Ошибка:**
```
ERROR: ansible 10.7.0 requires ansible-core~=2.17.7, but you have ansible-core 2.20.2 which is incompatible.
```

**Решение:** Удалить старый пакет `ansible` и оставить только `ansible-core`:

```bash
# 1. Удалить старый метапакет ansible
python3 -m pip uninstall -y ansible

# 2. Проверить, что остался только ansible-core
python3 -m pip list | grep -i ansible

# 3. Проверить версию
ansible --version
```

**Почему это происходит:**
- `ansible` - это старый метапакет, который включает `ansible-core` как зависимость
- `ansible-core` - это новый основной пакет (рекомендуется использовать его)
- Они конфликтуют, если установлены оба

**Рекомендация:** Используйте только `ansible-core`, метапакет `ansible` больше не нужен.

### Проблема: "command not found" после обновления

```bash
# Проверить, что путь в PATH
echo $PATH | grep -i python

# Если нет, добавить в ~/.zshrc
echo 'export PATH="$HOME/Library/Python/3.12/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Проблема: Права доступа

```bash
# Использовать --user флаг
python3 -m pip install --user --upgrade ansible-core

# Или исправить права на кэш
sudo chown -R $(whoami) ~/Library/Caches/pip
```

### Проблема: Полная переустановка

Если ничего не помогает:

```bash
# 1. Удалить все версии
python3 -m pip uninstall -y ansible ansible-core

# 2. Очистить кэш
python3 -m pip cache purge

# 3. Установить заново только ansible-core
python3 -m pip install --user ansible-core

# 4. Установить зависимости
python3 -m pip install --user netaddr
```

## Текущие требования проекта

Согласно `readme.md`:
- Минимальная версия: **Ansible 2.9.0+**
- Рекомендуемая версия: **Последняя стабильная (2.20.2+)**
- Ваша версия: **2.20.2** ✅ (полностью совместима и рекомендуется)

## Быстрая команда для обновления

```bash
python3 -m pip install --upgrade ansible-core && ansible --version
```

