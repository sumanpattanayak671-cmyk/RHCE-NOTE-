# Ansible Ad-Hoc Commands and Modules Lab

Hands-on Ansible notes for a control node managing two RHEL 10 servers.

## Lab Environment

| Component | Details |
|---|---|
| Control node | `ansible-server` |
| Managed node 1 | `192.168.10.3` |
| Managed node 2 | `192.168.10.4` |
| Inventory group | `suman` |
| Example control-node user | `suman1` |

> Replace hostnames, IP addresses, usernames, and paths with the values in your own lab.

## 1. Discover Ansible Modules

List available modules:

```bash
ansible-doc -l
```

Read documentation for a specific module:

```bash
ansible-doc copy
ansible-doc command
ansible-doc shell
ansible-doc file
```

Useful modules covered in these notes:

- `copy` — copy files or create files from supplied content.
- `command` — run a command without shell parsing.
- `raw` — execute a command directly over SSH, without requiring Python on the managed node.
- `shell` — run commands through a shell, supporting pipes, redirection, and shell operators.
- `file` — manage files, directories, permissions, ownership, and links.
- `fetch` — copy files from managed nodes back to the control node.
- `get_url` — download a file from a URL to a managed node.
- `user` and `group` — manage users and groups.
- `package`, `yum`, and `dnf` — manage packages.
- `stat` and `setup` — inspect files and gather system facts.
- `lineinfile` and `replace` — manage text in files.

## 2. Copy a File to Managed Nodes

Create a file on the control node:

```bash
touch suman.txt
```

Copy it to `/tmp` on every host in the `suman` inventory group:

```bash
ansible suman -m copy -a 'src=suman.txt dest=/tmp/suman.txt'
```

Verify:

```bash
ansible suman -m command -a 'ls -l /tmp/suman.txt'
```

### Create or replace a file with content

```bash
ansible suman -m copy -a 'content="I am Suman" dest=/tmp/suman.txt'
```

Verify the content:

```bash
ansible suman -m command -a 'cat /tmp/suman.txt'
```

**Remember:** `src` normally refers to a file on the control node. The `copy` module transfers it to the destination on the managed node.

## 3. Copy Files to a Root-Owned Location

To copy a control-node file to `/opt`, use privilege escalation. If `become` is not enabled in your configuration, run:

```bash
ansible suman -m copy -a 'src=suman.txt dest=/opt/suman.txt' -b
```

If you specifically need to connect as `root` using an SSH password (only if root SSH login is permitted):

```bash
ansible suman -m copy -a 'src=suman.txt dest=/opt/suman.txt' -u root -k
```

- `-b` — use privilege escalation (become).
- `-u root` — connect using the remote root account.
- `-k` — prompt for the SSH password.

Prefer `-b` with a normal remote account and `sudo` where possible.

## 4. Enable Privilege Escalation by Default

To avoid typing `-b` for every command, edit the Ansible configuration used by your control node:

```bash
sudo vim /etc/ansible/ansible.cfg
```

Set the following options under `[privilege_escalation]`:

```ini
[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

Then test:

```bash
ansible suman -m command -a 'id'
```

If the configuration is active and sudo permissions are correct, the command should run with root privileges.

> **Security note:** Defaulting all commands to root is powerful and risky. Consider enabling `become` per play or per command when you do not want every task to run as root. Confirm which `ansible.cfg` Ansible is actually using with `ansible --version`.

## 5. Run a Command as Another User

Use `--become-user` to select a different target account:

```bash
ansible suman -m command -a 'id' -b --become-user sampa
```

This requires the account to exist on the managed nodes and the configured privilege-escalation method to allow switching to it.

## 6. Set File Permissions, Owner, and Group

Copy a file to `/root` and set its owner, group, and permissions:

```bash
ansible suman -m copy -a 'src=suman.txt dest=/root/suman.txt mode=0755 owner=sampa group=sampa' -b
```

Verify:

```bash
ansible suman -m command -a 'ls -l /root/suman.txt'
```

`mode=0755` means the owner can read, write, and execute; group and others can read and execute. For ordinary text files, `0644` is more typical. Use executable permissions only when appropriate.

## 7. Copy a File Already on a Managed Node

The `remote_src=yes` option tells `copy` that the source file is already on the managed node.

For example, copy `/etc/redhat-release` to `/tmp/redhat-release` on each managed node:

```bash
ansible suman -m copy -a 'src=/etc/redhat-release dest=/tmp/redhat-release remote_src=yes' -b
```

Verify:

```bash
ansible suman -m command -a 'ls -l /tmp/redhat-release'
```

This example copies a file **on each managed node**. It does not transfer a file from one managed node to another.

## 8. Create Backups When Replacing Files

Use `backup=yes` to save a backup when the destination file is changed:

```bash
ansible suman -m copy -a 'src=suman.txt dest=/root/app.txt backup=yes' -b
```

A backup is created when an existing destination is overwritten. If the destination does not exist yet, there is nothing to back up.

## 9. `command` Module

Use `command` for straightforward commands that do not need shell features.

Check how long the managed nodes have been running:

```bash
ansible suman -m command -a 'uptime'
```

Check disk devices:

```bash
ansible suman -m command -a 'lsblk'
```

### Important limitation

The `command` module does not interpret shell operators such as `;`, `|`, `>`, or `&&` as a shell would. For example, this will not run both commands as intended:

```bash
ansible suman -m command -a 'uptime ; lsblk'
```

Run the commands separately, or use `shell` when shell syntax is required.

The `command` module generally relies on Python being available on the managed node. If Python is unavailable, `raw` can be useful for bootstrapping or simple remote commands.

## 10. `raw` Module

`raw` sends a command directly over SSH and does not require the normal Ansible Python module subsystem on the managed node.

Run one command:

```bash
ansible suman -m raw -a 'uptime'
```

Run multiple shell commands:

```bash
ansible suman -m raw -a 'uptime ; lsblk'
```

Use `raw` mainly for bootstrapping or special cases. For normal administration, prefer purpose-built Ansible modules where possible.

## 11. `shell` Module

Use `shell` when a task needs shell features such as pipes, redirection, environment expansion, or shell operators.

Examples:

```bash
ansible suman -m shell -a 'uptime; lsblk'
ansible suman -m shell -a 'cat /etc/redhat-release | grep Red'
```

### Create and run a script

On the control node, create `test.sh`:

```bash
cat > test.sh <<'EOF'
#!/bin/bash
echo "Hello World"
EOF
```

Copy it to the managed nodes:

```bash
ansible suman -m copy -a 'src=test.sh dest=/root/test.sh mode=0755' -b
```

Run it:

```bash
ansible suman -m shell -a '/root/test.sh' -b
```

Expected output includes:

```text
Hello World
```

**Best practice:** Prefer the `command` module when shell features are not needed. Use `shell` only when necessary, and avoid passing untrusted input into shell commands.

## 12. `file` Module

The `file` module can create or remove files and directories, change permissions and ownership, and manage symbolic links.

### Create a directory

```bash
ansible suman -m file -a 'path=/tmp/data state=directory'
```

Verify:

```bash
ansible suman -m command -a 'ls -ld /tmp/data'
```

### Remove a directory

```bash
ansible suman -m file -a 'path=/tmp/data state=absent'
```

Verify that it is absent:

```bash
ansible suman -m command -a 'ls -ld /tmp/data'
```

The verification command returns an error when the path no longer exists; that is expected after removal.

### Create a directory with ownership and permissions

For a directory accessible to everyone with read, write, and execute permissions:

```bash
ansible suman -m file -a 'path=/tmp/data state=directory mode=0777 owner=root group=root' -b
```

> `mode=0777` grants full permissions to everyone and is generally unsafe for real systems. Use the minimum permissions required. For example, `0755` is a common directory mode.

## 13. `fetch` Module

`fetch` copies a file **from a managed node to the control node**. It is the reverse direction of the usual `copy` workflow.

Fetch the release file from each managed node into a local `backup` directory:

```bash
mkdir -p backup
ansible suman -m fetch -a 'src=/etc/redhat-release dest=backup/' -b
```

Ansible normally creates host-specific directories, so the resulting structure resembles:

```text
backup/
├── 192.168.10.3/
│   └── etc/
│       └── redhat-release
└── 192.168.10.4/
    └── etc/
        └── redhat-release
```

## 14. `get_url` Module

Use `get_url` to download a file from a URL **to the managed node**.

Example:

```bash
ansible suman -m get_url -a 'url=https://linux-training.be/linuxfun.pdf dest=/tmp/linuxfun.pdf' -b
```

This requires the managed nodes to have network access to the URL. Use a trusted HTTPS source and verify the file where appropriate.

## 15. Quick Reference

| Goal | Command / option |
|---|---|
| List modules | `ansible-doc -l` |
| Read module documentation | `ansible-doc copy` |
| Copy control-node file to managed node | `ansible suman -m copy -a 'src=file dest=/tmp/file'` |
| Create a file with content | `ansible suman -m copy -a 'content="Hello" dest=/tmp/hello.txt'` |
| Run a simple command | `ansible suman -m command -a 'uptime'` |
| Run shell syntax | `ansible suman -m shell -a 'uptime; lsblk'` |
| Run without Python module subsystem | `ansible suman -m raw -a 'uptime'` |
| Create a directory | `ansible suman -m file -a 'path=/tmp/data state=directory'` |
| Remove a path | `ansible suman -m file -a 'path=/tmp/data state=absent'` |
| Fetch remote file to control node | `ansible suman -m fetch -a 'src=/etc/redhat-release dest=backup/' -b` |
| Download URL to managed node | `ansible suman -m get_url -a 'url=https://example.com/file dest=/tmp/file' -b` |
| Enable privilege escalation for one command | Add `-b` |
| Become another account | Add `-b --become-user sampa` |

## 16. Verify the Inventory and Connectivity

Before running module examples, confirm that the `suman` group exists in your inventory and that both nodes are reachable:

```bash
ansible-inventory --graph
ansible suman -m ping
```

The Ansible `ping` module tests Ansible connectivity and Python/module execution; it is not the same as the ICMP network `ping` command.

## Learning Notes

- **Control node to managed node:** commonly use `copy`.
- **Managed node to control node:** use `fetch`.
- **Download from a URL to managed node:** use `get_url`.
- **Simple command without shell syntax:** use `command`.
- **Shell operators or pipes needed:** use `shell`.
- **Python unavailable on the managed node:** consider `raw` for bootstrap tasks.
- **Manage files and directories declaratively:** use `file`.
- **Root permissions required:** use `become` (`-b`) with correctly configured sudo privileges.
