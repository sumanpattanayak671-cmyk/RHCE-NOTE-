# SESSION22P2 — Ansible Roles: Practical Apache HTTPD Lab

> **Topic:** Configure Apache HTTPD with an Ansible role  
> **Environment:** Ansible control node and managed nodes `node1` / `node2`  
> **Main role:** `/etc/ansible/roles/apache`

## Contents

1. Deploy HTTPD with a playbook and Jinja2 template
2. Create the Apache role
3. Split role tasks into separate files
4. Configure the template and handler
5. Call the role from a playbook
6. Understand the sample execution output
7. Troubleshooting and important notes
8. Original session note

---

## 1. Deploy HTTPD with a playbook and Jinja2 template

The initial example uses a Jinja2 template named `index.j2`. The variable `{{ ansible_hostname }}` is an Ansible fact, so each managed server can display its own hostname in the generated web page.

### `index.j2`

```html
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>
```

### `apache.yml`

The playbook installs HTTPD, starts and enables the service, deploys the page, and allows HTTP traffic through `firewalld`.

```yaml
---
- name: Configure HTTPD and deploy a host-specific page
  hosts: suman
  become: true

  tasks:
    - name: Install HTTPD
      ansible.builtin.dnf:
        name: httpd
        state: latest

    - name: Start and enable HTTPD
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

    - name: Render the web page
      ansible.builtin.template:
        src: /home/ansible/index.j2
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: "0644"

    - name: Allow HTTP through firewalld
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true
```

> **Note:** The source file includes an extra task that restarts HTTPD unconditionally. In a role, a handler notified by a configuration change is generally preferable because it avoids unnecessary restarts.

## 2. Create the Apache role

From the Ansible control node, initialize the role:

```bash
cd /etc/ansible/roles/
sudo ansible-galaxy init /etc/ansible/roles/apache --offline
```

The role structure shown in the session is:

```text
apache/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
├── vars/
│   └── main.yml
└── README.md
```

The exact generated structure can vary by Ansible version. The key directories used in this exercise are `tasks/`, `templates/`, and `handlers/`.

## 3. Split role tasks into separate files

The session separates the Apache setup into three task files. This makes the role easier to read and maintain.

### `tasks/install.yml`

```yaml
---
- name: Install HTTPD
  ansible.builtin.dnf:
    name: httpd
    state: latest
```

### `tasks/service.yml`

```yaml
---
- name: Start and enable HTTPD
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

### `tasks/config.yml`

```yaml
---
- name: Render the web page
  ansible.builtin.template:
    src: index1.j2
    dest: /var/www/html/index.html
    owner: root
    group: root
    mode: "0644"

- name: Allow HTTP through firewalld
  ansible.posix.firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true
```

The template is stored in the role's `templates/` directory, so `src: index1.j2` can refer to it by its filename.

### `tasks/main.yml`

```yaml
---
# Main task entry point for the apache role
- ansible.builtin.import_tasks: install.yml
- ansible.builtin.import_tasks: service.yml
- ansible.builtin.import_tasks: config.yml
```

**Important:** Use `import_tasks` (plural), not `import_task`. The singular form is not a valid Ansible task-import module.

## 4. Configure the template and handler

### `templates/index1.j2`

```html
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>
```

During execution, Ansible renders `{{ ansible_hostname }}` using facts gathered from each target host. For example, the page on `node1` should show `node1` if that is its reported hostname.

### `handlers/main.yml`

The source session defines a handler that restarts HTTPD:

```yaml
---
- name: Restarting apachi HTTPD Service
  ansible.builtin.service:
    name: httpd
    state: restarted
```

To make this handler run only when the page or configuration changes, add a notification to the relevant task:

```yaml
- name: Render the web page
  ansible.builtin.template:
    src: index1.j2
    dest: /var/www/html/index.html
    owner: root
    group: root
    mode: "0644"
  notify: Restarting apachi HTTPD Service
```

A handler does not run merely because it exists. A task must notify it, and normally it runs at the end of the play if the notifying task reports a change.

## 5. Add role metadata and call the role

### `meta/main.yml`

The session includes this role metadata:

```yaml
---
galaxy_info:
  author: SUMAN
  description: Apache httpd server config
  company: student at BWU
```

### `/etc/ansible/setup.yml`

```yaml
---
- name: Apache Server Setup Playbook through roles
  hosts: suman
  become: true

  roles:
    - apache
```

The inventory group `suman` should contain the managed hosts `node1` and `node2`, as in the sample run.

Run the playbook:

```bash
ansible-playbook setup.yml
```

## 6. Understand the sample execution output

The supplied session shows successful execution on both managed nodes:

```text
PLAY [apache Server Setup Playbook through roles]

TASK [Gathering Facts]
ok: [node2]
ok: [node1]

TASK [apache : Install HTTPD]
ok: [node2]
ok: [node1]

TASK [apache : Start and enable HTTPD]
ok: [node1]
ok: [node2]

TASK [apache : Render the web page]
ok: [node1]
ok: [node2]

TASK [apache : Allow HTTP through firewalld]
ok: [node1]
ok: [node2]

PLAY RECAP
node1 : ok=5 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
node2 : ok=5 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

- `ok=5` means five tasks completed successfully for each host, including fact gathering.
- `changed=0` means Ansible did not need to make changes during this run.
- `unreachable=0` means Ansible could contact the host.
- `failed=0` means no task failed.

A later run may show `changed` tasks if a package, file, service, or firewall setting needs updating.

## 7. Troubleshooting and important notes

### Check the role syntax

```bash
sudo ansible-playbook /etc/ansible/setup.yml --syntax-check
```

### List the tasks Ansible detects

```bash
sudo ansible-playbook /etc/ansible/setup.yml --list-tasks
```

### Check HTTPD status on the managed hosts

```bash
ansible suman -b -m ansible.builtin.command -a "systemctl is-active httpd"
```

### Test the generated page locally on a managed host

```bash
ansible suman -b -m ansible.builtin.command -a "cat /var/www/html/index.html"
```

### Common issues

- **Role not found:** Ensure `/etc/ansible/roles/apache/` exists and the configured roles path includes `/etc/ansible/roles`.
- **Template not found:** Keep `index1.j2` under `apache/templates/` and refer to it as `src: index1.j2`.
- **Firewall module not found:** Install the collection with `ansible-galaxy collection install ansible.posix` if needed.
- **Handler never runs:** Add `notify:` to a task and make sure that task actually changes something.
- **Hostname variable is blank or undefined:** Ensure fact gathering is enabled; it is enabled by default unless disabled in the play.
- **Unnecessary package upgrades:** `state: latest` may upgrade HTTPD. Use `state: present` if the goal is only to ensure it is installed.

---

## 8. Original session note

The full source note is preserved below for reference.

---

SESSION P2

-------------------------------

ansible@ansible-server:~$ cat index.j2
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>
ansible@ansible-server:~$ cat apache.yml
---
- name: Configure HTTPD and deploy a host-specific page
  hosts: suman
  become: true
  tasks:
    - name: Install HTTPD
      ansible.builtin.dnf:
        name: httpd
        state: latest
    - name: Start and enable HTTPD
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
    - name: Render the web page
      ansible.builtin.template:
        src: /home/ansible/index.j2
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: "0644"
    - name: Allow HTTP through firewalld
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true

    - name: Restarting apachi HTTPD Service
      service:
        name: httpd
        state: restarted
...
ansible@ansible-server:~$

ansible@ansible-server:~$ cd /etc/ansible/roles/
ansible@ansible-server:/etc/ansible/roles$ sudo ansible-galaxy init /etc/ansible/roles/apache --offline
- Role /etc/ansible/roles/apache was created successfully
ansible@ansible-server:/etc/ansible/roles$ 

ansible@ansible-server:/etc/ansible/roles$ tree apache/
apache/
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml

9 directories, 8 files

--------------------------------------------------------

ansible@ansible-server:/etc/ansible/roles/apache/tasks$ sudo cat install.yml
- name: Install HTTPD
  dnf:
    name: httpd
    state: latest
...
ansible@ansible-server:/etc/ansible/roles/apache/tasks$ sudo cat service.yml
 - name: Start and enable HTTPD
   service:
      name: httpd
      state: started
      enabled: true
...

ansible@ansible-server:/etc/ansible/roles/apache/tasks$ sudo cat config.yml
---
- name: Render the web page
  template:
      src: templates/index1.j2
      dest: /var/www/html/index.html
      owner: root
      group: root
      mode: "0644"
- name: Allow HTTP through firewalld
  firewalld:
      service: http
      permanent: true
      state: enabled
      immediate: true
...

---------------------------------------------------
ansible@ansible-server:/etc/ansible$ sudo cat /etc/ansible/roles/apache/tasks/main.yml
---
# tasks file for /etc/ansible/roles/apache
- import_tasks: install.yml
- import_tasks: service.yml
- import_tasks: config.yml

-----------------------------------
nsible@ansible-server:/etc/ansible/roles/apache/tasks$ cd /etc/ansible/roles/apache/templates/
ansible@ansible-server:/etc/ansible/roles/apache/templates$ cat index1.j2
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>

------------------------------------------

ansible@ansible-server:/etc/ansible/roles/apache/templates$ cd /etc/ansible/roles/apache/handlers/
ansible@ansible-server:/etc/ansible/roles/apache/handlers$ cat main.yml
---
# handlers file for /etc/ansible/roles/apache
- name: Restarting apachi HTTPD Service
  service:
      name: httpd
      state: restarted

------------------------------------------------------

ansible@ansible-server:/etc/ansible/roles/apache/handlers$ cd ..
ansible@ansible-server:/etc/ansible/roles/apache$ cd meta/
ansible@ansible-server:/etc/ansible/roles/apache/meta$

ansible@ansible-server:/etc/ansible$ sudo cat /etc/ansible/roles/apache/meta/main.yml
galaxy_info:
  author: SUMAN
  description: Apache httpd server config
  company: student at BWU

---------------------------------------------

ansible@ansible-server:/etc/ansible$ cat setup.yml
---
- name: apache Server Setup Playbook through roles
  hosts: suman
  become: true
  roles:
    - apache
...

------------------------------------------------------ansible@ansible-server:/etc/ansible$ ansible-playbook setup.yml

PLAY [apache Server Setup Playbook through roles] ****************************************************

TASK [Gathering Facts] *******************************************************************************
ok: [node2]
ok: [node1]

TASK [apache : Install HTTPD] ************************************************************************
ok: [node2]
ok: [node1]

TASK [apache : Start and enable HTTPD] ***************************************************************
ok: [node1]
ok: [node2]

TASK [apache : Render the web page] ******************************************************************
ok: [node1]
ok: [node2]

TASK [apache : Allow HTTP through firewalld] *********************************************************
ok: [node1]
ok: [node2]

PLAY RECAP *******************************************************************************************
node1                      : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node2                      : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
