# SESSION 10 — Ansible Variables

## Table of Contents

1. [Introduction](#1-introduction)
2. [Command-line variables](#2-command-line-variables)
3. [Password hashing for user accounts](#3-password-hashing-for-user-accounts)
4. [Interactive input with `vars_prompt`](#4-interactive-input-with-vars_prompt)
5. [Inventory-level variables](#5-inventory-level-variables)
6. [Host-scope variables](#6-host-scope-variables)
7. [Group-scope variables](#7-group-scope-variables)
8. [Using `host_vars` and `group_vars`](#8-using-host_vars-and-group_vars)
9. [Variable precedence and troubleshooting](#9-variable-precedence-and-troubleshooting)
10. [Practice tasks](#10-practice-tasks)
11. [Quick revision](#11-quick-revision)
12. [Original source text](#12-original-source-text)

---

## 1. Introduction

Ansible variables let you reuse a playbook with different values without rewriting the tasks. In this session, the examples cover:

- Passing variables when running a playbook with `-e` / `--extra-vars`.
- Creating users and supplying a password value.
- Hashing a password before passing it to Ansible's `user` module.
- Prompting for input with `vars_prompt`.
- Defining variables for individual hosts or groups in an inventory.
- Moving host and group variables into separate `host_vars` and `group_vars` files.

The examples use an inventory group named `suman`, with managed nodes such as `node1`, `node2`, and `node3`.

## 2. Command-line variables

### What are command-line variables?

Command-line variables are values supplied when you execute a playbook. Ansible accepts them through `-e` or `--extra-vars`.

General form:

```bash
ansible-playbook playbook.yml -e "variable_name=value"
```

You can pass multiple variables:

```bash
ansible-playbook playbook.yml -e "user_name=nandu password=123"
```

The source examples also use the equivalent separate-argument form:

```bash
ansible-playbook command_var.yml -e user_name=nandu -e password=123
```

### Example playbook: `command_var.yml`

```yaml
---
- name: Example of command-line variables
  hosts: suman
  tasks:
    - name: Add a user and password
      ansible.builtin.user:
        name: "{{ user_name }}"
        state: present
        password: "{{ password }}"
```

Run it:

```bash
ansible-playbook command_var.yml -e user_name=nandu -e password=123
```

**How it works**

- `hosts: suman` targets the hosts in the `suman` inventory group.
- `{{ user_name }}` is replaced with the value passed for `user_name`.
- `state: present` ensures that the user exists.
- `{{ password }}` receives the supplied password value.

**Important:** the first source example passes `123` directly to the `password` parameter. The output warns that the password does not appear to be hashed. On Linux, Ansible's `user` module expects the `password` field to contain a suitable password hash, not the plain-text password. Use the hashed-password example below instead.

## 3. Password hashing for user accounts

### Why hash the password?

The password supplied to Ansible should be transformed into a password hash suitable for the target system. Do not put a plain-text password directly into the `user` module's `password` field.

The source uses the `password_hash('sha512')` filter:

```yaml
---
- name: Example of command-line variables
  hosts: suman
  tasks:
    - name: Add a user with a hashed password
      ansible.builtin.user:
        name: "{{ user_name }}"
        state: present
        password: "{{ password | password_hash('sha512') }}"
```

Run the playbook with values:

```bash
ansible-playbook command_var.yml -e user_name=nandu3 -e password=1234
```

In this example, the value supplied as `password` is passed through the `password_hash` filter before being used by the module.

### Note about the source output

The source shows a deprecation warning related to Python's `crypt` module and mentions `passlib`. This warning depends on the Ansible/Python environment and version. If you see it, consult the documentation for the exact versions installed in your lab and use a supported password-hashing dependency. Do not suppress a warning without understanding it.

**Security reminder:** command-line extra variables can be exposed through shell history, process listings, logs, or terminal recordings. For real systems, avoid passing sensitive passwords directly on the command line. Consider Ansible Vault or a secure secrets-management workflow.

## 4. Interactive input with `vars_prompt`

`vars_prompt` allows a playbook to ask for values when it starts.

The source's `uin.yml` example is:

```yaml
---
- name: User input and hiding information
  hosts: suman
  vars_prompt:
    - name: user_name
      prompt: "Please enter your username."
      private: no
    - name: password
      prompt: "Enter your password"
      private: yes
  tasks:
    - name: Add a user with an encrypted password
      ansible.builtin.user:
        name: "{{ user_name }}"
        state: present
        password: "{{ password | password_hash('sha512') }}"
```

Run it:

```bash
ansible-playbook uin.yml
```

Ansible asks for the username and password before running the tasks.

- `name: user_name` stores the entered username in the `user_name` variable.
- `private: no` displays the username as it is typed.
- `name: password` stores the entered password in the `password` variable.
- `private: yes` hides the password while it is typed.
- The password is passed through `password_hash('sha512')` before it reaches the `user` module.

For new playbooks, `ansible.builtin.user` is the fully qualified module name. The shorter `user` name shown in the original notes is also commonly used.

## 5. Inventory-level variables

Inventory variables are values associated with hosts or groups in the inventory. They allow the same playbook to behave differently for different managed nodes.

The source introduces two categories:

1. **Host-scope variables** — associated with an individual host.
2. **Group-scope variables** — associated with every host in a group, unless overridden by a variable with higher precedence.

Example inventory:

```ini
[suman]
node1 dir=/tmp/testdir
node2 dir=/tmp/testdir

[liu]
node1
node2
node3

[ubantu]
node3 dir=/tmp/newdir
```

In this example:

- `node1` and `node2` have `dir=/tmp/testdir` in the `suman` group.
- `node3` has `dir=/tmp/newdir` in the `ubantu` group.
- The group names and host names must match the inventory and the playbook's `hosts` target.

A host can belong to more than one inventory group. If the same variable is defined in multiple places, Ansible's variable precedence rules determine which value is used.

## 6. Host-scope variables

A host variable is intended for a particular managed host. In the original inventory example, variables are written on the same line as the host:

```ini
[ubantu]
node3 dir=/tmp/newdir
```

The source also demonstrates a dedicated host variable file:

**File:** `/etc/ansible/host_vars/node3`

```yaml
dir: /tmp/helooo
```

Ansible automatically loads a host variable file whose filename matches the inventory host name. Thus, `node3` receives the value `/tmp/helooo` from this file, provided the file is in the expected `host_vars` directory associated with the inventory.

A common project layout is:

```text
/etc/ansible/
├── hosts
├── host_vars/
│   └── node3
└── group_vars/
    └── suman
```

The exact variable-directory location is normally relative to the inventory source or project layout being used. If variables are not being loaded, check the inventory path and Ansible configuration.

## 7. Group-scope variables

A group variable applies to the hosts belonging to a group.

The source shows this inventory configuration:

```ini
[suman]
node1
node2

[liu]
node1
node2
node3

[ubantu]
node3

[suman:vars]
dir=/tmp/exampledir
```

The `suman:vars` section assigns `/tmp/exampledir` to the `dir` variable for hosts in the `suman` group.

A playbook can then use that value:

```yaml
---
- name: Inventory variable example
  hosts: suman
  tasks:
    - name: Create a directory
      ansible.builtin.file:
        path: "{{ dir }}"
        state: directory

    - name: Display the directory variable
      ansible.builtin.debug:
        msg: "{{ dir }}"
```

Run it:

```bash
ansible-playbook inventory_var.yml
```

The file module creates the directory on each targeted host if it does not already exist. The debug task prints the value of `dir` for each host.

## 8. Using `host_vars` and `group_vars`

### Why use separate variable files?

When an inventory contains many hosts and groups, putting every variable directly in the main hosts file can make it difficult to maintain. The source recommends separating variables into directories:

```bash
sudo mkdir -p /etc/ansible/host_vars
sudo mkdir -p /etc/ansible/group_vars
```

### Host variable file

Create a file named after the host:

```bash
sudo vim /etc/ansible/host_vars/node3
```

Example contents:

```yaml
dir: /tmp/helooo
```

This assigns a host-specific value to `node3`.

### Group variable file

Create a file named after the group:

```bash
sudo vim /etc/ansible/group_vars/suman
```

Example contents:

```yaml
dir: /tmp/heloooo2
```

This assigns a group value to members of `suman`.

The source runs the playbook against `suman` and sees `/tmp/heloooo2` for `node1` and `node2`. It then changes the playbook target to `ubantu`; for `node3`, it sees `/tmp/helooo` from the host-specific variable file.

### Example playbook

```yaml
---
- name: Inventory variable example
  hosts: suman
  tasks:
    - name: Create a directory
      ansible.builtin.file:
        path: "{{ dir }}"
        state: directory

    - name: Display the directory variable
      ansible.builtin.debug:
        msg: "{{ dir }}"
```

For the `ubantu` group, change the target:

```yaml
hosts: ubantu
```

Then run:

```bash
ansible-playbook inventory_var.yml
```

**Expected behavior in the source example:**

- For `hosts: suman`, `node1` and `node2` use `/tmp/heloooo2` from `group_vars/suman`.
- For `hosts: ubantu`, `node3` uses `/tmp/helooo` from `host_vars/node3`.

The actual result depends on your inventory and variable-file locations. Keep group names and host names consistent.

## 9. Variable precedence and troubleshooting

When the same variable name is set in multiple places, Ansible uses variable precedence rules. As a practical rule for this session, a host-specific value can take precedence over a group variable for that host; command-line extra variables generally have very high precedence.

Troubleshooting checklist:

1. **Check the playbook syntax.**
   ```bash
   ansible-playbook --syntax-check inventory_var.yml
   ```
2. **Check which hosts are targeted.**
   ```bash
   ansible-inventory --graph
   ```
3. **Inspect the inventory.**
   ```bash
   ansible-inventory --list
   ```
4. **Check the variable files.** Ensure `host_vars` and `group_vars` are placed where Ansible expects them, and filenames match the inventory host/group names.
5. **Check YAML indentation.** The `file` and `debug` tasks must be indented under `tasks`.
6. **Check that the variable exists.** A task using `{{ dir }}` will fail if no applicable variable defines `dir`.
7. **Use a debug task to display the value.**
   ```yaml
   - name: Display the directory variable
     ansible.builtin.debug:
       msg: "{{ dir }}"
   ```
8. **Avoid exposing secrets.** Do not print passwords or other sensitive values with debug tasks.
9. **Use a hashed password.** The `user` module's `password` parameter should receive an appropriate hash, not a plain-text password.
10. **Review warnings.** Deprecation warnings may indicate a dependency or version change that needs attention.

## 10. Practice tasks

Try these tasks in your own lab:

1. Create a user using a username supplied through `-e`.
2. Update the playbook to hash the supplied password using `password_hash('sha512')`.
3. Create a playbook that prompts for a username visibly and a password privately.
4. Define `dir=/tmp/testdir` for `node1` and `node2` in the inventory.
5. Define a different directory for `node3` using `host_vars/node3`.
6. Define a shared directory variable for the `suman` group using `group_vars/suman`.
7. Run the same playbook against `suman` and `ubantu`, and compare the debug output.
8. Run `ansible-playbook --syntax-check inventory_var.yml` before execution.
9. Explain why the original un-hashed password example produced a warning.
10. Explain why a separate `host_vars` and `group_vars` structure can be easier to maintain than a large inventory file.

## 11. Quick revision

| Term | Meaning |
|---|---|
| `-e` / `--extra-vars` | Pass variables when launching a playbook. |
| `{{ variable_name }}` | Jinja expression used to reference a variable in YAML. |
| `password_hash('sha512')` | Filter used in the source examples to hash a password value. |
| `vars_prompt` | Prompt for values before the playbook runs. |
| `private: no` | Display the prompted value while typing. |
| `private: yes` | Hide the prompted value while typing. |
| Host variable | Variable associated with a particular host. |
| Group variable | Variable associated with a group of hosts. |
| `host_vars/` | Directory for host-specific variable files. |
| `group_vars/` | Directory for group-specific variable files. |
| `ansible.builtin.file` | Module used here to ensure a directory exists. |
| `ansible.builtin.debug` | Module used here to print a non-sensitive value. |

---

## 12. Original source text

The complete original text from `Session 10.txt` is appended below, without intentionally removing or rewriting its content. It is retained for reference alongside the detailed notes.

```text
Session 10

command line variable 
-------------------------
 you can define variables when you run playbookby passing variables at the command line using the -- extra-vars(or-e) argument

create user nandu and passwoed 123 by using command line 
suman1@ansible-server:~/playbook$ cat command_var.yml 
---
- name: example of command line variables
  hosts: suman
  tasks:
    - name : adding user and password
      user: 
          name: "{{ user_name }}"
          state: present
          password: "{{ password }}"
...
suman1@ansible-server:~/playbook$
suman1@ansible-server:~/playbook$ ansible-playbook command_var.yml -e user_name=nandu -e password=123

PLAY [example of command line variables] ****************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node1]
ok: [node2]

TASK [adding user and password] *************************************************************************************
[WARNING]: The input password appears not to have been hashed. The 'password' argument must be encrypted for this
module to work properly.
changed: [node2]
changed: [node1]

PLAY RECAP **********************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
-----------------------------------------------------------

create a user ram with an encrypted password 123

suman1@ansible-server:~/playbook$ cat command_var.yml 
---
- name: example of command line variables
  hosts: suman
  tasks:
    - name : adding user and password
      user: 
          name: "{{ user_name }}"
          state: present
          password: "{{ password | password_hash ('sha512') }}"
...
suman1@ansible-server:~/playbook$ ansible-playbook command_var.yml -e user_name=nandu3 -e password=1234

PLAY [example of command line variables] ****************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node1]
ok: [node2]

TASK [adding user and password] *************************************************************************************
[DEPRECATION WARNING]: Encryption using the Python crypt module is deprecated. The Python crypt module is deprecated
 and will be removed from Python 3.13. Install the passlib library for continued encryption functionality. This 
feature will be removed in version 2.17. Deprecation warnings can be disabled by setting deprecation_warnings=False 
in ansible.cfg.
changed: [node1]
changed: [node2]

PLAY RECAP **********************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ ansible suman -m command -a 'cat /etc/shadow' 

-----------------------------------------------------------
-----------------------------------------------------------
encrypt password and user input --->

suman1@ansible-server:~/playbook$ ansible-playbook uin.yml
please enter your username.: hi
enter your pass: 

PLAY [user input and hiding information] ****************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node1]
ok: [node2]

TASK [adding user name and passwordwith encription] *****************************************************************
[DEPRECATION WARNING]: Encryption using the Python crypt module is deprecated. The Python crypt module is deprecated
 and will be removed from Python 3.13. Install the passlib library for continued encryption functionality. This 
feature will be removed in version 2.17. Deprecation warnings can be disabled by setting deprecation_warnings=False 
in ansible.cfg.
changed: [node1]
changed: [node2]

PLAY RECAP **********************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ cat uin.yml 
---
- name: user input and hiding information
  hosts: suman
  vars_prompt:
    - name: user_name
      prompt: please enter your username.
      private: no
    - name: password
      prompt: enter your pass
      private: yes
  tasks:
    - name: adding user name and passwordwith encription 
      user:
        name: "{{ user_name }}"
        state: present
        password: "{{ password | password_hash ('sha512')}}"
...
suman1@ansible-server:~/playbook$ ansible suman -m command -a 'cat /etc/shadow' 


--------------------------------------------------------------
--------------------------------------------------------------
--------------------------------------------------------------
inventory level variables
------------>category
 1. host scope variables
 2. group scope variables 


----------------
1> host scope ...........

sudo cp /etc/ansible/hosts /etc/ansible/hosts.bkp

suman1@ansible-server:~/playbook$ sudo vi /etc/ansible/hosts

[suman]
node1 dir=/tmp/testdir
node2 dir=/tmp/testdir

[liu]
node1
node2
node3

[ubantu]
node3 dir=/tmp/newdir


:wq

suman1@ansible-server:~/playbook$ cat inventory_var.yml 
---
- name: inventory variable example
  hosts: all
  tasks: 
     - name: creating directorys
       file:
         path: "{{ dir }}"
         state: directory
     - debug:
         msg: "{{ dir }}"

suman1@ansible-server:~/playbook$ ansible-playbook inventory_var.yml

PLAY [inventory variable example] ***********************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node2]
ok: [node1]
ok: [node3]

TASK [creating directorys] ******************************************************************************************
changed: [node3]
ok: [node1]
ok: [node2]

TASK [debug] ********************************************************************************************************
ok: [node1] => {
    "msg": "/tmp/testdir"
}
ok: [node2] => {
    "msg": "/tmp/testdir"
}
ok: [node3] => {
    "msg": "/tmp/newdir"
}

PLAY RECAP **********************************************************************************************************
node1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

------------------------------------------------------------------------

group scop........>

sudo vi /etc/ansible/hosts

[suman]
node1
node2

[liu]
node1
node2
node3

[ubantu]
node3

[suman:vars]
dir=/tmp/exampledir
-- INSERT --                                                                                       67,20         Bot
:wq

-------------------

suman1@ansible-server:~/playbook$ cat inventory_var.yml 
---
- name: inventory variable example
  hosts: suman
  tasks: 
     - name: creating directorys
       file:
         path: "{{ dir }}"
         state: directory
     - debug:
         msg: "{{ dir }}"

------------------------

suman1@ansible-server:~/playbook$ ansible-playbook inventory_var.yml

PLAY [inventory variable example] ***********************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node2]
ok: [node1]

TASK [creating directorys] ******************************************************************************************
changed: [node1]
changed: [node2]

TASK [debug] ********************************************************************************************************
ok: [node1] => {
    "msg": "/tmp/exampledir"
}
ok: [node2] => {
    "msg": "/tmp/exampledir"
}

PLAY RECAP **********************************************************************************************************
node1                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

----------------------------------------------------------------------------------------
-----------------------------------------------------------------------------



=================================================================================

---------------------------------------------------------------------------------------

suppose wee have many groups so then it is deficult to maintain and using the original host file is a rsik 

solution --


 suman1@ansible-server:~/playbook$ sudo mkdir /etc/ansible/host_vars
suman1@ansible-server:~/playbook$ sudo mkdir /etc/ansible/group_vars
suman1@ansible-server:~/playbook$ sudo vim /etc/ansible/host_vars
suman1@ansible-server:~/playbook$ sudo vim /etc/ansible/host_vars/node3
suman1@ansible-server:~/playbook$ sudo cat /etc/ansible/host_vars/node3
dir: /tmp/helooo
suman1@ansible-server:~/playbook$ sudo vim /etc/ansible/group_vars/suman
suman1@ansible-server:~/playbook$ sudo cat /etc/ansible/group_vars/suman 
dir: /tmp/heloooo2
suman1@ansible-server:~/playbook$ cat inventory_var.yml 
---
- name: inventory variable example
  hosts: suman
  tasks: 
     - name: creating directorys
       file:
         path: "{{ dir }}"
         state: directory
     - debug:
         msg: "{{ dir }}"
suman1@ansible-server:~/playbook$ ansible-playbook inventory_var.yml

PLAY [inventory variable example] ***********************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node1]
ok: [node2]

TASK [creating directorys] ******************************************************************************************
changed: [node1]
changed: [node2]

TASK [debug] ********************************************************************************************************
ok: [node1] => {
    "msg": "/tmp/heloooo2"
}
ok: [node2] => {
    "msg": "/tmp/heloooo2"
}

PLAY RECAP **********************************************************************************************************
node1                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ vim inventory_var.yml 
suman1@ansible-server:~/playbook$ cat inventory_var.yml 
---
- name: inventory variable example
  hosts: ubantu
  tasks: 
     - name: creating directorys
       file:
         path: "{{ dir }}"
         state: directory
     - debug:
         msg: "{{ dir }}"
suman1@ansible-server:~/playbook$ ansible-playbook inventory_var.yml

PLAY [inventory variable example] ***********************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node3]

TASK [creating directorys] ******************************************************************************************
changed: [node3]

TASK [debug] ********************************************************************************************************
ok: [node3] => {
    "msg": "/tmp/helooo"
}

PLAY RECAP **********************************************************************************************************
node3                      : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ 






```
