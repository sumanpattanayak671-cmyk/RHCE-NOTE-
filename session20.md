# Session 20 — Ansible Tags

## Topic
Using tags in an Ansible playbook to run selected tasks instead of executing every task.

## Lab overview

- **Control node:** `ansible-server`
- **Managed nodes:** `node1`, `node2`
- **Inventory group:** `suman`
- **Web server:** Apache HTTP Server (`httpd`)
- **Website source file:** `/tmp/index.html`
- **Web root on managed nodes:** `/var/www/html/index.html`

The HTML file in this lab is a responsive **Linux Learning Hub** website with sections for RHEL, Ansible, cloud/virtualization, certifications, Linux commands, and training links.

## 1. Playbook without tags — `tag1.yml`

This playbook installs HTTPD, enables and starts the service, copies the website, opens HTTP in firewalld, and restarts HTTPD.

```yaml
---
- name: Configure HTTPD web server
  hosts: suman
  become: true

  tasks:
    - name: Install HTTPD
      yum:
        name: httpd
        state: present

    - name: Start and enable HTTPD
      service:
        name: httpd
        state: started
        enabled: true

    - name: Copy website HTML file
      copy:
        src: /tmp/index.html
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: '0644'

    - name: Allow HTTP through firewall
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true

    - name: Restart HTTPD
      service:
        name: httpd
        state: restarted
```

Run the complete playbook:

```bash
ansible-playbook tag1.yml -v
```

### Result recorded in the lab

The playbook completed on both `node1` and `node2` with no failed or unreachable hosts. HTTPD was installed, the HTML file was copied, the HTTP firewall service was enabled, and HTTPD was running.

## 2. Add tags — `tag2.yml`

A tag is a label assigned to a play or task. Tags allow selected tasks to be run with `--tags`.

```yaml
---
- name: Configure HTTPD web server
  hosts: suman
  become: true

  tasks:
    - name: Install HTTPD
      yum:
        name: httpd
        state: present
      tags:
        - install

    - name: Start and enable HTTPD
      service:
        name: httpd
        state: started
        enabled: true
      tags:
        - start

    - name: Copy website HTML file
      copy:
        src: /tmp/index.html
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: '0644'
      tags:
        - copy

    - name: Allow HTTP through firewall
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
      tags:
        - firewall

    - name: Restart HTTPD
      service:
        name: httpd
        state: restarted
      tags:
        - restart
```

## 3. Run selected tasks

### Run only the installation task

```bash
ansible-playbook tag2.yml --tags=install
```

Only the task labelled `install` runs (in addition to normal fact gathering, unless disabled).

### Run several selected tasks

```bash
ansible-playbook tag2.yml --tags="install,start,copy,firewall"
```

This runs the tasks labelled `install`, `start`, `copy`, and `firewall`. The `restart` task is not selected.

### Run the restart task only

```bash
ansible-playbook tag2.yml --tags=restart
```

### List available tags

```bash
ansible-playbook tag2.yml --list-tags
```

### List tasks and their tags

```bash
ansible-playbook tag2.yml --list-tasks
```

## 4. Tags used in this playbook

| Tag | Task |
|---|---|
| `install` | Install the HTTPD package |
| `start` | Start and enable HTTPD |
| `copy` | Copy the website HTML file |
| `firewall` | Allow HTTP through firewalld |
| `restart` | Restart HTTPD |

## 5. Important points

- Tags are written with the `tags:` keyword.
- A task can have one tag or multiple tags.
- `--tags=install` selects tasks with the `install` tag.
- Multiple tags can be supplied as a comma-separated list.
- Tasks without a matching tag are normally skipped when specific tags are requested.
- Ansible fact gathering may still run even when only one task tag is selected.
- `--list-tags` displays tags; `--list-tasks` displays the tasks that would be considered.
- The `yum` task is used in this lab to manage the HTTPD package on the RHEL nodes.
- The `firewalld` task enables the `http` service immediately and permanently.

## 6. Verification commands

Run these commands on the managed nodes, or use Ansible ad-hoc commands from the control node:

```bash
ansible suman -b -m command -a "systemctl is-active httpd"
ansible suman -b -m command -a "systemctl is-enabled httpd"
ansible suman -b -m command -a "ls -l /var/www/html/index.html"
ansible suman -b -m command -a "firewall-cmd --list-services"
```

To check the web page locally on a managed node:

```bash
curl http://localhost
```

If accessing it from another machine, use the managed node's IP address:

```text
http://<managed-node-IP>/
```

Ensure network connectivity and firewall rules permit HTTP traffic.

## Summary

This session demonstrates how to use Ansible tags to control which parts of a web-server configuration playbook execute. The example assigns separate tags to package installation, service startup, website deployment, firewall configuration, and service restart.
