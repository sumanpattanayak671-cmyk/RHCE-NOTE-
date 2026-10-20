# SESSION16 --- Ansible Loops: Hash Loops and Nested Loops

## 1. Session objectives

By the end of this session, you should understand how to:

-   Create multiple Linux groups with an Ansible loop.
-   Create users and assign each user to a specific group using a
    hash/dictionary-style loop.
-   Understand why a normal loop is not enough when every user must be
    paired with multiple groups.
-   Use `with_nested` to generate every combination of users and groups.
-   Validate a playbook, perform a check-mode run, and interpret the
    output.

## 2. Lab environment

The examples in the source notes use:

-   Ansible control node: `ansible-server`
-   Inventory group: `suman`
-   Managed nodes: `node1` and `node2`
-   Working directory: `~/playbook`

The commands below should be run from the control node in the directory
containing the playbooks.

## 3. Hash/dictionary-style loop: one group per user

### Example 1 --- `hash4.yml`

This playbook creates the `linux` and `unix` groups, then creates four
users and assigns each user to one corresponding group.

``` yaml
---
- name: Adding users and groups by using a hash loop
  hosts: suman
  tasks:
    - name: Adding groups
      ansible.builtin.group:
        name: "{{ item }}"
        state: present
      with_items:
        - linux
        - unix

    - name: Adding users
      ansible.builtin.user:
        name: "{{ item.usr }}"
        state: present
        groups: "{{ item.grp }}"
      loop:
        - { usr: sp1, grp: linux }
        - { usr: sp2, grp: linux }
        - { usr: sp3, grp: unix }
        - { usr: sp4, grp: unix }
```

### How it works

-   `hosts: suman` runs the tasks on every host in the `suman` inventory
    group.
-   The group task iterates over `linux` and `unix`. Here, `item` is a
    group name.
-   The user task iterates over a list of dictionaries.
-   `item.usr` reads the username from the current dictionary.
-   `item.grp` reads the group name from the same dictionary.
-   `state: present` ensures the user or group exists.

  User    Assigned group
  ------- ----------------
  `sp1`   `linux`
  `sp2`   `linux`
  `sp3`   `unix`
  `sp4`   `unix`

### Validate and test

``` bash
ansible-playbook hash4.yml --syntax-check
ansible-playbook hash4.yml --check --verbose
```

The source notes use `-C -v`; these are short forms of
`--check --verbose`.

**Important:** Check mode is a simulation. It helps preview likely
changes, but it does not prove that all changes will succeed when
applied. To make the changes, run the playbook without `--check` after
reviewing it:

``` bash
ansible-playbook hash4.yml
```

## 4. Another hash-loop example: `hash_loop3.yml`

This example creates four groups (`su`, `sa`, `si`, and `pe`) and four
users (`ad1`, `ad2`, `ad3`, and `ad4`). Each user is paired with one
group.

``` yaml
---
- name: Adding users with their groups
  hosts: suman
  tasks:
    - name: Adding groups
      ansible.builtin.group:
        name: "{{ item }}"
        state: present
      with_items:
        - su
        - sa
        - si
        - pe

    - name: Adding users with their groups by using a loop
      ansible.builtin.user:
        name: "{{ item.usr }}"
        state: present
        groups: "{{ item.grp }}"
      loop:
        - { usr: ad1, grp: su }
        - { usr: ad2, grp: sa }
        - { usr: ad3, grp: si }
        - { usr: ad4, grp: pe }
```

### User-to-group mapping

  User    Assigned group
  ------- ----------------
  `ad1`   `su`
  `ad2`   `sa`
  `ad3`   `si`
  `ad4`   `pe`

Validate and simulate:

``` bash
ansible-playbook hash_loop3.yml --syntax-check
ansible-playbook hash_loop3.yml --check --verbose
```

The source output shows that the `su` group already existed
(`changed: false`), while other groups and users were reported as
changes in check mode. The exact output depends on the current state of
each managed node.

## 5. Why use a nested loop?

A dictionary-style loop is convenient when each user has one clearly
specified group. But suppose the requirement changes: every user must be
added to **each** of several groups.

For example, four users and four groups produce:

-   `ad1` with `su`, `sa`, `si`, and `pe`
-   `ad2` with `su`, `sa`, `si`, and `pe`
-   `ad3` with `su`, `sa`, `si`, and `pe`
-   `ad4` with `su`, `sa`, `si`, and `pe`

That means (4 `\times 4`{=tex} = 16) user/group combinations. A nested
loop is useful because it generates the Cartesian product of the lists.

## 6. Nested loop example --- `hash5.yml`

The source notes demonstrate `with_nested`, using `item[0]` for the
username and `item[1]` for the group. The following is a clean, valid
YAML version of that approach:

``` yaml
---
- name: Adding users with multiple groups
  hosts: suman
  tasks:
    - name: Adding groups
      ansible.builtin.group:
        name: "{{ item }}"
        state: present
      loop:
        - su
        - sa
        - si
        - pe

    - name: Adding users to each group using a nested loop
      ansible.builtin.user:
        name: "{{ item[0] }}"
        state: present
        groups: "{{ item[1] }}"
        append: true
      with_nested:
        - [ad1, ad2, ad3, ad4]
        - [su, sa, si, pe]
```

### Understanding the nested-loop variables

Each iteration creates a pair:

-   `item[0]` --- the user from the first list.
-   `item[1]` --- the group from the second list.
-   `groups: "{{ item[1] }}"` --- adds the current group to the user.
-   `append: true` --- preserves the user's existing supplementary
    groups while adding the specified group.

For example, iterations include:

  Iteration example   `item[0]`   `item[1]`   Result
  ------------------- ----------- ----------- -------------------
  1                   `ad1`       `su`        Add `ad1` to `su`
  2                   `ad1`       `sa`        Add `ad1` to `sa`
  3                   `ad1`       `si`        Add `ad1` to `si`
  4                   `ad1`       `pe`        Add `ad1` to `pe`
  5                   `ad2`       `su`        Add `ad2` to `su`
  ...                 ...         ...         ...
  16                  `ad4`       `pe`        Add `ad4` to `pe`

### Important YAML correction

The source notes show the lists as:

``` yaml
with_nested:
  - { ad1, ad2, ad3, ad4 }
  - { su, sa, si, pe }
```

That brace notation is not the correct way to write these lists in YAML.
Use square-bracket lists, as in the corrected example:

``` yaml
with_nested:
  - [ad1, ad2, ad3, ad4]
  - [su, sa, si, pe]
```

## 7. Run and verify the nested-loop playbook

First check YAML/Ansible syntax:

``` bash
ansible-playbook hash5.yml --syntax-check
```

Then simulate the tasks:

``` bash
ansible-playbook hash5.yml --check --verbose
```

If the syntax check passes and the planned changes look correct, apply
the playbook:

``` bash
ansible-playbook hash5.yml
```

Check the users and their supplementary groups on a managed node:

``` bash
id ad1
id ad2
id ad3
id ad4
```

You can also inspect a user's group membership with:

``` bash
groups ad1
```

Run these verification commands on the managed node itself, or through
an approved Ansible command/task.

## 8. Understanding Ansible output

Common output fields:

-   `ok` --- the task completed without reporting a change.
-   `changed` --- Ansible reports that the task changed something or, in
    check mode, predicts a change.
-   `unreachable` --- Ansible could not connect to the host.
-   `failed` --- a task failed.
-   `skipped` --- a task was skipped.
-   `PLAY RECAP` --- summarizes results for each managed host.

In the supplied source output, `node1` and `node2` both show
`unreachable=0` and `failed=0`. In check mode, the `changed` count
represents predicted changes and should not be interpreted as proof that
the system was modified.

## 9. Key differences

  -----------------------------------------------------------------------
  Feature                 Hash/dictionary-style   Nested loop
                          loop                    
  ----------------------- ----------------------- -----------------------
  Main purpose            Pair each record's      Combine each value in
                          fields, such as user    one list with each
                          and group               value in another

  Typical expression      `item.usr`, `item.grp`  `item[0]`, `item[1]`

  Example mapping         `sp1` → `linux`         `ad1` → each group in
                                                  the list

  Number of user/group    One per dictionary      Product of list lengths
  pairings                entry                   

  Useful when             Each user has a         Every user needs every
                          predefined group        listed group
                          mapping                 
  -----------------------------------------------------------------------

## 10. Troubleshooting checklist

1.  **Syntax check fails:** Check indentation, colons, quotes, and list
    syntax.
2.  **A user task fails:** Confirm that the target groups are created
    before user tasks run.
3.  **Users do not appear in all groups:** Confirm that the nested loop
    has both lists and that `groups` uses `item[1]`.
4.  **Existing groups disappear:** Use `append: true` when the intention
    is to add a supplementary group without replacing the existing
    supplementary-group list.
5.  **A host is unreachable:** Check inventory addresses, SSH
    connectivity, credentials, and privilege escalation configuration.
6.  **Check mode reports changes:** Remember that `--check` previews
    changes; run without it only after reviewing the planned actions.

## 11. Practice exercises

1.  Create groups `dev`, `test`, and `ops` with a loop.
2.  Use a dictionary-style loop to create `user1`, `user2`, and `user3`,
    assigning each to a different group.
3.  Use `with_nested` to add `user1` and `user2` to all three groups.
    How many user/group combinations should run?
4.  Run `--syntax-check`, then `--check --verbose`, and explain the
    `PLAY RECAP`.
5.  Verify the final memberships using `id user1` and `groups user1`.

## 12. Quick revision

-   Use a regular loop to process a list of values.
-   Use a dictionary/hash-style loop when each item contains named
    fields, for example `usr` and `grp`.
-   Use a nested loop when you need every combination from multiple
    lists.
-   In the nested-loop example, `item[0]` is the username and `item[1]`
    is the group.
-   `append: true` helps preserve existing supplementary groups while
    adding another.
-   Run `ansible-playbook <file>.yml --syntax-check` before execution.
-   Use `--check --verbose` to preview changes, then execute normally
    when ready.

------------------------------------------------------------------------

**Session:** 16\
**Topic:** Ansible hash loops and nested loops\
**Lab files:** `hash4.yml`, `hash_loop3.yml`, `hash5.yml`
