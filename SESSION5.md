# SESSION 5 — Ansible Ad-Hoc Commands and Modules

> **Important:** The complete original Session 5 notes are preserved at the end of this document, including every command line, example, and output from the source text. The explanations below are additional study notes; they do not replace the original commands.

## Contents
1. Overwriting a file with `copy`
2. Creating backups with `backup=yes`
3. Editing files with `lineinfile`
4. Replacing text with `replace`
5. Managing users with `user`
6. Managing groups with `group`
7. Installing packages with `dnf`, `yum`, `apt`, and `package`
8. Understanding common Ansible output
9. Quick revision table
10. Original source notes — preserved verbatim

---

## 1. Overwriting a file with the `copy` module

### Goal
Replace the existing content of `/tmp/sumanw.txt` on all hosts in the `liu` inventory group with the text `suman is learning ansible`.

### Command from the notes
```bash
ansible liu -m copy -a 'content="suman is learning ansible " dest=/tmp/sumanw.txt'
```

### Explanation
- `ansible`: runs an Ansible ad-hoc command from the control node.
- `liu`: inventory group or host pattern; Ansible runs the task on every matching managed node.
- `-m copy`: selects the `copy` module.
- `-a`: supplies the module arguments.
- `content=...`: creates or overwrites the destination file with the supplied text.
- `dest=/tmp/sumanw.txt`: destination path on each managed node.

The following verification command reads the file on each node:
```bash
ansible liu -m command -a 'cat /tmp/sumanw.txt'
```

**Key point:** Using `content=` replaces the file's existing contents; it does not append to them. The task is suitable for small text files. Ensure the target path and intended content are correct before running it.

## 2. Overwrite a file and back up the old version

### Goal
Copy the control-node file `sampa.txt` to `/tmp/sumanw.txt` on each managed node, preserving a backup of the previous destination file.

### Command from the notes
```bash
ansible liu -m copy -a 'src=sampa.txt dest=/tmp/sumanw.txt backup=yes'
```

### Explanation
- `src=sampa.txt`: source file on the control node (relative paths are resolved from the directory in which the command is run).
- `dest=/tmp/sumanw.txt`: destination file on the managed nodes.
- `backup=yes`: asks Ansible to create a backup of the destination before replacing it when a change is required.

**Remember:** The source file must exist on the control node. Check the destination after running the command, and inspect the backup path reported by Ansible if you need to restore the previous content.

## 3. The `lineinfile` module

### Purpose
Use `lineinfile` to manage a particular line in a text file without rewriting the whole file manually. Common uses include:
- adding a line;
- ensuring a line is present;
- updating a line selected with a regular expression;
- removing a matching line;
- placing a line at the beginning or end of a file.

### 3.1 Add a line
```bash
ansible liu -m lineinfile -a 'path=/tmp/sumanw.txt line="hello everyone"'
```
- `path=` identifies the file on the managed node.
- `line=` is the exact line that should be present.
- If the line is absent, Ansible adds it. If the same line is already present, the module normally leaves the file unchanged, making it useful for repeatable automation.

Verify with:
```bash
ansible liu -m command -a "cat /tmp/sumanw.txt"
```

### 3.2 Insert a line at the beginning
```bash
ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="hello  sampa and everyone" insertafter=BOF'
```
`BOF` means **Beginning Of File**. In this command, the line is placed at the beginning of the file. The source uses `dest=`; `path=` is also commonly used as the file-path parameter.

### 3.3 Insert a line at the end
```bash
ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="thanks" insertafter=EOF'
```
`EOF` means **End Of File**. This is useful for adding a line at the end of a file.

### 3.4 Insert after a matching line
```bash
ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="and hope i passe the exam" insertafter=exam'
```
The notes demonstrate inserting a new line after a line matching `exam`. For dependable matching, Ansible's `insertafter` parameter is generally used with a regular expression; a pattern such as `insertafter='^i am prepaired for rhce exam$'` is more specific when that exact line is intended. The original command is preserved below exactly as supplied.

### 3.5 Remove a specific line
```bash
ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="thanks" state=absent'
```
`state=absent` removes the line specified by `line=`. The output field `found` indicates how many matching lines were found/removed in the reported result.

### 3.6 Remove lines beginning with `hello`
The original notes use:
```bash
ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt regexp=^hello state=absent'
```
- `regexp=^hello`: `^` anchors the expression to the start of a line, so it matches lines that begin with `hello`.
- `state=absent`: removes the matching line.

**Important distinction:** This expression removes lines that *start with* `hello`; it does not remove every line where `hello` appears anywhere. To match `hello` anywhere in a line, a pattern like `regexp='hello'` would be needed. The source command and its output are retained in the original section.

## 4. The `replace` module

### Purpose
The `replace` module searches a file using a regular expression and replaces matching text. Unlike `lineinfile`, it is designed for replacing matching portions of file content.

### Example from the notes
First inspect the file:
```bash
ansible liu -m command -a "cat /tmp/sampa.txt"
```
Then replace a line beginning with `hello` with `hi`:
```bash
ansible liu -m replace -a 'path=/tmp/sampa.txt regexp=^hello replace=hi'
```
Verify:
```bash
ansible liu -m command -a "cat /tmp/sampa.txt"
```

### Parameter breakdown
- `path=/tmp/sampa.txt`: file to edit on each managed node.
- `regexp=^hello`: matches `hello` at the beginning of a line.
- `replace=hi`: substitutes the matching text with `hi`.

Because the regular expression is `^hello`, the example changes `hello everyone` to `hi everyone`. It does not target the word `hello` in the middle of a line.

## 5. The `user` module

### Purpose
The `user` module creates, modifies, and removes local user accounts on managed nodes.

### 5.1 Create a user with a UID and supplementary group
Command from the notes:
```bash
ansible suman -m user -a 'name=user3 state=present uid=2020 groups=wheel'
```

Parameter breakdown:
- `suman`: target host pattern/inventory group.
- `-m user`: selects the user-management module.
- `name=user3`: account name.
- `state=present`: ensures the account exists.
- `uid=2020`: sets the numeric user ID.
- `groups=wheel`: adds `wheel` as a supplementary group.

The example output shows `/home/user3`, `/bin/bash`, UID `2020`, and group `wheel`. The `wheel` group is commonly used for administrative access on RHEL-like systems, but membership alone does not guarantee sudo access; sudo policy must also permit it.

Verify the account:
```bash
ansible suman -m command -a 'id -a user3'
```

### 5.2 Delete a user
```bash
ansible suman -m user -a 'name=user3 state=absent'
```
`state=absent` removes the user account. In the source output, `remove=false` means the user's home directory and files are not requested to be removed by that task.

Verify:
```bash
ansible suman -m command -a 'id -a user3'
```
After deletion, `id` reports that the user does not exist. That non-zero return code is expected for this verification command after successful removal.

## 6. The `group` module

### Purpose
The `group` module creates, modifies, and removes groups on managed nodes.

### 6.1 Create a group with a specific GID
```bash
ansible liu -m group -a 'name=suh state=present gid=1020'
```
- `name=suh`: group name.
- `state=present`: ensures the group exists.
- `gid=1020`: requests group ID 1020.

Verify the final line of `/etc/group`:
```bash
ansible liu -m command -a 'tail -1 /etc/group'
```
The sample output `suh:x:1020:` represents the group name, password placeholder, GID, and member list (empty in this example).

### 6.2 Delete the group
```bash
ansible liu -m group -a 'name=suh state=absent'
```
This removes the group named `suh`, provided the operating system permits the removal (for example, a primary group in use may prevent removal).

## 7. Package-management modules: `yum`, `dnf`, `apt`, and `package`

### Purpose
These modules install, update, and remove software packages on managed nodes.
- `yum`: traditionally used on older RHEL/CentOS systems.
- `dnf`: used on modern RHEL/Fedora systems.
- `apt`: used on Debian/Ubuntu systems.
- `package`: a generic module that can use a selected package manager; `use=` can explicitly select one.

### 7.1 Install `httpd` on the `suman` hosts using DNF
```bash
ansible suman -m dnf -a "name=httpd state=present"
```
- `name=httpd`: Apache HTTP Server package on RHEL-family systems.
- `state=present`: ensures it is installed. The notes also mention `state=install`; use the standard documented state `present` or `installed` for clarity.

### 7.2 Install `ksh` using the generic package module and DNF
```bash
ansible suman -m package -a 'name=ksh state=installed use=dnf'
```
`use=dnf` selects DNF as the package backend.

### 7.3 Install Apache on Ubuntu using APT
```bash
ansible ubantu -m apt -a "name=apache2 state=present"
```
`apache2` is the package name shown in the notes for Ubuntu/Debian. The inventory pattern is written as `ubantu` in the source and must match the actual inventory host/group name.

### 7.4 Install Apache with the generic package module and APT
The original source line is:
```bash
ansible ubantu -m package -a 'name=apache2 state=present use=apt
```
**Note:** As written in the source, this command is missing the closing single quote. A runnable version would be:
```bash
ansible ubantu -m package -a 'name=apache2 state=present use=apt'
```
The original unmodified line is also retained in the source section below.

**Privilege note:** Package installation, user/group administration, and changes to protected files often require elevated privileges. If privilege escalation is not configured in `ansible.cfg` or the inventory, use `-b` for the command, or configure privilege escalation for the lab as appropriate.

## 8. Understanding common Ansible output

| Output / field | Meaning |
|---|---|
| `SUCCESS` | The task ran successfully; it does not necessarily mean a change occurred. |
| `CHANGED` | Ansible reports that the task changed the managed node. |
| `FAILED` | The task failed; inspect the message and return code. |
| `changed: true` | The module reports a modification. |
| `changed: false` | No modification was needed or made. |
| `rc=0` | The command returned success. |
| `rc=1` | The command returned a non-zero status; interpretation depends on the command. |
| `state=present` | Ensure the resource exists. |
| `state=absent` | Ensure the resource is removed. |
| `backup=yes` | Create a backup before replacing an existing destination when a change is made. |
| `regexp=` | Match text using a regular expression. |
| `insertafter=BOF` | Insert at the beginning of the file. |
| `insertafter=EOF` | Insert at the end of the file. |

## 9. Quick revision table

| Module | Main use | Example parameter(s) |
|---|---|---|
| `copy` | Copy a file or write supplied content | `src=`, `content=`, `dest=`, `backup=yes` |
| `lineinfile` | Ensure, insert, edit, or remove a line | `path=`, `line=`, `regexp=`, `state=`, `insertafter=` |
| `replace` | Replace text matching a regular expression | `path=`, `regexp=`, `replace=` |
| `user` | Create or remove user accounts | `name=`, `uid=`, `groups=`, `state=` |
| `group` | Create or remove groups | `name=`, `gid=`, `state=` |
| `dnf` | Manage packages on modern RHEL-family systems | `name=`, `state=present` |
| `apt` | Manage packages on Ubuntu/Debian | `name=`, `state=present` |
| `package` | Generic package-management interface | `name=`, `state=`, `use=` |

---

# 10. Original source notes — preserved verbatim

The section below is copied from the uploaded `SESSI0N 5.txt` file without editing its command lines or output. Any spelling, formatting, or command issues in the original are intentionally left as supplied so that no original command line is lost.

```text
SESSI0N 5
------------------------------------
====================================
====================================

how to overwrite a test 
1. i have a file called /tmp/sumanw.txt where the file contain suman is suman now overwrite it's content as suman is learing ansible .

suman1@ansible-server:~$ ansible liu -m command -a 'cat /tmp/sumanw.txt'
192.168.105.6 | CHANGED | rc=0 >>
suman is suman 
192.168.105.5 | CHANGED | rc=0 >>
suman is suman 
192.168.105.4 | CHANGED | rc=0 >>
suman is suman 
suman1@ansible-server:~$ ansible liu -m copy -a 'content="suman is learning ansible " dest=/tmp/sumanw.txt'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "checksum": "570755c36b6e87589cc38b00499d59a9f3a35804",
    "dest": "/tmp/sumanw.txt",
    "gid": 0,
    "group": "root",
    "md5sum": "aa6b512ebf8bf91d1d85026c494a0448",
    "mode": "0644",
    "owner": "root",
    "size": 26,
    "src": "/home/suman1/.ansible/tmp/ansible-tmp-1792014421.747992-16199-100297081926318/source",
    "state": "file",
    "uid": 0
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "checksum": "570755c36b6e87589cc38b00499d59a9f3a35804",
    "dest": "/tmp/sumanw.txt",
    "gid": 0,
    "group": "root",
    "md5sum": "aa6b512ebf8bf91d1d85026c494a0448",
    "mode": "0644",
    "owner": "root",
    "secontext": "unconfined_u:object_r:user_home_t:s0",
    "size": 26,
    "src": "/home/suman1/.ansible/tmp/ansible-tmp-1792014421.892169-16198-149625900475678/source",
    "state": "file",
    "uid": 0
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "checksum": "570755c36b6e87589cc38b00499d59a9f3a35804",
    "dest": "/tmp/sumanw.txt",
    "gid": 0,
    "group": "root",
    "md5sum": "aa6b512ebf8bf91d1d85026c494a0448",
    "mode": "0644",
    "owner": "root",
    "secontext": "unconfined_u:object_r:user_home_t:s0",
    "size": 26,
    "src": "/home/suman1/.ansible/tmp/ansible-tmp-1792014421.8963957-16197-207017754573251/source",
    "state": "file",
    "uid": 0
}
suman1@ansible-server:~$ ansible liu -m command -a 'cat /tmp/sumanw.txt'
192.168.105.6 | CHANGED | rc=0 >>
suman is learning ansible 
192.168.105.5 | CHANGED | rc=0 >>
suman is learning ansible 
192.168.105.4 | CHANGED | rc=0 >>
suman is learning ansible 
------------------------------------------------------------------------------------------
2. now i have a file called sampa.txt that cantain i am prepared for rhce exam and i want to overwrite it on sumanw.txt but also want a backup of the previous content of the sumanw.txt file 

ans - suman1@ansible-server:~$ ansible liu -m copy -a 'src=sampa.txt dest=/tmp/sumanw.txt backup=yes'

----------------------------------------------------------
===========================================================
==========================================================
===========================================================
------------------------------------------------------------

Lineinfile------------>

1. Why do we use lineinfile?
- Add a new line to a configuration file.
- Modify an existing line.
- Remove a specific line.
- Ensure a particular configuration exists only once.
- Automate configuration changes across multiple managed nodes.

----------
1 .add one line "hello everyone " in sumanw.txt file by using ansible 

suman1@ansible-server:~$ ansible liu -m lineinfile -a 'path=/tmp/sumanw.txt line="hello everyone"'
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}

--------verify 
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
i am prepaired for rhce exam
hello everyone
192.168.105.5 | CHANGED | rc=0 >>
i am prepaired for rhce exam
hello everyone
192.168.105.4 | CHANGED | rc=0 >>
i am prepaired for rhce exam
hello everyone
suman1@ansible-server:~$ 

------------------------------------------
2. add line in the frist row 

-  ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="hello  sampa and everyone" insertafter=BOF'
-  suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
------------------------------------------
3. add line in the end of a file 

- suman1@ansible-server:~$ ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="thanks" insertafter=EOF'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks
192.168.105.5 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks
192.168.105.4 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks
suman1@ansible-server:~$ 

-------------------------------------------
4. write any line in after a specific line ---->

suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks
192.168.105.4 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks
192.168.105.5 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
hello everyone
thanks

suman1@ansible-server:~$ ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="and hope i passe the exam" insertafter=exam'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "msg": "line added"
}
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
thanks
192.168.105.5 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
thanks
192.168.105.4 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
thanks
suman1@ansible-server:~$ 
----------------------------------------------
5. remove any line ------------->

suman1@ansible-server:~$ ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt line="thanks" state=absent'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "found": 1,
    "msg": "1 line(s) removed"
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "found": 1,
    "msg": "1 line(s) removed"
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": true,
    "found": 1,
    "msg": "1 line(s) removed"
}
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
192.168.105.5 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
192.168.105.4 | CHANGED | rc=0 >>
hello  sampa and everyone
i am prepaired for rhce exam
and hope i passe the exam
hello everyone
suman1@ansible-server:~$ 

---------
6. delete line that  start or have hello word in line 

---->

suman1@ansible-server:~$ ansible liu -m lineinfile -a 'dest=/tmp/sumanw.txt regexp=^hello state=absent'
192.168.105.6 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": false,
    "found": 0,
    "msg": ""
}
192.168.105.5 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": false,
    "found": 0,
    "msg": ""
}
192.168.105.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "backup": "",
    "changed": false,
    "found": 0,
    "msg": ""
}
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sumanw.txt"
192.168.105.6 | CHANGED | rc=0 >>
i am prepaired for rhce exam
and hope i passe the exam
192.168.105.5 | CHANGED | rc=0 >>
i am prepaired for rhce exam
and hope i passe the exam
192.168.105.4 | CHANGED | rc=0 >>
i am prepaired for rhce exam
and hope i passe the exam
suman1@ansible-server:~$ 

----------------------------------
===================================
====================================
------------------------------------

replace module --------->

Definition: The Ansible replace module is used to find text in a file using a regular expression and replace all matching occurrences with the specified text without rewriting the entire file. 

replace hello to hi in sampa.txt

suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sampa.txt"
192.168.105.6 | CHANGED | rc=0 >>
this is a text line
hello everyone
192.168.105.5 | CHANGED | rc=0 >>
this is a text line
hello everyone
192.168.105.4 | CHANGED | rc=0 >>
this is a text line
hello everyone
suman1@ansible-server:~$ ansible liu -m replace -a 'path=/tmp/sampa.txt regexp=^hello replace=hi'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": "1 replacements made",
    "rc": 0
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": "1 replacements made",
    "rc": 0
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "msg": "1 replacements made",
    "rc": 0
}
suman1@ansible-server:~$ ansible liu -m command -a "cat /tmp/sampa.txt"
192.168.105.6 | CHANGED | rc=0 >>
this is a text line
hi everyone
192.168.105.5 | CHANGED | rc=0 >>
this is a text line
hi everyone
192.168.105.4 | CHANGED | rc=0 >>
this is a text line
hi everyone
suman1@ansible-server:~$ 

----------------------------
===================================
====================================
-------------------------------------
user module --------------->

Definition: The Ansible user module is used to create, modify, and delete user accounts on managed nodes.

--------------
create a user called user3 with uid 2020 groups=wheel 
---------------->

ansible suman -m user -a 'name=user3 state=present uid=2020 groups=wheel'
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "comment": "",
    "create_home": true,
    "group": 2020,
    "groups": "wheel",
    "home": "/home/user3",
    "name": "user3",
    "shell": "/bin/bash",
    "state": "present",
    "system": false,
    "uid": 2020
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "comment": "",
    "create_home": true,
    "group": 2020,
    "groups": "wheel",
    "home": "/home/user3",
    "name": "user3",
    "shell": "/bin/bash",
    "state": "present",
    "system": false,
    "uid": 2020
}

suman1@ansible-server:~$ ansible suman -m command -a 'id -a user3' 
192.168.105.5 | CHANGED | rc=0 >>
uid=2020(user3) gid=2020(user3) groups=2020(user3),10(wheel)
192.168.105.4 | CHANGED | rc=0 >>
uid=2020(user3) gid=2020(user3) groups=2020(user3),10(wheel)


---------------------------------------------------------------------
delete user3--------------->

suman1@ansible-server:~$ ansible suman -m user -a 'name=user3 state=absent'
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "force": false,
    "name": "user3",
    "remove": false,
    "state": "absent"
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "force": false,
    "name": "user3",
    "remove": false,
    "state": "absent"
}

suman1@ansible-server:~$ ansible suman -m command -a 'id -a user3' 
192.168.105.5 | FAILED | rc=1 >>
id: ‘user3’: no such usernon-zero return code
192.168.105.4 | FAILED | rc=1 >>
id: ‘user3’: no such usernon-zero return code
suman1@ansible-server:~$ 

----------------------------------------------------------------------
===================================================================
==========================================================
-----------------------------------------------------------------
Group modules --------------->

Definition: The Ansible group module is used to create, modify, and delete groups on managed nodes.



add a group called suh with gid 1020 

suman1@ansible-server:~$ ansible liu -m group -a 'name=suh state=present gid=1020'
192.168.105.6 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "gid": 1020,
    "name": "suh",
    "state": "present",
    "system": false
}
192.168.105.5 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "gid": 1020,
    "name": "suh",
    "state": "present",
    "system": false
}
192.168.105.4 | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": true,
    "gid": 1020,
    "name": "suh",
    "state": "present",
    "system": false
}
suman1@ansible-server:~$ ansible liu -m command -a 'tail -1 /etc/group'
192.168.105.6 | CHANGED | rc=0 >>
suh:x:1020:
192.168.105.5 | CHANGED | rc=0 >>
suh:x:1020:
192.168.105.4 | CHANGED | rc=0 >>
suh:x:1020:
suman1@ansible-server:~$ 

------------------------------

delete suh 
 ansible liu -m group -a 'name=suh state=absent'

---------------------------------------------------------------------
====================================================================
==================================================================
---------------------------------------------------------------------
12 . yum/dnf/Apt module :

Definition: The Ansible yum, dnf, and apt modules are used to install, update, and remove software packages on managed nodes.
- yum — used mainly on older RHEL/CentOS systems.
- dnf — used on modern RHEL/Fedora systems.
- apt — used on Debian/Ubuntu systems.


1. install httpd in suman by using dnf 

option 1 - ansible suman -m dnf -a "name=httpd state=present" # wen can also write state = install both are same
option 2 - ansible suman -m package -a 'name=ksh state=installed use=dnf'
2. install apachi2 in ubantu by using apt

option 1 - ansible ubantu -m apt -a "name=apache2 state=present"  
option 2 - ansible ubantu -m package -a 'name=apache2 state=present use=apt

------------------------------------------------------------------------
```
