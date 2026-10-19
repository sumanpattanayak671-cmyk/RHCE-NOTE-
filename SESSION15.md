# SESSION 15 --- Ansible Lookup Plugins and Loops

## 1. Learning objectives

By the end of this session, you should be able to:

-   List Ansible lookup plugins and inspect their documentation.
-   Use a variable file with a loop to manage multiple users.
-   Read a list of usernames from a file with `with_lines`.
-   Use a list of dictionaries (often informally called a "hash loop")
    when each user needs different properties.
-   Validate a playbook and understand the limits of check mode.
-   Recognize the difference between a user's primary group and
    supplementary groups.

------------------------------------------------------------------------

## 2. Discovering lookup plugins

Ansible lookup plugins retrieve data from sources on the **control
node** while Ansible processes a playbook. Examples include reading a
file, accessing an environment variable, or selecting a matching file.

### List available lookup plugins

``` bash
ansible-doc -t lookup -l
```

The session output included these plugins:

  ---------------------------------------------------------------------------
  Lookup plugin                           Purpose shown by `ansible-doc`
  --------------------------------------- -----------------------------------
  `ansible.builtin.config`                Lookup current Ansible
                                          configuration values

  `ansible.builtin.csvfile`               Read data from a TSV or CSV file

  `ansible.builtin.dict`                  Return key/value pair items from
                                          dictionaries

  `ansible.builtin.env`                   Read environment variables

  `ansible.builtin.file`                  Read file contents

  `ansible.builtin.fileglob`              List files matching a pattern

  `ansible.builtin.first_found`           Return the first file found from a
                                          list

  `ansible.builtin.indexed_items`         Rewrite lists as indexed items

  `ansible.builtin.ini`                   Read data from an INI file

  `ansible.builtin.inventory_hostnames`   Select inventory hosts matching a
                                          pattern

  `ansible.builtin.items`                 Work with items

  `ansible.builtin.lines`                 Read lines from a command

  `ansible.builtin.list`                  Return the supplied value

  `ansible.builtin.nested`                Combine lists into nested
                                          combinations

  `ansible.builtin.password`              Retrieve or generate a random
                                          password stored in a file

  `ansible.builtin.pipe`                  Read output from a command

  `ansible.builtin.random_choice`         Choose a random item from a list

  `ansible.builtin.sequence`              Generate a number sequence

  `ansible.builtin.subelements`           Traverse a nested key in a list of
                                          dictionaries

  `ansible.builtin.template`              Read a file after Jinja2 templating

  `ansible.builtin.together`              Merge lists in synchronized order

  `ansible.builtin.unvault`               Read Ansible Vault-encrypted file
                                          contents

  `ansible.builtin.url`                   Read contents from a URL

  `ansible.builtin.varnames`              Find matching variable names

  `ansible.builtin.vars`                  Look up templated variable values
  ---------------------------------------------------------------------------

> The exact list can vary with the installed Ansible version and
> collections.

### Read documentation for one lookup plugin

``` bash
ansible-doc -t lookup ansible.builtin.file
ansible-doc -t lookup ansible.builtin.env
ansible-doc -t lookup ansible.builtin.lines
```

General pattern:

``` bash
ansible-doc -t lookup <plugin_name>
```

**Remember:** `ansible-doc -t lookup -l` lists lookup plugins;
`ansible-doc -t lookup <name>` displays detailed documentation for a
selected plugin.

------------------------------------------------------------------------

## 3. Standard loop with a variable file

### Scenario

Suppose an administrator needs to create nine users on the hosts in the
inventory group `suman`. All of these users should have the same group
configuration. Instead of repeating the task nine times, store the
usernames in a variable file and loop over them.

### Step 1 --- Create the variable file

File: `userss.yml`

``` yaml
users:
  - user1
  - user2
  - user3
  - user4
  - user5
  - user6
  - user7
  - user8
  - user9
```

This YAML defines a variable named `users`, whose value is a list of
nine strings.

### Step 2 --- Create the playbook

File: `standerd_loop3.yml`

``` yaml
---
- name: Add users using a standard loop
  hosts: suman
  become: true

  vars_files:
    - /home/suman1/playbook/userss.yml

  tasks:
    - name: Ensure users exist and use wheel as their primary group
      ansible.builtin.user:
        name: "{{ item }}"
        state: present
        group: wheel
      loop: "{{ users }}"
```

#### How it works

-   `hosts: suman` targets the inventory group named `suman`.
-   `vars_files` loads the external YAML file.
-   `users` contains the list of usernames.
-   `loop: "{{ users }}"` iterates through that list.
-   `item` represents the current username for each iteration.
-   `state: present` ensures that the account exists.
-   `group: wheel` sets the user's **primary group** to `wheel`.
-   `become: true` requests privilege escalation, normally required for
    creating system users.

The original session used `with_items`:

``` yaml
with_items:
  - "{{ users }}"
```

The modern, preferred form for a straightforward list is:

``` yaml
loop: "{{ users }}"
```

Both approaches are loop mechanisms, but `loop` is generally clearer for
new playbooks.

### Step 3 --- Validate and test

``` bash
ansible-playbook standerd_loop3.yml --syntax-check
ansible-playbook standerd_loop3.yml --check --verbose
```

The session used `-C -v`; these are short forms of `--check --verbose`.

-   `--syntax-check` checks playbook syntax.
-   `--check` (or `-C`) asks Ansible to simulate changes where
    supported.
-   `--verbose` (or `-v`) provides more execution detail.

The session output showed each username being processed on `node1` and
`node2`, with a recap reporting no unreachable or failed hosts. However,
**check mode does not prove that users were actually created**. Run the
playbook without `--check` when you are ready to apply changes.

### Step 4 --- Apply the playbook

``` bash
ansible-playbook standerd_loop3.yml
```

Only run this against the intended lab inventory. Account creation
changes the target machines.

------------------------------------------------------------------------

## 4. `with_lines` to read usernames from a file

### Scenario

Suppose a text file already contains the administrator account names:

File: `adminuser`

``` text
admin1
admin2
admin3
admin4
```

The session demonstrates `with_lines` to execute a command on the
**control node** and use each output line as a loop item.

### Playbook from the session

File: `standerd4.yml`

``` yaml
---
- name: Add administrator users
  hosts: all
  become: true

  tasks:
    - name: Add each account listed in adminuser
      ansible.builtin.user:
        name: "{{ item }}"
        state: present
        group: wheel
      with_lines: cat adminuser
```

The session's command was:

``` bash
ansible-playbook standerd4.yml --syntax-check
ansible-playbook standerd4.yml --check --verbose
```

The output showed `admin1`, `admin2`, `admin3`, and `admin4` as loop
items on `node1`, `node2`, and `node3`.

### Important path and security notes

-   `with_lines` runs the command on the Ansible control node, not
    independently on each managed node.
-   `cat adminuser` uses a **relative path**. It works only if the
    controller's current working directory is the directory containing
    `adminuser`.
-   A clearer path is safer, for example
    `/home/suman1/playbook/adminuser`, provided that this is the real
    location of the file.
-   The command must output one username per line.
-   Because this executes a command, use it only with trusted commands
    and input.

For new playbooks, consider the `ansible.builtin.file` lookup to read a
file, then split it into lines. For example:

``` yaml
---
- name: Add administrator users from a text file
  hosts: all
  become: true

  vars:
    admin_users: >-
      {{ lookup('ansible.builtin.file', '/home/suman1/playbook/adminuser')
         .splitlines() }}

  tasks:
    - name: Ensure listed users exist
      ansible.builtin.user:
        name: "{{ item }}"
        state: present
        group: wheel
      loop: "{{ admin_users }}"
```

This alternative reads the file contents and splits them into lines
without invoking `cat`. Ensure the file has no blank lines or comments,
or filter those out before using its contents as usernames.

------------------------------------------------------------------------

## 5. A list of dictionaries ("hash loop")

### Why a simple list is not enough

A simple list such as:

``` yaml
users:
  - rahul
  - rohan
  - anuj
  - harsha
```

provides one value per iteration. If each user requires a different
group, the playbook needs more information than just the username.

The session's example requirement was:

  User     Intended group
  -------- ----------------
  Rahul    `wheel`
  Rohan    `wheel`
  Anuj     `root`
  Harsha   `root`

The session calls this a **hash loop** or **list of hash**. In
YAML/Ansible terminology, the data structure is more precisely a **list
of dictionaries**: every list item contains named keys and values.

### Playbook from the session

File: `hash1_loop.yml`

``` yaml
---
- name: Add users with their specified groups
  hosts: suman
  become: true

  tasks:
    - name: Ensure users exist with the specified primary group
      ansible.builtin.user:
        name: "{{ item.name }}"
        state: present
        groups: "{{ item.groups }}"
      loop:
        - { name: 'rahul1', groups: wheel }
        - { name: 'rahul2', groups: wheel }
        - { name: 'rahul3', groups: root }
        - { name: 'rahul4', groups: root }
        - { name: 'rahul5', groups: suman1 }
```

### Understand the dictionary fields

An item such as:

``` yaml
- { name: 'rahul1', groups: wheel }
```

is equivalent to the more readable YAML form:

``` yaml
- name: rahul1
  groups: wheel
```

-   `item.name` accesses the `name` value from the current dictionary.
-   `item.groups` accesses the `groups` value from the current
    dictionary.
-   `loop` iterates over the list of dictionaries.
-   Each iteration can therefore provide different values to the same
    task.

The session output showed all five dictionary items processed on `node1`
and `node2`, with no failures reported in the recap.

### Important distinction: `group` versus `groups`

The original playbook uses `groups: "{{ item.groups }}"`. In the Ansible
user module:

-   `group` sets the user's **primary group**.
-   `groups` sets the user's **supplementary groups**.
-   `groups` normally replaces the existing supplementary-group list
    unless `append: true` is used.

If the requirement is for `wheel` or `root` to be the primary group, use
`group`, not `groups`. If the requirement is supplementary membership,
use `groups`.

A safer example for supplementary membership is:

``` yaml
---
- name: Create users with supplementary group membership
  hosts: suman
  become: true

  tasks:
    - name: Ensure user exists and add the specified supplementary group
      ansible.builtin.user:
        name: "{{ item.name }}"
        state: present
        groups: "{{ item.groups }}"
        append: true
      loop:
        - name: rahul1
          groups: wheel
        - name: rahul2
          groups: wheel
        - name: rahul3
          groups: root
        - name: rahul4
          groups: root
```

**Security warning:** membership in `root` is highly privileged and is
not normally an appropriate group assignment for regular users. For
administrative access on many RHEL systems, membership in `wheel` is
commonly used with an appropriate `sudoers` policy. Verify the lab's
intended permissions before assigning groups.

Also ensure that every named group exists on the target hosts. Group
names and available privilege policies can differ across systems.

------------------------------------------------------------------------

## 6. Syntax checking, check mode, and output

### Recommended validation sequence

``` bash
# 1. Check YAML/playbook syntax
ansible-playbook hash1_loop.yml --syntax-check

# 2. Preview supported changes
ansible-playbook hash1_loop.yml --check --verbose

# 3. Apply changes after reviewing the preview
ansible-playbook hash1_loop.yml
```

### Understanding the recap

A play recap can include:

  -----------------------------------------------------------------------
  Field                               Meaning
  ----------------------------------- -----------------------------------
  `ok`                                Tasks completed without reporting a
                                      change or failure

  `changed`                           Tasks reported changes, or
                                      predicted changes in check mode

  `unreachable`                       Ansible could not reach the host

  `failed`                            Tasks failed

  `skipped`                           Tasks were skipped

  `rescued`                           Failures handled by a rescue block

  `ignored`                           Failures ignored by the playbook
  -----------------------------------------------------------------------

A recap with `failed=0` and `unreachable=0` is a good sign, but inspect
task output and verify the final state on the managed hosts. In check
mode, `changed` can mean a change is predicted rather than actually
applied.

### Verify users and groups on managed hosts

After applying the playbook, you can verify a user on a target host with
commands such as:

``` bash
id rahul1
getent passwd rahul1
getent group wheel
```

Use `id <username>` to inspect the user's UID, primary group, and
supplementary groups. Run these checks on the managed host (or use an
Ansible task) to confirm the actual result.

------------------------------------------------------------------------

## 7. Quick comparison

  ----------------------------------------------------------------------------------------------
  Technique                                 Input shape                  Best fit
  ----------------------------------------- ---------------------------- -----------------------
  `loop` with a list                        `user1`, `user2`, `user3`    Repeat the same
                                                                         operation for each
                                                                         value

  `vars_files` + `loop`                     A list stored in a separate  Keep data separate from
                                            YAML file                    task logic

  `with_lines`                              Output lines from a          Iterate over command
                                            controller-side command      output, one line at a
                                                                         time

  `lookup('ansible.builtin.file', ...)` +   Text-file contents           Read a local text file
  `splitlines()`                                                         into a list

  `loop` with a list of dictionaries        `{name: ..., groups: ...}`   Pass multiple per-item
                                                                         attributes, such as
                                                                         username and group
  ----------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 8. Common mistakes and troubleshooting

1.  **Incorrect variable-file path**\
    Confirm that `/home/suman1/playbook/userss.yml` exists on the
    controller and is readable.

2.  **Wrong inventory group**\
    `hosts: suman` must match a group in the inventory. Use
    `ansible-inventory --graph` to inspect inventory groups.

3.  **Missing privilege escalation**\
    User creation usually requires administrative privileges. Add
    `become: true` and configure privilege escalation appropriately.

4.  **Incorrect loop variable**\
    A list of strings uses `{{ item }}`. A list of dictionaries uses
    expressions such as `{{ item.name }}` and `{{ item.groups }}`.

5.  **Wrong primary/supplementary group setting**\
    Use `group` for the primary group; use `groups` for supplementary
    groups. Consider `append: true` when you want to preserve existing
    supplementary memberships.

6.  **Relative file path with `with_lines`**\
    `cat adminuser` depends on the controller's current working
    directory. Prefer an explicit, verified path.

7.  **Assuming check mode creates users**\
    `--check` is a preview. Apply the playbook without `--check` when
    you intend to make changes.

8.  **Group does not exist**\
    Ensure the requested group exists on each target, or create it with
    an appropriate `ansible.builtin.group` task before assigning users.

9.  **Unsafe administrative membership**\
    Do not add regular users to `root` casually. Confirm the intended
    privilege model first.

------------------------------------------------------------------------

## 9. Practice exercises

### Exercise 1 --- Use an external list

Create `developers.yml` with five usernames. Write a playbook that loads
the file and ensures each user exists on a lab host group.

### Exercise 2 --- Read from a text file

Create `operators.txt` with one username per line. Use a controller-side
file lookup to turn the contents into a list and loop over the names.

### Exercise 3 --- Use dictionaries

Create a list of users with a `name` and a `groups` field. Write a
playbook that uses `item.name` and `item.groups` and explain whether
each group is primary or supplementary.

### Exercise 4 --- Validate safely

For each exercise, run `--syntax-check`, then `--check --verbose`.
Inspect the predicted changes before applying anything.

### Exercise 5 --- Verify the final state

Apply one exercise in a disposable lab and use `id`, `getent passwd`,
and `getent group` to verify the users and memberships.

------------------------------------------------------------------------

## 10. Session summary

-   Discover lookup plugins with `ansible-doc -t lookup -l`.
-   Read a plugin's documentation with
    `ansible-doc -t lookup <plugin_name>`.
-   Store repeated values in a variable file and use `loop` to process
    them.
-   `with_lines` iterates over lines produced by a command executed on
    the control node.
-   A list of dictionaries lets each iteration carry multiple fields,
    such as a username and group.
-   `item` represents the current list entry; use dictionary keys such
    as `item.name` when the entry is a dictionary.
-   Use `--syntax-check` to validate syntax and `--check --verbose` to
    preview supported changes.
-   Remember that `group` means primary group and `groups` means
    supplementary groups in `ansible.builtin.user`.
-   Verify real changes on the managed hosts after applying the
    playbook.

**Key commands to remember**

``` bash
ansible-doc -t lookup -l
ansible-doc -t lookup ansible.builtin.file
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml --check --verbose
ansible-playbook playbook.yml
```
