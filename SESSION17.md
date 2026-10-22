# SESSION17 — RHCE Ansible `when` Conditions (Detailed Beginner Guide)

## 1. Lab Overview

This session practises Ansible conditional execution using a mixed operating-system lab.

| Node | Operating system | Main practice areas |
|---|---|---|
| node1 | RHEL | DNF, HTTPD, SELinux, firewalld |
| node2 | RHEL | DNF, HTTPD, SELinux, firewalld |
| node3 | Ubuntu | APT, Apache2, Ubuntu-specific tasks |

The playbooks below assume your inventory groups or host entries identify these machines as `node1`, `node2`, and `node3`. Adjust hostnames if your inventory uses different names.

## 2. What Is a `when` Condition?

A `when` condition tells Ansible whether a task should run on a host.

- If the condition is true, Ansible runs the task.
- If the condition is false, Ansible skips the task.
- Conditions are commonly used when servers have different operating systems, packages, services, or configuration requirements.

A `when` statement is similar to an `if` statement in a programming language.

Example:

```yaml
- name: Display a message on RHEL only
  ansible.builtin.debug:
    msg: "This is a RHEL server"
  when: ansible_distribution == "RedHat"
```

Here, Ansible displays the message only on hosts whose distribution fact is `RedHat`.

### Important YAML rules

1. Use spaces for indentation; do not use tabs.
2. Put `when` at the same indentation level as the module (`ansible.builtin.debug`, `ansible.builtin.dnf`, etc.).
3. Do not put quotation marks around the entire condition.
4. The `==` operator means “is equal to.”
5. Facts must be available. Fact gathering is enabled by default unless you disable it.

## 3. Understand the Most Useful Facts

### `ansible_distribution`

Identifies the specific distribution, for example:

```yaml
when: ansible_distribution == "RedHat"
```

This checks specifically for Red Hat Enterprise Linux. Ubuntu commonly reports `Ubuntu`.

### `ansible_os_family`

Groups distributions into a broader operating-system family:

```yaml
when: ansible_os_family == "RedHat"
```

This matches RHEL and compatible Red Hat-family distributions such as Rocky Linux and AlmaLinux.

```yaml
when: ansible_os_family == "Debian"
```

This matches Debian-family distributions such as Ubuntu and Debian.

### `ansible_facts['pkg_mgr']`

Reports the package manager Ansible detects, such as `dnf` or `apt`.

```yaml
when: ansible_facts['pkg_mgr'] == "dnf"
```

This condition is useful when choosing a package-management task based on the detected package manager.

### Check facts on your hosts

```bash
ansible all -m setup -a 'filter=ansible_distribution'
ansible all -m setup -a 'filter=ansible_os_family'
ansible all -m setup -a 'filter=ansible_pkg_mgr'
```

Expected values depend on each machine's actual operating system and configuration.

---

# Exercise 1 — Display the OS Type on Each Node

**Level:** Beginner

## Goal

- Display `This is RHEL server` on node1 and node2.
- Display `This is Ubuntu server` on node3.

## Playbook: `q1_os_check.yml`

```yaml
---
- name: Check operating system
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Display RHEL message
      ansible.builtin.debug:
        msg: "This is RHEL server"
      when: ansible_distribution == "RedHat"

    - name: Display Ubuntu message
      ansible.builtin.debug:
        msg: "This is Ubuntu server"
      when: ansible_distribution == "Ubuntu"
```

## Explanation

- `hosts: all` targets every host in the inventory.
- `gather_facts: true` collects information about each host before tasks run.
- `ansible.builtin.debug` prints a message without changing the server.
- The first `when` permits the RHEL message only on Red Hat Enterprise Linux.
- The second `when` permits the Ubuntu message only on Ubuntu.
- The task whose condition is false is reported as skipped.

## Run and verify

```bash
ansible-playbook q1_os_check.yml --syntax-check
ansible-playbook q1_os_check.yml
```

## Expected result

| Host | First task | Second task |
|---|---|---|
| node1 (RHEL) | Runs | Skipped |
| node2 (RHEL) | Runs | Skipped |
| node3 (Ubuntu) | Skipped | Runs |

---

# Exercise 2 — Install Apache on All Three Nodes

**Level:** Beginner

## Goal

Install the correct Apache package for each operating-system family:

- RHEL family: `httpd`
- Debian family / Ubuntu: `apache2`

## Playbook: `q2_apache.yml`

```yaml
---
- name: Install Apache based on OS
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Install Apache on RHEL family
      ansible.builtin.dnf:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install Apache on Debian family
      ansible.builtin.apt:
        name: apache2
        state: present
        update_cache: true
      when: ansible_os_family == "Debian"
```

## Explanation

- `become: true` requests privilege escalation, normally to root, for package installation.
- `ansible.builtin.dnf` manages packages with DNF.
- `ansible.builtin.apt` manages packages with APT.
- `state: present` means the package should be installed; it does not mean “install again every time.”
- `update_cache: true` asks APT to refresh package metadata before installing.
- Each task's condition ensures the package module runs on the appropriate OS family.

## Run and verify

```bash
ansible-playbook q2_apache.yml --syntax-check
ansible-playbook q2_apache.yml
```

Check packages on RHEL:

```bash
ansible 'node1:node2' -b -m command -a "rpm -q httpd"
```

Check the package on Ubuntu:

```bash
ansible node3 -b -m command -a "dpkg-query -W apache2"
```

## Troubleshooting

- RHEL systems need working and enabled DNF repositories.
- Ubuntu needs working APT repositories and network access if the package is not already available.
- Confirm your inventory host pattern `node1:node2` matches your inventory naming. If not, target the correct hosts or use a group.

---

# Exercise 3 — Start and Enable the Correct Apache Service

**Level:** Intermediate

## Goal

Start the right service and enable it to start automatically at boot.

- RHEL: service name `httpd`
- Ubuntu: service name `apache2`

## Playbook: `q3_service.yml`

```yaml
---
- name: Start Apache services
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Start HTTPD on RHEL family
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true
      when: ansible_os_family == "RedHat"

    - name: Start Apache on Debian family
      ansible.builtin.service:
        name: apache2
        state: started
        enabled: true
      when: ansible_os_family == "Debian"
```

## Explanation

- `ansible.builtin.service` manages system services.
- `state: started` ensures the service is running.
- `enabled: true` enables the service to start at boot.
- The OS condition prevents Ansible from trying to start `httpd` on Ubuntu or `apache2` on RHEL.

## Run and verify

```bash
ansible-playbook q3_service.yml --syntax-check
ansible-playbook q3_service.yml
```

Check RHEL services:

```bash
ansible 'node1:node2' -b -m command -a "systemctl is-active httpd"
```

Check Ubuntu:

```bash
ansible node3 -b -m command -a "systemctl is-active apache2"
```

The relevant package must be installed first. If the service is missing, install the package before running this exercise.

---

# Exercise 4 — Create Different Users on RHEL and Ubuntu

**Level:** Intermediate

## Goal

- Create `rheladmin` on RHEL.
- Create `ubuntuadmin` on Ubuntu.

## Playbook: `q4_users.yml`

```yaml
---
- name: Create OS-specific users
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Create user on RHEL
      ansible.builtin.user:
        name: rheladmin
        state: present
      when: ansible_distribution == "RedHat"

    - name: Create user on Ubuntu
      ansible.builtin.user:
        name: ubuntuadmin
        state: present
      when: ansible_distribution == "Ubuntu"
```

## Explanation

- `ansible.builtin.user` manages user accounts.
- `name` is the account name to manage.
- `state: present` ensures the user exists.
- The conditions restrict each account to its intended distribution.

## Run and verify

```bash
ansible-playbook q4_users.yml --syntax-check
ansible-playbook q4_users.yml
```

Check RHEL:

```bash
ansible 'node1:node2' -b -m command -a "id rheladmin"
```

Check Ubuntu:

```bash
ansible node3 -b -m command -a "id ubuntuadmin"
```

**Note:** Checking `rheladmin` on Ubuntu is expected to fail because this exercise does not create that user there. Likewise, `ubuntuadmin` is not created by this playbook on RHEL.

---

# Exercise 5 — Install a Package Using the Detected Package Manager

**Level:** Intermediate to advanced

## Goal

Install `tree` using DNF on hosts where the detected package manager is `dnf`, and using APT on hosts where it is `apt`.

## Playbook: `q5_pkg_mgr.yml`

```yaml
---
- name: Install tree based on package manager
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Install tree using DNF
      ansible.builtin.dnf:
        name: tree
        state: present
      when: ansible_facts['pkg_mgr'] == "dnf"

    - name: Install tree using APT
      ansible.builtin.apt:
        name: tree
        state: present
        update_cache: true
      when: ansible_facts['pkg_mgr'] == "apt"
```

## Explanation

- `ansible_facts['pkg_mgr']` contains the detected package-manager name.
- If its value is `dnf`, the DNF task runs.
- If its value is `apt`, the APT task runs.
- The conditions prevent the wrong package module from running on a host.

## Run and verify

```bash
ansible all -m setup -a 'filter=ansible_pkg_mgr'
ansible-playbook q5_pkg_mgr.yml --syntax-check
ansible-playbook q5_pkg_mgr.yml
```

Verify installation:

```bash
ansible all -b -m command -a "tree --version"
```

The last command assumes the `tree` executable is installed and available in the command path.

---

# Exercise 6 — Run a Task Only When a Boolean Variable Is True

**Level:** Advanced

## Goal

Install Apache only when `install_web` is true, and select the correct package for each OS family.

## Playbook: `q6_variable.yml`

```yaml
---
- name: Conditional Apache installation
  hosts: all
  become: true
  gather_facts: true

  vars:
    install_web: true

  tasks:
    - name: Install Apache on RHEL family
      ansible.builtin.dnf:
        name: httpd
        state: present
      when:
        - install_web | bool
        - ansible_os_family == "RedHat"

    - name: Install Apache on Debian family
      ansible.builtin.apt:
        name: apache2
        state: present
        update_cache: true
      when:
        - install_web | bool
        - ansible_os_family == "Debian"
```

## Explanation

- `vars` defines a playbook variable.
- `install_web: true` enables the installation.
- `install_web | bool` converts the variable to a Boolean value for the condition.
- When a `when` condition is written as a YAML list, all listed conditions must be true.
- Therefore, a task runs only when installation is enabled **and** the host belongs to the correct OS family.

## Test both cases

### Case A: Enable installation

```yaml
vars:
  install_web: true
```

Both OS-specific tasks run only on their matching operating systems.

### Case B: Disable installation

Change the value to:

```yaml
vars:
  install_web: false
```

Both installation tasks are skipped on all hosts.

Run the playbook after each change:

```bash
ansible-playbook q6_variable.yml
```

---

# Exercise 7 — Check Whether Apache Is Installed Before Starting It

**Level:** Advanced

## Goal

Check whether the correct Apache package exists. Start the service only when the package check succeeds.

## Playbook: `q7_service_check.yml`

```yaml
---
- name: Check Apache before starting service
  hosts: all
  become: true
  gather_facts: true

  tasks:
    - name: Check HTTPD package on RHEL family
      ansible.builtin.command:
        cmd: rpm -q httpd
      register: rhel_httpd
      changed_when: false
      failed_when: false
      when: ansible_os_family == "RedHat"

    - name: Start HTTPD if installed
      ansible.builtin.service:
        name: httpd
        state: started
      when:
        - ansible_os_family == "RedHat"
        - rhel_httpd.rc == 0

    - name: Check Apache package on Debian family
      ansible.builtin.command:
        cmd: dpkg-query -W apache2
      register: ubuntu_apache
      changed_when: false
      failed_when: false
      when: ansible_os_family == "Debian"

    - name: Start Apache if installed
      ansible.builtin.service:
        name: apache2
        state: started
      when:
        - ansible_os_family == "Debian"
        - ubuntu_apache.rc == 0
```

## Explanation of the important keywords

### `register`

```yaml
register: rhel_httpd
```

Saves the result of the package-check command in a variable named `rhel_httpd`.

### `changed_when: false`

The package check only inspects the system, so Ansible should not report the check itself as a change.

### `failed_when: false`

A failed package query should not stop the play. The next task can use the return code to decide whether to start the service.

### `rc == 0`

`rc` means return code. For these commands, a return code of `0` indicates that the package query succeeded. A nonzero code indicates that it did not succeed.

### Why are there two conditions?

```yaml
when:
  - ansible_os_family == "RedHat"
  - rhel_httpd.rc == 0
```

Both must be true:

1. The host belongs to the Red Hat OS family.
2. The HTTPD package check succeeded.

The OS condition also prevents Ansible from trying to use a result variable that belongs to the other OS branch.

## Run and verify

```bash
ansible-playbook q7_service_check.yml --syntax-check
ansible-playbook q7_service_check.yml
```

Verify RHEL:

```bash
ansible 'node1:node2' -b -m command -a "systemctl is-active httpd"
```

Verify Ubuntu:

```bash
ansible node3 -b -m command -a "systemctl is-active apache2"
```

If the package is absent, this playbook skips the service-start task; it does not install the package.

---

# Exercise 8 — Your Practice Challenge

**Level:** Intermediate

## Task

Write a playbook that creates:

- `/opt/rhel-lab` on RHEL nodes.
- `/opt/ubuntu-lab` on Ubuntu nodes.
- Permissions must be `0755` for both directories.

## Hint

Use `ansible.builtin.file`, `state: directory`, and two `when` conditions.

A directory task will use options like:

```yaml
ansible.builtin.file:
  path: /opt/rhel-lab
  state: directory
  mode: '0755'
```

Write the full playbook yourself first, then test it.

## Suggested validation workflow

```bash
ansible-playbook q8_directories.yml --syntax-check
ansible-playbook q8_directories.yml
```

Verify directories and permissions:

```bash
ansible 'node1:node2' -b -m command -a "stat -c '%a %n' /opt/rhel-lab"
ansible node3 -b -m command -a "stat -c '%a %n' /opt/ubuntu-lab"
```

Expected permissions: `755`.

---

# 5. Common Mistakes and Fixes

| Mistake | How to fix it |
|---|---|
| Wrong indentation | Use consistent spaces in YAML |
| Facts are unavailable | Enable `gather_facts: true` |
| Wrong package name | Use `httpd` for RHEL and `apache2` for Ubuntu |
| Wrong service name | Use `httpd` on RHEL and `apache2` on Ubuntu |
| Wrong OS family value | Check with the Ansible setup module |
| Package installation fails | Check repository configuration and connectivity |
| User verification fails on the other OS | Verify only where that user is intended to exist |
| Service does not start | Confirm the package is installed and inspect service logs |

## 6. RHCE Lab Practice Checklist

Use this checklist while practising:

- [ ] Inspect the inventory and confirm all three nodes are reachable.
- [ ] Run `ansible all -m setup -a 'filter=ansible_distribution'`.
- [ ] Create and syntax-check each playbook.
- [ ] Run each playbook and read the `ok`, `changed`, `skipped`, and `failed` counts.
- [ ] Verify the result directly on the relevant hosts.
- [ ] Change `install_web` from `true` to `false` and run Exercise 6 again.
- [ ] Complete Exercise 8 without looking at a solution.

### Useful commands

```bash
ansible all --list-hosts
ansible all -m ping
ansible all -m setup -a 'filter=ansible_distribution'
ansible-playbook q1_os_check.yml --syntax-check
ansible-playbook q1_os_check.yml
```

**Key takeaway:** `when` controls *whether* a task runs. Facts tell you about the host, variables let you control behaviour, and `register` saves task results so later tasks can make decisions.
