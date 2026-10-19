# SESSION 14 — Ansible Loops

## 1. Introduction to Ansible Loops

Ansible loops allow a task to be repeated for multiple items without writing a separate task for every item. They are useful when you need to create several user accounts, install multiple packages, create directories, or perform the same operation on a list of resources.

### Learning objectives

By the end of this session, you should be able to:

- Explain why loops are useful in Ansible.
- Identify the repeated-task problem in a playbook without loops.
- Use a simple loop to process a list of items.
- Understand the session's introduction to mapping/hash loops and nested loops.
- Use a loop to create user accounts and install packages.

## 2. Problems in Ansible Without Loops

Suppose you need to create `n` user accounts. Without a loop, you may have to create a separate task for every user. As the number of accounts increases, the playbook becomes longer and harder to maintain.

### Example: `users.yml`

```yaml
---
- name: user account creating without loops
  hosts: all
  tasks:
    - name: adding user hello world
      user:
        name: world
        state: present

    - name: adding user hello amit
      user:
        name: amit
        state: present
```

### Explanation

- `hosts: all` targets every host in the selected inventory.
- `tasks` contains the operations Ansible should execute.
- The `user` module manages user accounts.
- `name` identifies the account to manage.
- `state: present` ensures that the account exists.

This playbook creates two user accounts, `world` and `amit`, with separate tasks. For many accounts, repeating this structure would be inefficient.

## 3. Using Ansible Loops

A loop repeats a task for each item in a list. Instead of duplicating the task, you define it once and supply the items to process.

The session introduces three loop categories:

1. **Simple/standard loop** — repeats a task for individual items in a list.
2. **Hash/mapping loop** — introduces data represented as related fields, such as usernames and passwords.
3. **Nested loop** — a loop inside another loop.

The examples below preserve the terminology and examples from the original session.

## 4. Simple/Standard Loop

A standard loop is useful when each iteration uses one item, such as a username or package name. In the supplied example, Ansible exposes the current item through the variable `item`.

### Example data: `file.txt`

```text
Username:
vikas
amit
greta
Tarun
```

This text file lists usernames as an example of a collection of individual values. The playbook shown below writes the list directly into the YAML file rather than reading `file.txt`.

### Example: `standerd_loop.yml`

```yaml
---
- name: example of standerd loop
  hosts: all
  tasks:
    - name: adding user accounts
      user:
        name: "{{ item }}"
        state: present
      with_items:
        - vikas
        - amit
        - greta
        - tala
        - fedora
...
```

### How it works

- `hosts: all` runs the play against all inventory hosts.
- The task named `adding user accounts` uses the `user` module.
- `name: "{{ item }}"` sets the username to the current loop item.
- `state: present` ensures that the user exists.
- `with_items` provides the list of usernames.
- Ansible repeats the task for `vikas`, `amit`, `greta`, `tala`, and `fedora`.

**Important:** The values in `file.txt` are not automatically read by this playbook. The five usernames are explicitly listed under `with_items`.

### Expected result

When the playbook completes successfully, Ansible will have ensured that all five listed accounts exist on each targeted host, subject to permissions, platform support, and task success.

## 5. Using a Loop to Install Packages

Loops can repeat package-management tasks as well as user-management tasks.

### Example: `yum_loop.yml`

```yaml
---
- name: Ansible loop playbook for yum
  hosts: node2
  tasks:
    - name: using yum with loop
      yum:
        name: "{{ item }}"
        state: present
      with_items:
        - zsh
        - httpd
        - vsftpd
        - ftp
        - wget
        - gzip
...
```

### Explanation

- `hosts: node2` targets the inventory host or group named `node2`.
- The `yum` module manages packages on supported systems.
- `name: "{{ item }}"` supplies the current package name.
- `state: present` requests that the package be installed.
- `with_items` lists the packages: `zsh`, `httpd`, `vsftpd`, `ftp`, `wget`, and `gzip`.

Ansible repeats the package task for each item. The packages must be available through the configured repositories, and the remote host must meet the module's requirements.

**RHEL note:** The original example uses the `yum` module. On current Ansible installations, `ansible.builtin.dnf` is commonly used for modern RHEL systems and supports package management on current RHEL releases. Check your installed Ansible version and lab requirements before changing the module in an exam or class exercise.

## 6. Hash/Mapping Loops

The source introduces hash/mapping loops using a table that associates usernames with passwords.

### Example data: `file1.txt`

```text
Username:        Password:
vikas            1234
amit             123
greta            12
```

This illustrates records with more than one related value. A mapping-style loop can be useful when each record has fields such as a username and another property.

The original source does **not** include a complete hash/mapping-loop playbook, so no additional playbook is attributed to the session here.

### Conceptual distinction

- A simple loop processes one value per iteration, such as a username.
- A mapping-style loop processes related fields together, such as a username and a password.

**Security reminder:** The passwords in the supplied table are examples only and are weak. Do not use them for real accounts. Avoid storing plaintext passwords in playbooks or source-control repositories. For real automation, use Ansible Vault or another approved secrets-management method and follow your organization's password policy.

## 7. Nested Loops

The source defines a nested loop as a loop inside another loop.

Nested loops are useful when an operation must be repeated across two sets of values—for example, applying several actions to several targets. The original session does not provide a nested-loop playbook, so this section explains only the concept and does not invent a source example.

## 8. Quick Comparison

| Approach | Structure | Example use |
|---|---|---|
| Separate tasks without loops | One task per item | Create a small number of users |
| Simple/standard loop | One task and a list of items | Create multiple users |
| Hash/mapping loop | Records containing related fields | Work with username/property pairs |
| Nested loop | A loop inside another loop | Combine two sets of values |

## 9. Common Errors and Checks

1. **Incorrect indentation:** YAML indentation defines the structure. Align `with_items` with the module task, as shown in the examples.
2. **Incorrect variable:** Use `{{ item }}` when following the provided `with_items` examples.
3. **Wrong target:** Confirm that `node2` exists in the inventory before running `yum_loop.yml`.
4. **Missing privileges:** Creating users and installing packages usually require sufficient privileges on the managed host. Configure privilege escalation where your lab requires it.
5. **Unavailable packages:** A package installation may fail if the package name is wrong or the enabled repositories do not provide it.
6. **Confusing a text file with loop input:** The sample `file.txt` is illustrative; the supplied playbook explicitly defines its loop items and does not read the file.

## 10. Practice and Revision Questions

1. What is an Ansible loop?
2. Why are loops useful when creating many user accounts?
3. What does `{{ item }}` represent in the supplied examples?
4. What is the purpose of `with_items`?
5. Which module is used to manage user accounts in `standerd_loop.yml`?
6. Which host is targeted by `yum_loop.yml`?
7. Name the six packages in the package-loop example.
8. What is meant by a hash/mapping loop in this session?
9. What is a nested loop?
10. Why should plaintext passwords not be used in real playbooks?

### Short exam-ready answer

**Ansible loop:** An Ansible loop repeats a task for multiple items, reducing duplicated YAML. In the session examples, `with_items` provides a list and `{{ item }}` represents the current value. Loops are demonstrated for creating user accounts and installing packages.

## 11. Commands and Playbook Review Checklist

Before executing either playbook:

- Check YAML formatting and indentation.
- Confirm the inventory contains the intended hosts.
- Confirm SSH connectivity to managed nodes.
- Ensure the required privilege escalation is configured.
- Review the user and package lists before applying changes.
- Run the playbook in the intended lab environment and inspect its result.

The source does not include execution output, so no successful run is assumed here.

---

# Original Session 14 Text

The complete original text from `session14.txt` is preserved below.

```text
session14
-------------------------

Ansible loop
--------------
----------------
# Problems in ansible without loops

1. Let,s us suppose we have n number of user accounts - create separate tasks for each user account in the ansible playbook.

-> 
suman1@ansible-server:~$ cat users.yml 
---
- name: user account creating without loops
  hosts: all
  tasks:
    - name: adding user hello world
      user:
        name: world
        state: present

    - name: adding user hello amit
      user:
        name: amit
        state: present

suman1@ansible-server:~$ 

-----------------------------------------------------------------
2> using Ansible Loops
 a- simple/Standerd Loop------------>

suman1@ansible-server:~$ cat file.txt  
Username:
vikas
amit
greta
Tarun

 b- Hash/Maping Loops------>

suman1@ansible-server:~/playbook$ cat file1.txt 
Username:		Password:
vikas			1234
amit			123
greta			12
suman1@ansible-server:~/playbook$ 

 c- Nested Loops

when we have loop inside a loop we called it Nested loop  
-----------------------------------------------------------------


 simple/Standerd Loop------------>


suman1@ansible-server:~/playbook$ cat standerd_loop.yml 
---
- name: example of standerd loop
  hosts: all
  tasks:
    - name: adding user accounts
      user:
        name: "{{ item }}"
        state: present
      with_items:
        - vikas
        - amit
        - greta
        - tala
        - fedora
...

---------

suman1@ansible-server:~/playbook$ cat yum_loop.yml
---
- name: Ansible loop playbook for yum
  hosts: node2
  tasks:
    - name: using yum with loop
      yum:
        name: "{{ item }}"
        state: present
      with_items:
        - zsh
        - httpd
        - vsftpd
        - ftp
        - wget
        - gzip
...
----------------------------------------------------------






```
