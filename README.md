# Ansible Automation for MariaDB HA Cluster

This repository contains the Infrastructure as Code (IaC) playbooks used to automate, secure, and manage a 3-node MariaDB High-Availability (HA) Master-Slave cluster.

## 🚀 Project Overview

Managing database clusters manually leads to configuration drift. This project utilizes Ansible to enforce a single source of truth across all nodes, ensuring that security policies (such as read-only locks and restricted service users) are applied uniformly and idempotently without manual SSH intervention.

---

## 🗺️ 1. Inventory & Architecture

The cluster is managed via a custom SSH port (`2229`) for security hardening.

**File: `hosts.ini`**

```ini
[db_servers]
node2 ansible_host=192.30
node3 ansible_host=192.30

[db_servers:vars]
ansible_user=root
ansible_port=2229
```

---

## 🛠️ 2. Core Playbooks

### A. The "Permanent Lock" (Configuration Management)

This playbook ensures that the replicas are strictly locked to read-only mode. It uses the `lineinfile` module to permanently edit the `50-server.cnf` file so the setting survives a server reboot.

**File: `lock_config.yml`**

```yaml
---
- name: Permanently Lock Slaves to Read-Only
  hosts: db_servers
  become: yes
  tasks:
    - name: Ensure read_only=1 is in 50-server.cnf
      ansible.builtin.lineinfile:
        path: /etc/mysql/mariadb.conf.d/50-server.cnf
        regexp: '^read_only'
        line: 'read_only = 1'
        insertafter: '\[mysqld\]'
      register: config_file

    - name: Restart MariaDB if config changed
      ansible.builtin.service:
        name: mariadb
        state: restarted
      when: config_file.changed
```

### B. Secure User Provisioning

The standard `read_only` flag does not block the root user. This playbook installs the required dependencies and creates a restricted application user with only `SELECT` privileges.

**File: `create_readonly_user.yml`**

```yaml
---
- name: Configure Read-Only Database Users
  hosts: db_servers
  become: yes
  vars_files:
    - secrets.yml
  tasks:
    - name: Install Python MySQL dependencies
      ansible.builtin.apt:
        name: python3-pymysql
        state: present

    - name: Create app_readonly user with Select permissions
      community.mysql.mysql_user:
        name: app_readonly
        password: "{{ db_password }}"
        priv: "*.*:SELECT"
        host: "%"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock
```

---

## 🔐 3. Secret Management (Ansible Vault)

To prevent hardcoding sensitive database passwords inside the GitHub repository, we used Ansible Vault to encrypt our variables.

**File: `secrets.yml`** *(Encrypted in repo)*

```yaml
db_password: "your_password_here"
```

> Encrypt with: `ansible-vault encrypt secrets.yml`

---

## 🚀 4. Execution Commands

Bypass strict host key checking *(initial run only)*:

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```

Test connectivity:

```bash
ansible all -i hosts.ini -m ping
```

Execute the configuration lock:

```bash
ansible-playbook -i hosts.ini lock_config.yml
```

Run playbooks with encrypted variables:

```bash
ansible-playbook -i hosts.ini create_readonly_user.yml --ask-vault-pass
```

---

## 📦 5. Version Control Security

> **CRITICAL:** The `.gitignore` file is strictly configured to prevent pushing live IPs and unencrypted passwords to the public repository.

**File: `.gitignore`**

```
hosts.ini
secrets.yml
*.log
.vault_pass
```
