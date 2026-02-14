# TODO and project goals

## Goals

### 1. Support new Ubuntu versions
- [ ] Add support for Ubuntu 24.04 (Noble Numbat)
- [ ] Add support for Ubuntu 25.04 (Plucky Pangolin)
- [ ] Update docs with supported versions
- [ ] Test installation on Ubuntu 24.04
- [ ] Test installation on Ubuntu 25.04

### 2. LINSTOR package versions
- [ ] Check current LINSTOR package versions in LINBIT repos
- [ ] Ensure latest packages are installed
- [ ] Add explicit version pinning if required
- [ ] Test with latest versions

### 3. Documentation
- [ ] Update README with current supported OS
- [ ] Update Ansible requirements (verify minimum version)
- [ ] Add testing notes for new Ubuntu versions

### 4. Testing and validation
- [ ] Test playbook on Ubuntu 24.04
- [ ] Test playbook on Ubuntu 25.04
- [ ] Verify all roles (controller, satellite, storage-pool)
- [ ] Verify storage pool creation
- [ ] Verify firewall rules

## Version and OS locations

1. **README.md** — Supported OS: Ubuntu 22.04, 24.04, 25.04+
2. **README.md** — Ansible: verify minimum version
3. **LINSTOR packages** (no version pin) — `roles/linstor/controller/meta/main.yaml`, `roles/linstor/satellite/meta/main.yaml`
4. **OS check** — `roles/commons/os-checker/tasks/main.yaml` (RedHat and Debian families)

## Action plan

1. Update README with supported Ubuntu versions and Ansible requirements.
2. Check LINBIT package availability for Ubuntu 24.04 and 25.04; verify `linbit-manage-node.py` with new versions.
3. Test in Ubuntu 24.04 and 25.04 environments; run playbook and validate.
4. If issues appear: adapt roles, firewall, and service configs.

## Notes

- LINSTOR packages are installed from LINBIT repo after registration via `linbit-manage-node.py`.
- Versions are not pinned; latest from repo are used.
- To pin versions, change `roles/commons/pre-install/tasks/pkg.yaml`.
