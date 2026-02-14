# Version definitions in the project

This document lists where package, OS, and dependency versions are defined.

## 1. Operating system versions

### README.md
**File:** `README.md`  
**Content:** Target systems are RHEL 7/8/9 or Ubuntu 22.04, 24.04, 25.04+ (or compatible variants).

### OS version check
**File:** `roles/commons/os-checker/tasks/main.yaml`  
**Description:** Reads OS version from `/etc/os-release`; used for OS family (RedHat/Debian), not for hard version limits.

```yaml
- name: get os_version from /etc/os-release
  when: ansible_os_family is not defined
  raw: "grep '^VERSION_ID=' /etc/os-release | sed s'/VERSION_ID=//'"
  register: os_version
  changed_when: False
```

## 2. Ansible version

**File:** `README.md`  
**Content:** Deployment environment must have Ansible `2.9.0+` and `python-netaddr`.

## 3. LINSTOR package versions

### Important: package versions are not pinned

Packages are installed **without version pins**, i.e. latest available from LINBIT repositories.

### Package list for RHEL/CentOS

**File:** `roles/linstor/controller/meta/main.yaml`, `roles/linstor/satellite/meta/main.yaml`  
**Packages:** `lb_rpm_pkgs`: kmod-drbd, drbd, linstor-controller, linstor-satellite, linstor-client, python-linstor

### Package list for Ubuntu/Debian

**File:** `roles/linstor/controller/meta/main.yaml`, `roles/linstor/satellite/meta/main.yaml`  
**Packages:** `lb_deb_pkgs`: drbd-dkms, drbd-utils, linstor-controller, linstor-satellite, linstor-client, python-linstor

### Installation

**File:** `roles/commons/pre-install/tasks/pkg.yaml`  
`yum` / `apt` are used without version; they install the latest from the repository.

## 4. linbit-manage-node.py version

**File:** `roles/commons/pre-install/tasks/pkg.yaml`  
Script is fetched with `force: yes`, so the latest version is downloaded on every run.

## 5. Pinning package versions

To pin versions, change `roles/commons/pre-install/tasks/pkg.yaml` and define versioned lists in `group_vars/all.yaml` or role meta (e.g. `lb_rpm_pkgs_with_versions`, `lb_deb_pkgs_with_versions`).

## 6. Summary table

| Component        | File        | Current value              | Status   |
|-----------------|-------------|----------------------------|----------|
| Ubuntu versions | README.md   | 22.04, 24.04, 25.04+       | Updated  |
| RHEL versions   | README.md   | 7/8/9                      | Current  |
| Ansible version | README.md   | 2.9.0+                     | Updated  |
| LINSTOR packages| meta/main.yaml | No version (latest)     | Verify   |
| linbit-manage-node.py | pkg.yaml | Always latest        | Current  |

## 7. Recommendations

1. **Production:** Consider pinning LINSTOR package versions for stability.
2. **Development:** Keep using latest versions for new features and fixes.
3. **Monitoring:** Periodically check LINBIT repos for new Ubuntu releases.
4. **Testing:** Test the playbook in a lab before upgrading OS versions.
