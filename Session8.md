# Session 8 — Ansible Playbooks, Multiple Plays, and Interactive Execution

## 1. Session overview

This session demonstrates how to write and run an Ansible YAML playbook to perform common administration tasks on managed nodes. The original notes use the inventory group `suman` (with `node1` and `node2`) and later show a second play for Ubuntu hosts.

The main tasks are:
1. Create `/tmp/testdir` on all hosts in the `suman` group.
2. Copy the control node's `/etc/fstab` file to `/tmp/testdir` on each managed node.
3. Install `httpd` and `wget`.
4. Start and enable the `httpd` service.
5. Run playbooks in check mode and step-by-step mode.
6. Put multiple plays for different operating systems in one YAML file.
7. Specify a remote connection user and privilege escalation.

## 2. YAML and playbook basics

An Ansible playbook is a YAML file that describes the hosts to target and the tasks to execute. YAML indentation is important: use spaces consistently, not tabs.

- `---` marks the beginning of a YAML document.
- `hosts` selects the inventory group or hosts targeted by a play.
- `gather_facts: false` skips automatic fact collection, which can make a simple play run faster.
- `tasks` contains the ordered list of tasks.
- `name` gives a readable description to a play or task.
- A module such as `file`, `copy`, `yum`, `apt`, or `service` performs the requested operation.

## 3. Basic playbook: `task.yml`

Create the file:

```bash
vim task.yml
```

A cleaned-up version that carries out the tasks in the notes is:

```yaml
---
- name: Configure web servers
  hosts: suman
  gather_facts: false
  become: true
  tasks:
    - name: Create /tmp/testdir
      ansible.builtin.file:
        path: /tmp/testdir
        state: directory

    - name: Copy the control node fstab file
      ansible.builtin.copy:
        src: /etc/fstab
        dest: /tmp/testdir/fstab
        mode: '0644'

    - name: Install httpd and wget
      ansible.builtin.yum:
        name:
          - httpd
          - wget
        state: present

    - name: Start and enable httpd
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
```

### Explanation of each task

**Task 1 — `file` module**

```yaml
- name: Create /tmp/testdir
  ansible.builtin.file:
    path: /tmp/testdir
    state: directory
```

- `path` is the directory to create.
- `state: directory` ensures that the directory exists. If it already exists, Ansible normally reports no change.
- This operation runs on every host matched by `hosts: suman`.

**Task 2 — `copy` module**

```yaml
- name: Copy the control node fstab file
  ansible.builtin.copy:
    src: /etc/fstab
    dest: /tmp/testdir/fstab
```

- `src` is read from the Ansible control node, where the playbook is launched.
- `dest` is the path on each managed node.
- Using `/tmp/testdir/fstab` makes the destination filename explicit. In the original notes, `dest: /tmp/testdir` is a directory destination, so Ansible copies the file into that directory while retaining the filename.
- The notes contain `/ete/fstab` in the task description, which appears to be a typo. The command uses `/etc/fstab`, the usual path.

**Task 3 — `yum` module**

```yaml
- name: Install httpd and wget
  ansible.builtin.yum:
    name:
      - httpd
      - wget
    state: present
```

- `name` lists the packages.
- `state: present` ensures the packages are installed.
- `yum` is commonly used on RHEL-family systems. On newer Ansible versions, `ansible.builtin.dnf` is also a suitable choice for modern RHEL systems.
- The playbook requires package repositories to be configured and reachable.

**Task 4 — `service` module**

```yaml
- name: Start and enable httpd
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

- `state: started` starts the service if it is not running.
- `enabled: true` configures it to start automatically at boot.
- The package name and service name are both `httpd` on RHEL-family systems.

> **Important:** Installing packages and managing system services generally require administrative privileges. The example includes `become: true`; alternatively, configure privilege escalation in `ansible.cfg` if that is your chosen lab method.

## 4. Check mode and verbose output

The original notes show:

```bash
ansible-playbook task.yml -C -v
```

- `-C` / `--check` asks Ansible to predict changes without applying most of them.
- `-v` enables verbose output.
- Check mode is a dry run, but not every module can predict changes perfectly. A task may be skipped or provide incomplete information in check mode.

To execute the playbook for real, use:

```bash
ansible-playbook task.yml
```

## 5. Understanding the play recap

The notes show a successful run with `node1` and `node2`:

```text
PLAY RECAP
node1 : ok=4 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
node2 : ok=4 changed=3 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Meaning:

- `ok=4`: four tasks completed successfully for that host.
- `changed=3`: three tasks made changes during that run.
- `unreachable=0`: Ansible could reach the host.
- `failed=0`: no task failed.
- `skipped=0`: no tasks were skipped.
- `rescued=0`: no failed task was handled by a rescue block.
- `ignored=0`: no failures were ignored.

The exact `changed` count can differ on later runs because Ansible modules are generally idempotent: once the desired state is already present, a repeated run often reports `ok` instead of `changed`.

## 6. Verify the HTTP service

The original notes use:

```bash
ansible suman -m command -a 'systemctl status httpd'
```

This runs the `systemctl status httpd` command on the hosts in `suman`. The recorded output indicates:

- `Loaded: ... enabled` — the service is configured to start at boot.
- `Active: active (running)` — the service is currently running.
- `Server configured, listening on: port 80` — Apache reports that it is listening on HTTP port 80.

The `command` module may report `CHANGED` because Ansible cannot generally determine whether an arbitrary command changed the system. For a cleaner read-only status check, use:

```bash
ansible suman -m command -a 'systemctl is-active httpd'
ansible suman -m command -a 'systemctl is-enabled httpd'
```

## 7. Interactive execution with `--step`

The notes show:

```bash
ansible-playbook task.yml --step
```

Step mode asks for confirmation before each task. The prompt offers these choices:

- `y` — run this task.
- `n` — skip this task.
- `c` — continue running the remaining tasks without asking for each one.

This is useful while learning a playbook or checking which task will run next. It is interactive and therefore less suitable for unattended automation.

## 8. A more readable playbook with task names

Task names make output easier to understand and troubleshoot. The notes call this a “better version” of `task.yml`. Here is a consistently indented version:

```yaml
---
- name: First playbook for RHEL hosts
  hosts: suman
  gather_facts: false
  become: true
  tasks:
    - name: Creating /tmp/testdir
      ansible.builtin.file:
        path: /tmp/testdir
        state: directory

    - name: Copy /etc/fstab to /tmp/testdir
      ansible.builtin.copy:
        src: /etc/fstab
        dest: /tmp/testdir/fstab

    - name: Installing httpd and wget
      ansible.builtin.yum:
        name:
          - httpd
          - wget
        state: present

    - name: Start and enable service
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
```

## 9. Multiple plays in one YAML file

An Ansible playbook can contain more than one play. Each play can target a different inventory group and perform operating-system-specific tasks.

The notes demonstrate:
- A first play targeting `suman` for RHEL-family nodes.
- A second play targeting `ubantu` for Ubuntu nodes.

Example, with corrected YAML indentation:

```yaml
---
- name: Configure RHEL hosts
  hosts: suman
  gather_facts: false
  become: true
  tasks:
    - name: Create /tmp/testdir
      ansible.builtin.file:
        path: /tmp/testdir
        state: directory

    - name: Copy /etc/fstab to /tmp/testdir
      ansible.builtin.copy:
        src: /etc/fstab
        dest: /tmp/testdir/fstab

    - name: Install httpd and wget
      ansible.builtin.yum:
        name:
          - httpd
          - wget
        state: present

    - name: Start and enable httpd
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

- name: Deploy Apache on Ubuntu hosts
  hosts: ubantu
  gather_facts: false
  become: true
  tasks:
    - name: Install Apache package
      ansible.builtin.apt:
        name: apache2
        state: present
        update_cache: true

    - name: Start and enable Apache
      ansible.builtin.service:
        name: apache2
        state: started
        enabled: true
```

**Important notes:**
- The inventory group must match the name used in `hosts:` exactly. If the group is named `ubuntu` in your inventory, change `hosts: ubantu` to `hosts: ubuntu`.
- RHEL uses `httpd` as the Apache package/service name; Ubuntu commonly uses `apache2`.
- The control node's `/etc/fstab` is copied to the managed nodes. Be sure this is what you intend before running the playbook.
- If the same playbook targets multiple operating systems, separate plays are a clear way to use different package modules and package names.

## 10. Run a play as a particular remote user

The notes ask how to run a play as `suman1` and show:

```yaml
- name: ******
  hosts:*********
  user: suman1
  become: true
```

A valid example is:

```yaml
- name: Configure managed hosts
  hosts: suman
  remote_user: suman1
  become: true
  tasks:
    - name: Show the remote identity
      ansible.builtin.command: whoami
```

- `remote_user: suman1` selects the SSH user for the managed hosts. The play-level alias `user: suman1` is also commonly accepted.
- `become: true` asks Ansible to escalate privileges for tasks, typically to root.
- The remote account must exist on each managed node, SSH access must work, and privilege escalation must be permitted. If sudo requires a password, use `--ask-become-pass` (`-K`) when running the playbook.

## 11. Useful commands for this session

```bash
# Check which hosts belong to the group
ansible suman --list-hosts

# Check playbook syntax without executing tasks
ansible-playbook --syntax-check task.yml

# Preview likely changes
ansible-playbook task.yml --check --verbose

# Execute the playbook
ansible-playbook task.yml

# Run one task at a time with confirmation
ansible-playbook task.yml --step

# Verify HTTP service state on all hosts in the group
ansible suman -m command -a 'systemctl is-active httpd'
ansible suman -m command -a 'systemctl is-enabled httpd'
```

## 12. Quick revision

| Concept | Meaning |
|---|---|
| `hosts` | Selects the inventory group/hosts for a play |
| `gather_facts: false` | Skips automatic fact gathering |
| `file` with `state: directory` | Ensures a directory exists |
| `copy` | Copies a file from the control node to managed hosts |
| `yum` | Installs/manages packages on RHEL-family systems |
| `apt` | Installs/manages packages on Debian/Ubuntu systems |
| `service` | Manages service state and boot-time enablement |
| `state: started` | Ensures a service is running |
| `enabled: true` | Enables service startup at boot |
| `become: true` | Enables privilege escalation for tasks |
| `--check` / `-C` | Dry-run/check mode |
| `-v` | Verbose output |
| `--step` | Prompts before each task |
| Multiple plays | Allows one playbook to target different host groups |

## 13. Original Session 8 notes — preserved verbatim

The complete original source text is included below so that the original commands, output, spelling, and formatting are retained.

```text
Session8
-----------
yaml

----------------------------

- 1. Create a directory on all machines in /tmp directory with the same names as testdir
- 2. copy the /ete/fstab file from the central server to the /tmp/testdir on the managed nodes
- 3. install httpd package on all nodes.
- 4. start and enabled httpd service. 

------------------------------
ans 

vim task.yml 
thne type code in yml formate

---
- hosts: suman
  gather_facts: false
  tasks:
    - file:
        path: /tmp/testdir
        state: directory
    - copy:
        src: /etc/fstab
        dest: /tmp/testdir
    - yum:
        name:
           - httpd
           - wget

    - service:
        name: httpd
        state: started
        enabled: true

:wq


ansible-playbook task.yml -C -v

suman1@ansible-server:~$ ansible-playbook task.yml

PLAY [suman] *******************************************************************

TASK [file] ********************************************************************
changed: [node1]
changed: [node2]

TASK [copy] ********************************************************************
changed: [node1]
changed: [node2]

TASK [yum] *********************************************************************
ok: [node2]
ok: [node1]

TASK [service] *****************************************************************
changed: [node2]
changed: [node1]

PLAY RECAP *********************************************************************
node1                      : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~$ ansible suman -m command -a 'systemctl status httpd'
node2 | CHANGED | rc=0 >>
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Fri 2026-10-16 16:46:16 IST; 6min ago
 Invocation: 4efa8b16901d46ffb9be4f2949c762e7
       Docs: man:httpd.service(8)
   Main PID: 80306 (httpd)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
      Tasks: 177 (limit: 22568)
     Memory: 13.4M (peak: 13.7M)
        CPU: 611ms
     CGroup: /system.slice/httpd.service
             ├─80306 /usr/sbin/httpd -DFOREGROUND
             ├─80307 /usr/sbin/httpd -DFOREGROUND
             ├─80308 /usr/sbin/httpd -DFOREGROUND
             ├─80309 /usr/sbin/httpd -DFOREGROUND
             └─80353 /usr/sbin/httpd -DFOREGROUND

Oct 16 16:46:16 node2.suman.local systemd[1]: Starting httpd.service - The Apache HTTP Server...
Oct 16 16:46:16 node2.suman.local (httpd)[80306]: httpd.service: Referenced but unset environment variable evaluates to an empty string: OPTIONS
Oct 16 16:46:16 node2.suman.local httpd[80306]: Server configured, listening on: port 80
Oct 16 16:46:16 node2.suman.local systemd[1]: Started httpd.service - The Apache HTTP Server.
node1 | CHANGED | rc=0 >>
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Fri 2026-10-16 16:46:16 IST; 6min ago
 Invocation: ad9c24e8ec0b4475b21d2a5515c6bead
       Docs: man:httpd.service(8)
   Main PID: 88637 (httpd)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
      Tasks: 177 (limit: 22568)
     Memory: 13.5M (peak: 13.7M)
        CPU: 749ms
     CGroup: /system.slice/httpd.service
             ├─88637 /usr/sbin/httpd -DFOREGROUND
             ├─88638 /usr/sbin/httpd -DFOREGROUND
             ├─88639 /usr/sbin/httpd -DFOREGROUND
             ├─88640 /usr/sbin/httpd -DFOREGROUND
             └─88697 /usr/sbin/httpd -DFOREGROUND

Oct 16 16:46:16 node1.suman.local systemd[1]: Starting httpd.service - The Apache HTTP Server...
Oct 16 16:46:16 node1.suman.local (httpd)[88637]: httpd.service: Referenced but unset environment variable evaluates to an empty string: OPTIONS
Oct 16 16:46:16 node1.suman.local httpd[88637]: Server configured, listening on: port 80
Oct 16 16:46:16 node1.suman.local systemd[1]: Started httpd.service - The Apache HTTP Server.
suman1@ansible-server:~$ 

interactive mode ----------------->

suman1@ansible-server:~$ ansible-playbook task.yml --step

PLAY [suman] *******************************************************************
Perform task: TASK: file (N)o/(y)es/(c)ontinue: y

Perform task: TASK: file (N)o/(y)es/(c)ontinue: ********************************

TASK [file] ********************************************************************
ok: [node2]
ok: [node1]
Perform task: TASK: copy (N)o/(y)es/(c)ontinue: y

Perform task: TASK: copy (N)o/(y)es/(c)ontinue: ********************************

TASK [copy] ********************************************************************
ok: [node2]
ok: [node1]
Perform task: TASK: yum (N)o/(y)es/(c)ontinue: y

Perform task: TASK: yum (N)o/(y)es/(c)ontinue: *********************************

TASK [yum] *********************************************************************
ok: [node1]
ok: [node2]
Perform task: TASK: service (N)o/(y)es/(c)ontinue: y

Perform task: TASK: service (N)o/(y)es/(c)ontinue: *****************************

TASK [service] *****************************************************************
ok: [node2]
ok: [node1]


better virsion of test.yml

suman1@ansible-server:~$ cat task.yml 
---
- name: frist playbook
  hosts: suman
  gather_facts: false
  tasks:
    - name: creating /tmp/testdir 
      file:
        path: /tmp/testdir
        state: directory
    - name: copy /etc/fstab to /tmp/testdir
      copy:
        src: /etc/fstab
        dest: /tmp/testdir
    - name: installing httpd and wget
      yum:
        name: 
           - httpd
           - wget

    - name: starte and enable service
      service:
        name: httpd
        state: started
        enabled: true
suman1@ansible-server:~$ 

now if we want to write 2 plays in one file it also possible ----->

suman1@ansible-server:~$ cat task2.yml 
---
- name: frist playbook
  hosts: suman
  gather_facts: false
  tasks:
    - name: creating /tmp/testdir 
      file:
        path: /tmp/testdir
        state: directory
    - name: copy /etc/fstab to /tmp/testdir
      copy:
        src: /etc/fstab
        dest: /tmp/testdir
    - name: installing httpd and wget
      yum:
        name: 
           - httpd
           - wget

    - name: starte and enable service
      service:
        name: httpd
        state: started
        enabled: true
- name: deploy apache2 in ubantu 
  hosts: ubantu
  tasks:
    - name: download apache2
      apt:
        name: apache2
        state: present
    - name: start apache2
      service:
          name: apache2
          state: started
          enabled: true

suman1@ansible-server:~$ 

---------------------------------------------
if we want to run it from a specific user suppose suman1

so we need to add 
- name: ******
  hosts:*********
  user: suman1
  become: true 
		



```
