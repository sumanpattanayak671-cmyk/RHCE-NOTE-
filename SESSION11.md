# SESSION 11 — Ansible Facts, Custom Facts, and Magic Variables

## Learning objectives

By the end of this session, you should be able to:

- Explain Ansible facts and where they come from.
- Use facts in a playbook to configure managed nodes.
- Understand `ansible_kernel`, `ansible_default_ipv4.address`, `ansible_fqdn`, and `ansible_hostname`.
- Control automatic fact gathering.
- Create and read custom facts with `ansible_local`.
- Use custom facts to install and manage a service.
- Recognize common Ansible magic variables and use `inventory_hostname` in a task.

---

## 1. Ansible facts

**Ansible facts** are information collected about managed hosts. They can describe a host's operating system, kernel, network interfaces, IP addresses, hostname, memory, and other system details. Facts are commonly gathered by the `setup` module.

In a playbook, fact gathering is enabled by default unless it is disabled in the play or through configuration.

### Example: save each host's kernel version

The source session uses this playbook:

```yaml
---
- name: ansible facts example
  hosts: all
  tasks:
    - name: capture the kernal version
      copy:
        content: "{{ ansible_kernel }}"
        dest: /tmp/kernel_version.txt
```

**How it works**

- `hosts: all` targets all hosts selected from the inventory.
- `ansible_kernel` is a fact containing the kernel version of the managed host.
- The `copy` module writes that value into `/tmp/kernel_version.txt` on each managed host.
- `{{ ... }}` is Jinja2 variable syntax used to insert the value into the task.

After the playbook runs, check the file on every managed host:

```bash
ansible all -m command -a 'cat /tmp/kernel_version.txt'
```

The session output showed different kernel versions on `node3` and on `node1`/`node2`. That is expected when the hosts run different kernels.

> **Note:** The original session uses the spelling `kernal` in the task description; the usual spelling is `kernel`. This does not affect the variable name `ansible_kernel`.

---

## 2. Using facts to update `/etc/hosts`

The next playbook uses facts to create a host entry:

```yaml
---
- name: /etc/hosts entry
  hosts: all
  tasks:
    - name: add ip and fpdn
      lineinfile:
        line: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }} {{ ansible_hostname }}"
        dest: /etc/hosts
```

### Meaning of the variables

| Variable | Meaning |
|---|---|
| `ansible_default_ipv4.address` | IPv4 address associated with the host's default IPv4 interface/route |
| `ansible_fqdn` | Fully qualified domain name (FQDN) reported for the host |
| `ansible_hostname` | Hostname of the managed host |
| `lineinfile` | Ansible module used to ensure a line is present in a file |
| `dest: /etc/hosts` | File to update on the managed host |

A resulting entry may look like:

```text
10.195.74.152 node1.suman.local node1
```

The exact IP address and names depend on each managed host's configuration.

The source also shows this equivalent dictionary-style fact reference:

```yaml
line: "{{ ansible_default_ipv4['address'] }} {{ ansible_fqdn }} {{ ansible_hostname }}"
```

Both forms access the `address` key. Dot notation is concise; bracket notation can be useful when accessing dictionary keys.

### Important practical points

- Editing `/etc/hosts` usually requires elevated privileges. Run the playbook with `become: true` or use `ansible-playbook -b ...` when appropriate.
- The shown task does not provide `lineinfile` with a `regexp` or `state`. For repeated runs, a more controlled version can use a matching expression to avoid duplicate entries.
- Ensure that the gathered IP and host names are the values you actually want to publish in `/etc/hosts`.

For example, a more idempotent version is:

```yaml
---
- name: Ensure managed hosts have an /etc/hosts entry
  hosts: all
  become: true
  tasks:
    - name: Add or update this host's entry
      ansible.builtin.lineinfile:
        path: /etc/hosts
        regexp: "^{{ ansible_default_ipv4.address }}\\s"
        line: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }} {{ ansible_hostname }}"
        state: present
```

This example is an explanatory improvement; the original playbook is preserved at the end of this document.

---

## 3. Disabling automatic fact gathering globally

The session asks how to turn off fact gathering globally and shows this configuration:

```ini
[defaults]
gathering = explicit
```

The file path in the original notes is written as `/ete/ansible/ansible.cfg`; the usual configuration path on many Linux installations is `/etc/ansible/ansible.cfg`. The active configuration path can vary by installation, so confirm which file Ansible is using.

With `gathering = explicit`, Ansible does not automatically gather facts for every play. If a play needs facts, request them explicitly:

```yaml
---
- name: Gather facts only when requested
  hosts: all
  gather_facts: true
  tasks:
    - name: Show the kernel version
      ansible.builtin.debug:
        msg: "{{ ansible_kernel }}"
```

You can also control gathering per play:

```yaml
gather_facts: false
```

**Remember:** If fact gathering is disabled, facts such as `ansible_kernel` and `ansible_default_ipv4.address` may be unavailable unless you explicitly gather them (for example, by enabling gathering in the play or running the `setup` module).

---

## 4. Custom facts

**Custom facts** are facts that you define yourself for your infrastructure. The source session creates a local fact file on `node1`.

### Step 1: Create the facts directory and file on `node1`

Run on the managed node:

```bash
sudo mkdir -p /etc/ansible/facts.d
sudo vim /etc/ansible/facts.d/httpd.fact
```

The file content from the session is:

```ini
[basic]
package=httpd
service=httpd
state=started
enabled=true
```

This is INI-style data. The section name is `basic`, and it contains values that describe the desired HTTP server package and service state.

For local facts to be collected, the file must be in the configured local facts directory on the managed host. Files ending in `.fact` are commonly used for this purpose.

### Step 2: Read the custom facts from the control node

```bash
ansible node1 -m setup -a "filter=ansible_local"
```

The output contains a structure similar to:

```yaml
ansible_local:
  httpd:
    basic:
      enabled: "true"
      package: httpd
      service: httpd
      state: started
```

The hierarchy is important:

- `ansible_local` is the top-level namespace for local custom facts.
- `httpd` comes from the fact file name `httpd.fact`.
- `basic` is the INI section.
- `package`, `service`, `state`, and `enabled` are values inside that section.

Because these values come from an INI-style file, they may be represented as strings. In particular, `enabled=true` may be returned as the string `"true"` rather than a YAML Boolean.

---

## 5. Use custom facts to install and start `httpd`

The source first checks the state of the service and package on `node1`. The service is inactive, and the package query reports that `httpd` is not installed. It then runs a playbook that installs the package and starts/enables the service.

### Original playbook structure

```yaml
---
- name: install httpd
  hosts: node1
  tasks:
    - name: install httpd
      yum:
        name: "{{ ansible_facts['ansible_local']['httpd']['basic']['package'] }}"
        state: latest
    - name: start and enable httpd
      service:
        name: "{{ ansible_facts['ansible_local']['httpd']['basic']['service'] }}"
        state: "{{ ansible_facts['ansible_local']['httpd']['basic']['state'] }}"
        enabled: "{{ ansible_facts['ansible_local']['httpd']['basic']['enabled'] }}"
```

The variable paths read values from the custom fact file. This makes the package and service settings configurable through the fact file instead of hard-coding them in each task.

### Run and verify

The session runs:

```bash
ansible-playbook custom_facts.yml
```

Then it checks the package:

```bash
ansible node1 -b -m command -a "rpm -q httpd"
```

And checks the service:

```bash
ansible node1 -b -m command -a 'systemctl status httpd'
```

The successful output in the source shows the `httpd` package installed and the service `active (running)` and enabled.

### Recommended improvements

For modern playbooks, use fully qualified module names and `become: true`. The following version also explicitly gathers facts, which is important because the playbook depends on `ansible_local`:

```yaml
---
- name: Install and manage httpd using custom facts
  hosts: node1
  become: true
  gather_facts: true

  tasks:
    - name: Install the package specified in custom facts
      ansible.builtin.package:
        name: "{{ ansible_facts['ansible_local']['httpd']['basic']['package'] }}"
        state: present

    - name: Start and enable the service specified in custom facts
      ansible.builtin.service:
        name: "{{ ansible_facts['ansible_local']['httpd']['basic']['service'] }}"
        state: "{{ ansible_facts['ansible_local']['httpd']['basic']['state'] }}"
        enabled: "{{ ansible_facts['ansible_local']['httpd']['basic']['enabled'] | bool }}"
```

This is a recommended variant, not a replacement for the original notes. `state: present` installs the package if missing without requiring the newest version; use `latest` if you specifically want the latest available package version. The `| bool` filter converts the custom fact's string value into a Boolean for the `enabled` parameter.

### Troubleshooting note from the session

The initial `systemctl status httpd` check returned a non-zero status because the service was inactive. The `rpm -q httpd` check also failed because the package was not installed. After the playbook ran, both checks showed the expected installed/running state. A command returning `FAILED` does not always mean Ansible connectivity is broken; check the command's return code and output.

---

## 6. Ansible magic variables

**Magic variables** are built-in variables supplied by Ansible. They provide context about the inventory, hosts, play, execution mode, and Ansible version. They are not normally defined manually in `vars`.

The source session lists these variables:

| Magic variable | Purpose |
|---|---|
| `inventory_hostname` | Name of the host as defined in the inventory |
| `groups` | All inventory groups and their hosts |
| `hostvars` | Variables and facts associated with inventory hosts |
| `group_names` | Inventory groups to which the current host belongs |
| `play_hosts` | Hosts targeted by the current play (legacy/deprecated in newer usage; prefer current play-host variables where appropriate) |
| `ansible_play_hosts` | Hosts still active in the current play |
| `ansible_play_name` | Name of the current play |
| `ansible_check_mode` | Indicates whether check mode is enabled |
| `ansible_version` | Information about the running Ansible version |

### Example: display the inventory hostname

```yaml
---
- name: Display hostname
  hosts: all
  gather_facts: true

  tasks:
    - name: Show inventory hostname
      ansible.builtin.debug:
        msg: "Inventory hostname is {{ inventory_hostname }}"
```

If your inventory contains an alias such as:

```ini
node1 ansible_host=192.168.105.4
```

then `inventory_hostname` is `node1`, while `ansible_host` supplies the address Ansible connects to. These values do not have to be the same.

### Other useful examples

Show the groups containing the current host:

```yaml
- name: Show this host's groups
  ansible.builtin.debug:
    var: group_names
```

Show all hosts in a group:

```yaml
- name: Show web group members
  ansible.builtin.debug:
    var: groups['web']
```

Read a variable/fact for another inventory host:

```yaml
- name: Show node1's hostname fact
  ansible.builtin.debug:
    msg: "{{ hostvars['node1']['ansible_hostname'] }}"
```

The last example requires the relevant fact to have been gathered for `node1` in the current execution or otherwise be available. `hostvars` does not automatically guarantee that every host's facts are present.

---

## 7. Quick revision

- **Facts:** System information gathered from managed hosts, commonly by the `setup` module.
- **`ansible_kernel`:** Kernel version fact.
- **`ansible_default_ipv4.address`:** Default IPv4 address fact.
- **`ansible_fqdn`:** Fully qualified domain name fact.
- **`ansible_hostname`:** Hostname fact.
- **`gathering = explicit`:** Disables automatic fact gathering by default; gather facts explicitly when needed.
- **Custom facts:** User-defined local facts commonly stored in `/etc/ansible/facts.d/` and accessed under `ansible_local`.
- **`inventory_hostname`:** The inventory name/alias for the current host.
- **`groups`, `hostvars`, `group_names`:** Useful for accessing inventory and host context.
- **Idempotency:** A well-designed playbook should be safe to run repeatedly without creating unnecessary changes.

## 8. Practice tasks

1. Write a playbook that saves `ansible_distribution` and `ansible_kernel` into separate files under `/tmp` on all hosts.
2. Use `debug` to display `inventory_hostname`, `ansible_hostname`, and `ansible_fqdn` side by side.
3. Create `/etc/ansible/facts.d/web.fact` on a managed node with a section containing a package and service name, then retrieve it with the `setup` module.
4. Configure a playbook to install and enable the package/service specified in your custom facts.
5. Set `gather_facts: false` for a play and observe what happens when a task references `ansible_kernel`. Then explicitly gather facts and test again.
6. Use `groups` and `hostvars` to display the inventory members of a group and a hostname fact from another host.

---

# Original SESSION11 source text

The complete source text is reproduced below for reference. Commands, outputs, spelling, and formatting from the original notes are retained in this section.

```text
SESSION11

-------------
Ansible Facts:----------------------->




suman1@ansible-server:~/playbook$ ansible all -m command -a 'cat /tmp/kernel_version.txt'
node3 | CHANGED | rc=0 >>
7.0.0-30-generic
node1 | CHANGED | rc=0 >>
6.12.0-211.7.3.el10_2.x86_64
node2 | CHANGED | rc=0 >>
6.12.0-211.7.3.el10_2.x86_64
suman1@ansible-server:~/playbook$ cat facts.yml 
---
- name: ansible facts example
  hosts: all
  tasks:
    - name: capture the kernal version
      copy:
        content: "{{ ansible_kernel }}"
        dest: /tmp/kernel_version.txt
...
suman1@ansible-server:~/playbook$ 

----------------------------------------------------------------------

suman1@ansible-server:~/playbook$ cat fact2.yml 
---
- name: /etc/hosts entry
  hosts: all
  tasks:
    - name: add ip and fpdn 
      lineinfile:
         line: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }} {{ ansible_hostname }}"
         dest: /etc/hosts
...
suman1@ansible-server:~/playbook$ ansible all -m command -a 'cat /etc/hosts'
node3 | CHANGED | rc=0 >>
127.0.0.1 localhost
127.0.1.1 suman-VMware-Virtual-Platform

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
10.195.74.1 node3.suman.com node3
node2 | CHANGED | rc=0 >>
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# See hosts(5) for proper format and other examples:
# 192.168.1.10 foo.example.org foo
# 192.168.1.13 bar.example.org bar
10.195.74.50 node2.suman.local node2
node1 | CHANGED | rc=0 >>
# Loopback entries; do not change.
# For historical reasons, localhost precedes localhost.localdomain:
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
# See hosts(5) for proper format and other examples:
# 192.168.1.10 foo.example.org foo
# 192.168.1.13 bar.example.org bar
10.195.74.152 node1.suman.local node1
suman1@ansible-server:~/playbook$ 

-------------------------------------------------------
line: "{{ ansible_default_ipv4.address }} {{ ansible_fqdn }} {{ ansible_hostname }}"
line: "{{ ansible_default_ipv4['address'] }} {{ ansible_fqdn }} {{ ansible_hostname }}"
-------------------------------------------------------------------------------------------

how to off facts gadring globaly 

vim /ete/ansible/ansible.cfg
[defaults]
gathering = explicit
-------------------------------------------------------------------

costom facts

---

in node1 

root@node1:~# sudo mkdir -p /etc/ansible/facts.d

root@node1:~# vim /etc/ansible/facts.d/httpd.fact
root@node1:~# cat /etc/ansible/facts.d/httpd.fact 
[basic]
package=httpd
service=httpd
state=started
enabled=true

go back to controle node

suman1@ansible-server:~/playbook$ ansible node1 -m setup -a "filter=ansible_local"
node1 | SUCCESS => {
    "ansible_facts": {
        "ansible_local": {
            "httpd": {
                "basic": {
                    "enabled": "true",
                    "package": "httpd",
                    "service": "httpd",
                    "state": "started"
                }
            }
        },
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
suman1@ansible-server:~/playbook$ 

suman1@ansible-server:~/playbook$ ansible node1 -b -m command -a 'systemctl status httpd'
node1 | FAILED | rc=3 >>
○ httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; bad; preset: disabled)
     Active: inactive (dead) since Sat 2026-10-17 13:55:54 IST; 1min 31s ago
   Duration: 21h 9min 36.351s
 Invocation: ad9c24e8ec0b4475b21d2a5515c6bead
       Docs: man:httpd.service(8)
    Process: 88637 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=exited, status=0/SUCCESS)
   Main PID: 88637 (code=exited, status=0/SUCCESS)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
   Mem peak: 13.7M
        CPU: 1min 4.673s

Oct 16 16:46:16 node1.suman.local systemd[1]: Starting httpd.service - The Apache HTTP Server...
Oct 16 16:46:16 node1.suman.local (httpd)[88637]: httpd.service: Referenced but unset environment variable evaluates to an empty string: OPTIONS
Oct 16 16:46:16 node1.suman.local httpd[88637]: Server configured, listening on: port 80
Oct 16 16:46:16 node1.suman.local systemd[1]: Started httpd.service - The Apache HTTP Server.
Oct 17 13:55:53 node1.suman.local systemd[1]: Stopping httpd.service - The Apache HTTP Server...
Oct 17 13:55:54 node1.suman.local systemd[1]: httpd.service: Deactivated successfully.
Oct 17 13:55:54 node1.suman.local systemd[1]: Stopped httpd.service - The Apache HTTP Server.
Oct 17 13:55:54 node1.suman.local systemd[1]: httpd.service: Consumed 1min 4.673s CPU time, 13.7M memory peak.Warning: The unit file, source configuration file or drop-ins of httpd.service changed on disk. Run 'systemctl daemon-reload' to reload units.non-zero return code
suman1@ansible-server:~/playbook$ ansible node1 -b -m command -a "rpm -q httpd"
node1 | FAILED | rc=1 >>
package httpd is not installednon-zero return code
suman1@ansible-server:~/playbook$ ansible-playbook custom_facts.yml

PLAY [install httpd] ****************************************************************************

TASK [Gathering Facts] **************************************************************************
ok: [node1]

TASK [install httpd] ****************************************************************************
changed: [node1]

TASK [start and enable httpd] *******************************************************************
changed: [node1]

PLAY RECAP **************************************************************************************
node1                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ ansible node1 -b -m command -a "rpm -q httpd"
node1 | CHANGED | rc=0 >>
httpd-2.4.63-13.el10.x86_64
suman1@ansible-server:~/playbook$ ansible node1 -b -m command -a 'systemctl status httpd'
node1 | CHANGED | rc=0 >>
● httpd.service - The Apache HTTP Server
     Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; preset: disabled)
     Active: active (running) since Sat 2026-10-17 13:58:49 IST; 21s ago
 Invocation: 72d9e6ccadc844978689200afbf0bdf0
       Docs: man:httpd.service(8)
   Main PID: 115581 (httpd)
     Status: "Total requests: 0; Idle/Busy workers 100/0;Requests/sec: 0; Bytes served/sec:   0 B/sec"
      Tasks: 177 (limit: 22568)
     Memory: 13.4M (peak: 13.6M)
        CPU: 615ms
     CGroup: /system.slice/httpd.service
             ├─115581 /usr/sbin/httpd -DFOREGROUND
             ├─115582 /usr/sbin/httpd -DFOREGROUND
             ├─115583 /usr/sbin/httpd -DFOREGROUND
             ├─115584 /usr/sbin/httpd -DFOREGROUND
             └─115585 /usr/sbin/httpd -DFOREGROUND

Oct 17 13:58:48 node1.suman.local systemd[1]: Starting httpd.service - The Apache HTTP Server...
Oct 17 13:58:49 node1.suman.local (httpd)[115581]: httpd.service: Referenced but unset environment variable evaluates to an empty string: OPTIONS
Oct 17 13:58:49 node1.suman.local httpd[115581]: Server configured, listening on: port 80
Oct 17 13:58:49 node1.suman.local systemd[1]: Started httpd.service - The Apache HTTP Server.
suman1@ansible-server:~/playbook$ cat custom_facts.yml 
---
- name: install httpd
  hosts: node1
  tasks:
    - name: install httpd
      yum:
         name: "{{ ansible_facts['ansible_local']['httpd']['basic']['package'] }}"
         state: latest
    - name: start and enable httpd
      service:
         name: "{{ ansible_facts['ansible_local']['httpd']['basic']['service'] }}"
         state: "{{ ansible_facts['ansible_local']['httpd']['basic']['state'] }}"
         enabled: "{{ ansible_facts['ansible_local']['httpd']['basic']['enabled'] }}"

-------------------------------------------------------------------------------------------------------


magic variable :-

2. Important Ansible magic variables
Magic variable	Purpose
inventory_hostname	Name of the host in your inventory
groups	All hosts grouped by inventory group
hostvars	Variables and facts associated with inventory hosts
group_names	Groups to which the current host belongs
play_hosts	Hosts targeted by the current play
ansible_play_hosts	Hosts still active in the current play
ansible_play_name	Name of the current play
ansible_check_mode	Indicates whether check mode is enabled
ansible_version	Information about the running Ansible version
3. Practical examples
Example 1: Display the inventory hostname
---
- name: Display hostname
  hosts: all
  gather_facts: true

  tasks:
    - name: Show inventory hostname
      ansible.builtin.debug:
        msg: "Inventory hostname is {{ inventory_hostname }}"

-----------------------------------------------------------------------




```
