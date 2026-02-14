# Installation troubleshooting

## Timeout when creating temporary file

### Error
```
[ERROR]: Task failed: Module failed: failed to create temporary content file: The read operation timed out
```

### Causes
1. **remote_tmp** — wrong path or permissions
2. **pipelining** — conflict with become
3. **Slow connection** — timeout when transferring files

### Fixes

#### Fix 1: Set remote_tmp in ansible.cfg

In `ansible.cfg`:
```ini
remote_tmp = /tmp/.ansible-${USER}/tmp
```

Or use an absolute path:
```ini
remote_tmp = /tmp/ansible-tmp
```

#### Fix 2: Disable pipelining (temporary)

If the issue persists:
```ini
pipelining = False
```

#### Fix 3: Increase timeout

In `ansible.cfg`:
```ini
timeout = 60
```

In the `get_url` task add:
```yaml
timeout: 60
```

#### Fix 4: Create remote_tmp on hosts manually

On each host:
```bash
sudo mkdir -p /tmp/.ansible-tmp
sudo chmod 1777 /tmp/.ansible-tmp
```

Or via Ansible:
```bash
ansible all -i clusters/linstor.ini -m file -a "path=/tmp/.ansible-tmp state=directory mode=1777" --become
```

## remote_tmp warning

### Error
```
[WARNING]: Module remote_tmp /root/.ansible/tmp did not exist and was created with a mode of 0700, this may cause issues when running as another user.
```

### Fix

Use a path that works for all users:
```ini
remote_tmp = /tmp/.ansible-${USER}/tmp
```

Or create the directory beforehand with suitable permissions.

## Non-root user with become

If using a non-root user with `become: yes`:

1. **Ensure the user can use sudo:**
   ```bash
   ssh <user>@<host> "sudo -n true"
   ```

2. **Ensure sudo does not require a password:**
   ```bash
   ssh <user>@<host> "echo '<user> ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/<user>"
   ```
   Replace `<user>` and `<host>` with your user and host.

3. **Or use root:**
   In `group_vars/all.yaml`:
   ```yaml
   ansible_user: root
   ```

## Timeout when fetching linbit-manage-node.py

### Fix

1. **Increase timeout in the task:**
   ```yaml
   - name: fetch the latest linbit-manage-node.py
     get_url:
       url: "https://my.linbit.com/linbit-manage-node.py"
       dest: "/root/linbit-manage-node.py"
       mode: "0640"
       force: "yes"
       timeout: 60
   ```

2. **Check URL availability:**
   ```bash
   curl -I https://my.linbit.com/linbit-manage-node.py
   ```

3. **Download manually and copy:**
   ```bash
   # On your machine
   curl -o linbit-manage-node.py https://my.linbit.com/linbit-manage-node.py

   # Copy to hosts
   ansible all -i clusters/linstor.ini -m copy -a "src=linbit-manage-node.py dest=/root/linbit-manage-node.py mode=0640" --become
   ```

## Pre-run checks

```bash
# 1. Connectivity
ansible all -i clusters/linstor.ini -m ping

# 2. Sudo
ansible all -i clusters/linstor.ini -m command -a "sudo -n true" --become

# 3. Temp directory
ansible all -i clusters/linstor.ini -m file -a "path=/tmp/.ansible-tmp state=directory mode=1777" --become

# 4. URL
curl -I https://my.linbit.com/linbit-manage-node.py
```

## Suggested ansible.cfg

```ini
[defaults]
roles_path = ./roles
inventory  = ./clusters/linstor.ini
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
