# Ansible Cheat Sheet

> Configuration management and automation with Ansible — ad-hoc commands, playbooks, inventories, modules, roles, and vault.

---

## Table of Contents

- [Ad-Hoc Commands](#1-ad-hoc-commands)
- [Inventory](#2-inventory)
- [Playbook Structure](#3-playbook-structure)
- [Common Modules](#4-common-modules)
- [Variables & Facts](#5-variables--facts)
- [Conditionals, Loops & Handlers](#6-conditionals-loops--handlers)
- [Templates & Files](#7-templates--files)
- [Roles & Galaxy](#8-roles--galaxy)
- [Ansible Vault](#9-ansible-vault)
- [ansible.cfg & CLI Options](#10-ansiblecfg--cli-options)
- [Useful Patterns](#11-useful-patterns)

---

## 1. Ad-Hoc Commands

```
ansible all -m ping                              # Connectivity check
ansible web -m command -a "uptime"
ansible all -a "df -h"                           # Default module = command
ansible web -m shell -a "cat /etc/os-release | grep PRETTY"
ansible web -m apt -a "name=nginx state=present" -b          # -b = become sudo
ansible web -m service -a "name=nginx state=restarted" -b
ansible db -m copy -a "src=./my.cnf dest=/etc/my.cnf" -b
ansible all -m setup                             # Gather all facts
ansible web --list-hosts                         # Show matched hosts
ansible all -m reboot -b                         # Reboot everything
ansible web -m user -a "name=deploy shell=/bin/bash" -b
```

## 2. Inventory

**INI format (`inventory.ini`):**
```ini
[web]
web1.example.com
web2.example.com ansible_user=ubuntu ansible_port=2222

[db]
db1.example.com ansible_host=10.0.1.10

[all:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/deploy.pem
ansible_python_interpreter=/usr/bin/python3

[prod:children]
web
db
```

**YAML format (`inventory.yml`):**
```yaml
all:
  children:
    web:
      hosts:
        web1.example.com:
          ansible_user: ubuntu
        web2.example.com:
    db:
      hosts:
        db1.example.com:
          ansible_host: 10.0.1.10
  vars:
    ansible_user: admin
    env: production
```

**Dynamic inventory (AWS EC2 example):**
```
ansible-inventory --list -i aws_ec2.yml
ansible-playbook site.yml -i aws_ec2.yml
```

```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
keyed_groups:
  - key: tags.Role
    prefix: tag_role
filters:
  instance-state-name: running
```

## 3. Playbook Structure

```yaml
---
- name: Configure web servers
  hosts: web
  become: true                       # sudo
  gather_facts: true

  vars:
    app_version: "2.1.0"
    http_port: 80

  vars_files:
    - secrets.yml
    - vars/{{ env }}.yml            # environment-specific vars

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Start and enable nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        mode: "0644"
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

**Run playbooks:**
```
ansible-playbook site.yml
ansible-playbook site.yml -i inventory.ini
ansible-playbook site.yml -l web1.example.com         # --limit subset
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --skip-tags slow
ansible-playbook site.yml --start-at-task "Deploy config"
ansible-playbook site.yml --step                      # Confirm each task
ansible-playbook site.yml --syntax-check              # Validate only
ansible-playbook site.yml --check                     # Dry run (no changes)
ansible-playbook site.yml --check --diff              # Dry run + show diffs
ansible-playbook site.yml -e "env=prod app_version=2.2.0"   # Extra vars (highest priority)
ansible-playbook site.yml --vault-password-file .vault_pass
```

## 4. Common Modules

| Module | Purpose | Example |
| --- | --- | --- |
| `apt` / `dnf` / `yum` | Packages | `apt: name=nginx state=present update_cache=yes` |
| `copy` | Copy files | `copy: src=a.conf dest=/etc/a.conf mode=0644` |
| `template` | Jinja2 templates | `template: src=nginx.j2 dest=/etc/nginx/nginx.conf` |
| `file` | Files/dirs/links/permissions | `file: path=/opt/app state=directory mode=0755` |
| `service` | Manage services | `service: name=nginx state=started enabled=yes` |
| `systemd` | systemd units | `systemd: name=app state=restarted daemon_reload=yes` |
| `command` | Run commands (no shell) | `command: /usr/bin/make dbmigrate` |
| `shell` | Shell commands | `shell: psql < /tmp/backup.sql` |
| `user` | Manage users | `user: name=deploy shell=/bin/bash` |
| `group` | Manage groups | `group: name=developers state=present` |
| `git` | Clone repos | `git: repo=URL dest=/opt/app version=main` |
| `get_url` | Download files | `get_url: url=... dest=/tmp/file.tar.gz` |
| `unarchive` | Extract archives | `unarchive: src=x.tar.gz dest=/opt remote_src=yes` |
| `cron` | Cron jobs | `cron: name="backup" job="/opt/backup.sh" minute="0" hour="2"` |
| `mount` | Mounts | `mount: path=/data src=/dev/sdb1 fstype=ext4 state=mounted` |
| `uri` | HTTP requests | `uri: url=http://localhost/health status_code=200` |
| `lineinfile` | Edit lines in files | `lineinfile: path=/etc/ssh/sshd_config regexp=^PermitRootLogin line="PermitRootLogin no"` |
| `blockinfile` | Insert blocks | `blockinfile: path=... block="..."` |
| `replace` | Regex replace | `replace: path=... regexp=... replace=...` |
| `docker_container` | Manage containers | `docker_container: name=web image=nginx ports=["80:80"]` |
| `k8s` | Kubernetes objects | `k8s: state=present definition=...` |
| `debug` | Print messages | `debug: msg="IP is {{ ansible_default_ipv4.address }}"` |
| `set_fact` | Set host facts | `set_fact: db_host="{{ inventory_hostname }}"` |
| `wait_for` | Wait for ports/URLs | `wait_for: port=8080 delay=5 timeout=60` |
| `fail` | Fail playbook | `fail: msg="Required variable missing"` |

## 5. Variables & Facts

**Precedence (highest → lowest):** extra vars `-e` > task vars > block vars > role vars > playbook vars > inventory host_vars/group_vars > facts.

```yaml
# Group variables — group_vars/web.yml
nginx_workers: 4
app_env: production

# Host variables — host_vars/web1.yml
custom_setting: true
```

```yaml
tasks:
  - name: Use gathered facts
    ansible.builtin.debug:
      msg: "Hostname {{ ansible_hostname }} has {{ ansible_memtotal_mb }} MB RAM on {{ ansible_distribution }} {{ ansible_distribution_version }}"

  - name: Set a fact for later tasks
    ansible.builtin.set_fact:
      primary_ip: "{{ ansible_default_ipv4.address }}"

  - name: Debug with condition
    ansible.builtin.debug:
      msg: "This is Ubuntu"
    when: ansible_os_family == "Debian"
```

## 6. Conditionals, Loops & Handlers

```yaml
tasks:
  - name: Install apache on Debian family
    ansible.builtin.apt:
      name: apache2
      state: present
    when: ansible_os_family == "Debian"
    # when: (app_env == "prod") and (enable_ssl | bool)

  - name: Create app users
    ansible.builtin.user:
      name: "{{ item }}"
      state: present
    loop:
      - alice
      - bob
    # loop: "{{ app_users }}"

  - name: Create vhosts
    ansible.builtin.template:
      src: vhost.j2
      dest: "/etc/nginx/sites-available/{{ item.name }}"
    loop:
      - { name: app1, port: 8080 }
      - { name: app2, port: 8081 }
    loop_control:
      label: "{{ item.name }}"

  - name: Restart app (only when config changed)
    ansible.builtin.systemd:
      name: app
      state: restarted
```

**Handlers** run once at the end, only when notified:
```yaml
handlers:
  - name: Restart app
    ansible.builtin.systemd:
      name: app
      state: restarted
    listen: "restart app"        # multiple notifies can share a listen name
```

## 7. Templates & Files

```jinja2
{# nginx.conf.j2 #}
user www-data;
worker_processes {{ nginx_workers | default(2) }};

{% if enable_ssl | bool %}
listen 443 ssl;
ssl_certificate {{ ssl_cert_path }};
{% endif %}

{% for upstream in app_upstreams %}
upstream app{{ loop.index }} { server {{ upstream }}; }
{% endfor %}
```

## 8. Roles & Galaxy

```
ansible-galaxy init roles/nginx      # Scaffold role directory
ansible-galaxy install geerlingguy.nginx
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.docker
```

**requirements.yml:**
```yaml
roles:
  - name: geerlingguy.nginx
    version: 3.1.4
collections:
  - name: community.docker
    version: ">=3.0.0"
```

**Role usage:**
```yaml
- hosts: web
  roles:
    - { role: nginx, nginx_workers: 8, tags: [web] }
    - app
```

**Standard role layout:**
```
roles/nginx/
├── defaults/main.yml      # Lowest precedence defaults
├── handlers/main.yml
├── meta/main.yml          # Dependencies, galaxy info
├── tasks/main.yml
├── templates/nginx.conf.j2
├── files/
└── vars/main.yml          # High precedence
```

## 9. Ansible Vault

```
ansible-vault create secrets.yml
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault view secrets.yml              # Read without decrypting on disk
ansible-vault edit secrets.yml
ansible-vault rekey secrets.yml             # Change password
ansible-vault encrypt_string 'SuperSecret123' --name db_password

ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass
ansible-playbook site.yml --vault-id prod@prompt
```

```yaml
# Playbook referencing vaulted vars
vars_files:
  - secrets.yml
tasks:
  - name: Use secret
    ansible.builtin.debug:
      msg: "Password length is {{ db_password | length }}"
    no_log: true                     # Don't leak secrets in output
```

## 10. ansible.cfg & CLI Options

```ini
# ansible.cfg
[defaults]
inventory = ./inventory.ini
remote_user = ubuntu
private_key_file = ~/.ssh/deploy.pem
host_key_checking = False
retry_files_enabled = False
roles_path = ./roles:/etc/ansible/roles
stdout_callback = yaml
interpreter_python = auto_silent
forks = 20

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

| CLI option | Description |
| --- | --- |
| `-i` | Inventory file |
| `-u` | Remote user |
| `-b` / `--become` | Privilege escalation |
| `-K` / `--ask-become-pass` | Prompt for sudo password |
| `-k` / `--ask-pass` | Prompt for SSH password |
| `-e` / `--extra-vars` | Extra variables |
| `-l` / `--limit` | Limit to hosts/pattern |
| `-t` / `--tags` | Run tagged tasks |
| `--check` / `--diff` | Dry run / show diffs |
| `--syntax-check` | Validate playbook |
| `-v / -vvv` | Verbose / debug output |
| `-f` | Parallel forks (default 5) |

## 11. Useful Patterns

```yaml
# Rolling deployment with serial + health check
- name: Rolling deploy
  hosts: web
  serial: 1                              # one host at a time
  tasks:
    - name: Pull new image
      ansible.builtin.command: docker pull myapp:{{ version }}
    - name: Restart container
      ansible.builtin.systemd:
        name: myapp
        state: restarted
    - name: Wait for health endpoint
      ansible.builtin.uri:
        url: http://localhost:8080/health
        status_code: 200
      retries: 12
      delay: 5

# Delegate task to localhost / another host
- name: Add node to load balancer
  ansible.builtin.uri:
    url: "http://lb/admin/enable?node={{ inventory_hostname }}"
  delegate_to: localhost

# Run once (e.g., DB migration)
- name: Run migrations
  ansible.builtin.command: /opt/app/migrate
  run_once: true
```

```bash
# Fast facts gathering for an inventory
ansible all -m setup --tree ./facts

# Ad-hoc file distribution
ansible web -m copy -a "src=./app.jar dest=/opt/app/app.jar backup=yes" -b

# Rolling restart
ansible web -m shell -a "systemctl restart myapp" -f 1 --serial 1
```

---
