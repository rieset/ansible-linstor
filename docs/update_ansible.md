# Updating Ansible

## Current version

You have: **Ansible [core 2.17.14]** (or similar)  
Install path: e.g. `/Users/<user>/Library/Python/3.12/bin/ansible`  
This indicates a **pip** installation.

## How to update

### Option 1: Update via pip3 (recommended)

```bash
# Upgrade ansible-core to latest
python3 -m pip install --upgrade ansible-core

# Or if using the legacy 'ansible' package
python3 -m pip install --upgrade ansible
```

### Option 2: Pin to a specific version

```bash
# Upgrade to latest 2.17.x
python3 -m pip install --upgrade "ansible-core>=2.17.0"

# Or exact version
python3 -m pip install --upgrade ansible-core==2.17.15
```

### Option 3: Reinstall (if something is broken)

```bash
python3 -m pip uninstall ansible-core
python3 -m pip install ansible-core
```

## After updating

```bash
ansible --version
ansible --help
```

## Dependencies

Install `netaddr` (required by the playbook):

```bash
python3 -m pip install netaddr
```

## Other install methods

### Homebrew (macOS)

```bash
brew install ansible
brew upgrade ansible
```

### pipx (isolated)

```bash
python3 -m pip install --user pipx
pipx install ansible-core
pipx upgrade ansible-core
```

## Compatibility

The playbook requires Ansible `2.9.0+`. Your version (e.g. 2.20.2) is compatible.

## Troubleshooting

### Dependency conflict (ansible vs ansible-core)

**Error:**
```
ERROR: ansible 10.7.0 requires ansible-core~=2.17.7, but you have ansible-core 2.20.2 which is incompatible.
```

**Fix:** Remove the legacy `ansible` meta-package and keep only `ansible-core`:

```bash
python3 -m pip uninstall -y ansible
python3 -m pip list | grep -i ansible
ansible --version
```

Use only `ansible-core`; the `ansible` meta-package is no longer needed.

### "command not found" after update

Ensure the Python bin path is in `PATH`, e.g. add to `~/.zshrc`:
```bash
export PATH="$HOME/Library/Python/3.12/bin:$PATH"
```

### Permission issues

```bash
python3 -m pip install --user --upgrade ansible-core
# or fix cache ownership
sudo chown -R $(whoami) ~/Library/Caches/pip
```

### Full reinstall

```bash
python3 -m pip uninstall -y ansible ansible-core
python3 -m pip cache purge
python3 -m pip install --user ansible-core
python3 -m pip install --user netaddr
```

## Project requirements

Per README: minimum **Ansible 2.9.0+**, recommended latest stable (e.g. 2.20.2+).

## Quick upgrade command

```bash
python3 -m pip install --upgrade ansible-core && ansible --version
```
