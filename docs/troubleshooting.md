# Устранение проблем при установке

## Проблема: Timeout при создании временного файла

### Ошибка:
```
[ERROR]: Task failed: Module failed: failed to create temporary content file: The read operation timed out
```

### Причины:
1. **Проблема с remote_tmp директорией** - неправильные права или путь
2. **Проблема с pipelining** - конфликт при использовании become
3. **Медленное соединение** - timeout при загрузке файлов

### Решения:

#### Решение 1: Исправить remote_tmp в ansible.cfg

Измените `ansible.cfg`:
```ini
remote_tmp = /tmp/.ansible-${USER}/tmp
```

Или используйте абсолютный путь:
```ini
remote_tmp = /tmp/ansible-tmp
```

#### Решение 2: Отключить pipelining (временно)

Если проблема сохраняется, временно отключите pipelining:
```ini
pipelining = False
```

#### Решение 3: Увеличить timeout

Добавьте в `ansible.cfg`:
```ini
timeout = 60
```

И в задачу `get_url` добавьте:
```yaml
timeout: 60
```

#### Решение 4: Создать remote_tmp вручную на хостах

Выполните на каждом хосте:
```bash
sudo mkdir -p /tmp/.ansible-tmp
sudo chmod 1777 /tmp/.ansible-tmp
```

Или через Ansible:
```bash
ansible all -i clusters/linstor-52.ini -m file -a "path=/tmp/.ansible-tmp state=directory mode=1777" --become
```

## Проблема: Предупреждение о remote_tmp

### Ошибка:
```
[WARNING]: Module remote_tmp /root/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when running as another user.
```

### Решение:

Используйте путь, доступный для всех пользователей:
```ini
remote_tmp = /tmp/.ansible-${USER}/tmp
```

Или создайте директорию заранее с правильными правами.

## Проблема: Пользователь с become

Если вы используете не-root пользователя с `become: yes`:

1. **Убедитесь, что пользователь может использовать sudo:**
   ```bash
   ssh <user>@<host> "sudo -n true"
   ```

2. **Проверьте, что sudo не требует пароль:**
   ```bash
   ssh <user>@<host> "echo '<user> ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/<user>"
   ```
   
   Замените `<user>` на имя вашего пользователя и `<host>` на IP адрес хоста.

3. **Или используйте root напрямую:**
   Измените в `group_vars/all.yaml`:
   ```yaml
   ansible_user: root
   ```

## Проблема: Timeout при загрузке linbit-manage-node.py

### Решение:

1. **Увеличить timeout в задаче:**
   ```yaml
   - name: fetch the latest linbit-manage-node.py
     get_url:
       url: "https://my.linbit.com/linbit-manage-node.py"
       dest: "/root/linbit-manage-node.py"
       mode: "0640"
       force: "yes"
       timeout: 60
   ```

2. **Проверить доступность URL:**
   ```bash
   curl -I https://my.linbit.com/linbit-manage-node.py
   ```

3. **Загрузить файл вручную и скопировать:**
   ```bash
   # На локальной машине
   curl -o linbit-manage-node.py https://my.linbit.com/linbit-manage-node.py
   
   # Скопировать на хосты
   ansible all -i clusters/linstor-52.ini -m copy -a "src=linbit-manage-node.py dest=/root/linbit-manage-node.py mode=0640" --become
   ```

## Быстрая проверка перед запуском

```bash
# 1. Проверить подключение
ansible all -i clusters/linstor-52.ini -m ping

# 2. Проверить sudo
ansible all -i clusters/linstor-52.ini -m command -a "sudo -n true" --become

# 3. Проверить создание временной директории
ansible all -i clusters/linstor-52.ini -m file -a "path=/tmp/.ansible-tmp state=directory mode=1777" --become

# 4. Проверить доступность URL
curl -I https://my.linbit.com/linbit-manage-node.py
```

## Рекомендуемая конфигурация ansible.cfg

```ini
[defaults]
roles_path = ./roles
inventory  = ./hosts.ini
remote_user = root
remote_tmp = /tmp/.ansible-${USER}/tmp
local_tmp  = $HOME/.ansible/tmp
pipelining = True
become = True
host_key_checking = False
deprecation_warnings = False
interpreter_python = auto_silent
callback_whitelist = profile_tasks
timeout = 60
```

