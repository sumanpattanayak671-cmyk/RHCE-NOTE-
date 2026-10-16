# SESSION 6 — Ansible System Information, Mounts, and Services

> **Preservation note:** The complete original `Session 6.txt` content is included verbatim at the end of this document. All original command lines and terminal output are retained. The explanations and safety notes below are additional material.

## Contents
1. `stat` module
2. `mount` module
3. `setup` module and facts
4. `service` module
5. `systemd` module
6. Important command checks and safety notes
7. Quick revision table
8. Original Session 6 notes — preserved verbatim

---

## 1. `stat` module

### Definition
The Ansible `stat` module retrieves information about a file or directory on a managed node. It can report whether the path exists, whether it is a file or directory, its permissions, owner, group, size, and other metadata.

### Example from the notes
```bash
ansible liu -m stat -a "path=/etc/passwd"
```

### Breakdown
- `ansible`: runs an ad-hoc task from the control node.
- `liu`: inventory group or host pattern.
- `-m stat`: selects the `stat` module.
- `path=/etc/passwd`: path to inspect on every selected managed node.

**Use case:** Check whether a configuration file exists and inspect its permissions before editing it. The module gathers metadata; it does not change the file.

---

## 2. `mount` module

### Definition
The Ansible `mount` module configures filesystems and mount points. It can mount or unmount a filesystem and manage entries in `/etc/fstab` so the mount can persist across reboots. The notes identify it as part of the `ansible.posix` collection.

### Install the collection
```bash
ansible-galaxy collection install ansible.posix
```

This installs the collection that provides the `ansible.posix.mount` module. Depending on the Ansible installation and collection configuration, the fully qualified module name `ansible.posix.mount` may be used.

### Example in the notes: mount `/dev/sda` at `/opt` as XFS
The original commands are retained exactly in the source section. The intended sequence is to prepare the disk, create a filesystem, configure the mount, mount it, and verify the result.

Commands from the notes:
```bash
ansible node2 -m mount -a 'path=/opt src=dev/sda fstype=xfs opts=defaults state=present'
ansible node2 -m command -a 'cat /etc/fstab'
ansible node2 -m command -a 'df -Th'
ansible node2 -m command -a 'mkfs.xfs /dev/sda'
ansible node2 -m mount -a 'path=/opt src=/dev/sda fstype=xfs opts=defaults state=mounted'
ansible node2 -m command -a 'df -Th'
```

### Parameters
- `path=/opt`: directory used as the mount point.
- `src=/dev/sda`: device or filesystem source to mount.
- `fstype=xfs`: filesystem type.
- `opts=defaults`: standard mount options.
- `state=present`: ensure the mount is configured in `/etc/fstab` but do not necessarily mount it immediately.
- `state=mounted`: ensure the filesystem is mounted and configured persistently.

### Important safety warning
**`mkfs.xfs /dev/sda` formats the selected device and can destroy existing data.** Before running any formatting command, confirm that `/dev/sda` is the intended, correct, unused disk and that no needed data is stored on it. Device names can differ between machines. Never copy a disk-formatting command blindly to production systems.

The first original mount command uses `src=dev/sda` without the leading `/`. For a device path, `/dev/sda` is normally the intended form. The original command is preserved unmodified at the end.

### Verify the mount
```bash
ansible node2 -m command -a 'cat /etc/fstab'
ansible node2 -m command -a 'df -Th'
```
- `cat /etc/fstab` displays persistent mount configuration.
- `df -Th` displays mounted filesystems, filesystem types, and human-readable sizes.

### Unmount a filesystem and remove its persistent entry
Commands from the notes:
```bash
ansible node2 -m mount -a 'path=/opt src=/dev/sda state=unmounted'
ansible node2 -m command -a 'df -Th'
ansible node2 -m mount -a 'path=/opt src=/dev/sda state=absent_from_fstab'
ansible node2 -m command -a 'cat /etc/fstab'
```
- `state=unmounted`: unmounts the filesystem but leaves the `/etc/fstab` entry in place.
- `state=absent_from_fstab`: removes the corresponding persistent mount entry; it is not a substitute for unmounting a currently mounted filesystem.
- `df -Th` and `cat /etc/fstab` are used to verify the result.

Unmounting `/opt` can affect applications or files stored under that directory. Check the target host and mount point before running these tasks.

---

## 3. `setup` module

### Definition
The Ansible `setup` module gathers **facts** about managed nodes. Facts include operating-system details, hostnames, network interfaces and addresses, memory, processors, and kernel information.

### Gather all facts
```bash
ansible liu -m setup
```
This collects and displays the facts available from each host matched by `liu`. Large output is normal.

### Filter for selected facts

**Default IPv4 network information**
```bash
ansible liu -m setup -a "filter=ansible_default_ipv4"
```

**Operating-system distribution information**
```bash
ansible liu -m setup -a "filter=ansible_distribution*"
```

**Memory information**
```bash
ansible liu -m setup -a "filter=ansible_memory_mb"
```

**Kernel-related facts — example from the notes**
```bash
ansible liu -m setup -a "filter=*kernel*"
```

The original sample output shows different kernels on the hosts: one Ubuntu kernel and two RHEL 10.2-style kernels. The `discovered_interpreter_python` fact shows which Python interpreter Ansible discovered for that host.

### Facts in playbooks
By default, playbooks gather facts at the beginning when `gather_facts: true` is in effect (the usual default). Setting `gather_facts: false` disables that automatic gathering. Facts can then be referenced in playbooks, for example through variables such as `ansible_distribution` or `ansible_default_ipv4`.

**Exam tip:** `setup` gathers information; it is not intended to change system configuration. Use `filter=` to limit the output when looking for one category of facts.

---

## 4. `service` module

### Definition
The Ansible `service` module provides a common interface for controlling services. Depending on the service manager, it can start, stop, restart, reload, and enable or disable services.

### Start a service and enable it at boot
```bash
ansible suman -m service -a 'name=httpd state=started enabled=true'
```
- `name=httpd`: service name.
- `state=started`: ensure the service is running.
- `enabled=true`: enable the service to start automatically at boot.

### Restart a service
The original notes contain:
```bash
ansible suman -m service -a 'name=httpd state=restart'
```
For the Ansible service module, the documented state value is typically `restarted`, not `restart`. A runnable form is:
```bash
ansible suman -m service -a 'name=httpd state=restarted'
```
The original command remains unchanged in the source section.

**Platform note:** On RHEL-family systems, Apache's package and service are generally called `httpd`; on Ubuntu/Debian, the package and service are generally called `apache2`. Make sure the service exists on the selected nodes.

---

## 5. `systemd` module

### Definition
The Ansible `systemd` module manages services and systemd units. It supports actions such as starting, stopping, restarting, reloading, enabling, and disabling units. The source notes use `systemd`; in current Ansible documentation, the fully qualified name is commonly `ansible.builtin.systemd` (and `ansible.builtin.systemd_service` is also available in modern Ansible versions).

### 5.1 Reload systemd's unit definitions
The original notes show:
```bash
ansible suman -m systemd -a 'daemon-reload=yes'
```
A common documented parameter spelling is `daemon_reload=yes` (underscore). The original example and output are retained below.

### 5.2 Start a service
```bash
ansible liu -b -m systemd -a "name=httpd state=started"
```
Ensures Apache is running. `-b` enables privilege escalation, which is usually required for service administration.

### 5.3 Stop a service
```bash
ansible liu -b -m systemd -a "name=httpd state=stopped"
```
Stops Apache.

### 5.4 Restart a service
```bash
ansible liu -b -m systemd -a "name=httpd state=restarted"
```
Restarts Apache. A restart can briefly interrupt connections.

### 5.5 Enable a service at boot
```bash
ansible liu -b -m systemd -a "name=httpd enabled=yes"
```
Configures Apache to start automatically during boot. Enabling a service does not necessarily start it immediately; use `state=started` as well if both outcomes are required.

### 5.6 Disable a service at boot
```bash
ansible liu -b -m systemd -a "name=httpd enabled=no"
```
Prevents Apache from starting automatically at boot. Disabling does not stop a service that is already running.

### 5.7 Start and enable a service together
```bash
ansible liu -b -m systemd -a "name=firewalld state=started enabled=yes"
```
Starts `firewalld` and enables it at boot. Be careful changing firewall services on remote hosts, because firewall changes can affect connectivity.

### 5.8 Reload a service
```bash
ansible liu -b -m systemd -a "name=httpd state=reloaded"
```
Requests that the service reload its configuration without a full restart, if the service supports reloading.

### 5.9 Check service status
```bash
ansible liu -m command -a "systemctl status sshd"
```
This uses the `command` module to run `systemctl status sshd` on the matched nodes. The command's exit status may be non-zero if the service is inactive or missing; read the output to understand the reason. On some systems, the SSH service is named `sshd`; on Ubuntu it is often named `ssh`.

### 5.10 Reload systemd after changing a unit file
```bash
ansible liu -b -m systemd -a "daemon_reload=yes"
```
This asks systemd to reread its unit definitions. It is useful after adding or changing a systemd unit file.

### 5.11 Reload unit definitions and restart SSH
```bash
ansible liu -b -m systemd -a "name=sshd daemon_reload=yes state=restarted"
```
Reloads systemd's unit definitions and restarts SSH. **Use care when restarting SSH over a remote connection:** a configuration error or service failure can lock you out. Validate the SSH configuration and keep an alternate console or recovery route available when possible.

---

## 6. Quick revision table

| Module | Main purpose | Example |
|---|---|---|
| `stat` | Inspect file/directory metadata | `ansible liu -m stat -a "path=/etc/passwd"` |
| `mount` | Manage mounts and `/etc/fstab` | `state=mounted`, `state=unmounted` |
| `setup` | Gather host facts | `ansible liu -m setup -a "filter=ansible_default_ipv4"` |
| `service` | Control services through a common interface | `state=started`, `enabled=true` |
| `systemd` | Manage systemd services and units | `state=restarted`, `daemon_reload=yes` |

## 7. RHCE lab reminders

1. Confirm the inventory pattern (`liu`, `node2`, or `suman`) matches the intended managed nodes.
2. Use privilege escalation (`-b`) when changing mounts, formatting filesystems, or managing privileged services.
3. Verify changes with commands such as `df -Th`, `cat /etc/fstab`, `systemctl status`, or `setup` filters.
4. Treat formatting disks, unmounting filesystems, changing firewalls, and restarting SSH as potentially disruptive actions.
5. The source commands and their original outputs are preserved below even where an additional note explains a likely typo or safer documented form.

---

# 8. Original Session 6 notes — preserved verbatim

The following block contains the uploaded file's complete original text, without modifying command lines or output.

```text
Session 6

-----------------------------------
State module :-------------------

Definition: The Ansible stat module is used to check the status and retrieve information about files and directories on managed nodes, such as their existence, permissions, ownership, and size.

------

ansible liu -m stat -a "path=/etc/passwd"

------------------------------------------
------------------------------------------
============================================

mount module --------->

Definition: The Ansible mount module is used to mount, unmount, and configure filesystems on managed nodes, including making mounts persistent across reboots.
It is provided by the ansible.posix collection

to install mount module

- ansible-galaxy collection install ansible.posix

1. in node2 we have sda mount it on /opt directory in xfs

- ansible node2 -m mount -a 'path=/opt src=dev/sda fstype=xfs opts=defaults state=present'
- ansible node2 -m command -a 'cat /etc/fstab'
- ansible node2 -m command -a 'df -Th'
- ansible node2 -m command -a 'mkfs.xfs /dev/sda'
- ansible node2 -m mount -a 'path=/opt src=dev/sda fstype=xfs opts=defaults state=mounted'
- ansible node2 -m command -a 'df -Th'

2. for unmount it

- ansible node2 -m mount -a 'path=/opt src=/dev/sda state=unmounted'
- ansible node2 -m command -a 'df -Th'
- ansible node2 -m mount -a 'path=/opt src=/dev/sda state=absent_from_fstab'
- ansible node2 -m command -a 'cat /etc/fstab'

--------------------------------------------------------
=====================================================
=======================================================
---------------------------------------------------------

setup module -------->

Definition: The Ansible setup module is used to collect system information (facts) from managed nodes, such as IP addresses, operating systems, RAM, CPU, and hostname.

------
Example
ansible liu -m setup


This command collects and displays system facts from the managed host liu.
Useful examples
1. Display only network information
ansible liu -m setup -a "filter=ansible_default_ipv4"


2. Display operating system information
ansible liu -m setup -a "filter=ansible_distribution*"


3. Display memory information
ansible liu -m setup -a "filter=ansible_memory_mb"


Remember for RHCE: The setup module gathers facts automatically at the beginning of a playbook by default, unless fact gathering is disabled with gather_facts: false.

----------------------------------------------
suman1@ansible-server:~$ ansible liu -m setup -a "filter=*kernel*"
192.168.105.6 | SUCCESS => {
    "ansible_facts": {
        "ansible_kernel": "7.0.0-30-generic",
        "ansible_kernel_version": "#30-Ubuntu SMP PREEMPT_DYNAMIC Fri Jul 31 18:22:54 UTC 2026",
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
192.168.105.4 | SUCCESS => {
    "ansible_facts": {
        "ansible_kernel": "6.12.0-211.7.3.el10_2.x86_64",
        "ansible_kernel_version": "#1 SMP PREEMPT_DYNAMIC Fri May  8 22:40:33 EDT 2026",
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
192.168.105.5 | SUCCESS => {
    "ansible_facts": {
        "ansible_kernel": "6.12.0-211.7.3.el10_2.x86_64",
        "ansible_kernel_version": "#1 SMP PREEMPT_DYNAMIC Fri May  8 22:40:33 EDT 2026",
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}

----------------------------------------------------------------------
======================================================================
======================================================================
----------------------------------------------------------------------

service module ------------->

Definition: The Ansible service module is used to start, stop, restart, and reload services, and to enable or disable them on managed nodes.


ansible suman -m service -a 'name=httpd state=started enabled=true'

 ansible suman -m service -a 'name=httpd state=restart'

----------------------------------------------------------
==========================================================
==========================================================
----------------------------------------------------------
systemd module------------------>

Definition: The Ansible systemd module is used to start, stop, restart, reload, and enable or disable systemd services on managed nodes.

- ansible suman -m systemd -a 'daemon-reload=yes'
192.168.105.5 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "name": null,
    "status": {}
}
192.168.105.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "name": null,
    "status": {}
}
suman1@ansible-server:~$

1. Start a service
Start the Apache web server.
ansible liu -b -m systemd -a "name=httpd state=started"


2. Stop a service
ansible liu -b -m systemd -a "name=httpd state=stopped"


Stops the Apache service.
3. Restart a service
ansible liu -b -m systemd -a "name=httpd state=restarted"


Restarts Apache.
4. Enable a service at boot
ansible liu -b -m systemd -a "name=httpd enabled=yes"


Configures Apache to start automatically when RHEL boots.
5. Disable a service at boot
ansible liu -b -m systemd -a "name=httpd enabled=no"


Prevents Apache from starting automatically at boot. It does not stop an already running service.
6. Start and enable a service together
ansible liu -b -m systemd -a "name=firewalld state=started enabled=yes"


Starts the firewall and enables it at boot.
7. Reload a service
ansible liu -b -m systemd -a "name=httpd state=reloaded"


Reloads the service configuration without a full restart, if supported by the service.
8. Check the status of a service
The systemd module is primarily for managing services. To inspect service status, use the command module:
ansible liu -m command -a "systemctl status sshd"


9. Reload the systemd manager configuration
After adding or changing a systemd unit file:
ansible liu -b -m systemd -a "daemon_reload=yes"


This tells systemd to reread its unit files.
10. Restart a service after reloading unit files
ansible liu -b -m systemd -a "name=sshd daemon_reload=yes state=restarted"
This reloads systemd's unit definitions and restarts SSH. Use care when restarting SSH on a remote server.

------------------------------------------
---------------------------------------------
--------------------------------------------
--------------------------------------------
```
