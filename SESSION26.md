# Session 26 — Ansible Navigator

> **Topic:** Installing and exploring Ansible Navigator, configuring inventory and settings, running a playbook, and reading execution results  
> **Lab controller:** `ansible-server`  
> **Inventory hosts:** `node1`, `node2`, and `node3`

---

## 1. Learning Objectives

By the end of this session, you should be able to:

- Explain what Ansible Navigator is and when it is useful.
- Install Ansible Navigator and Podman on the control node.
- Open the Ansible Navigator terminal interface and understand its main commands.
- Configure an inventory and `ansible-navigator.yml`.
- Run an Ansible playbook through Navigator.
- Switch between the interactive view and standard terminal output.
- Understand `register`, `debug`, the Ansible ping module, and a play recap.
- Recognize Python interpreter discovery warnings.
- Troubleshoot common installation, inventory, and playbook problems.

---

## 2. What Is Ansible Navigator?

**Ansible Navigator** is a command-line tool for running and exploring Ansible automation. It can provide an interactive terminal user interface (TUI), inspect Ansible configuration and collections, show execution-environment images, run playbooks, and review saved playbook artifacts.

Navigator is often used with **execution environments (EEs)**. An execution environment is a container image containing Ansible and the dependencies required to run automation. Podman can be used as the container engine for these images.

In this lab, the configuration explicitly sets `execution-environment.enabled: false`. Therefore, Navigator is configured to run without an execution environment, even though an image name and Podman engine are listed in the configuration.

### Ansible tools in this lab

| Tool | Purpose |
|---|---|
| Ansible | Runs automation against managed hosts |
| Ansible Navigator | Provides a Navigator interface for running and inspecting automation |
| Podman | Runs and manages containers; it can run execution environments |
| Inventory | Lists managed hosts and host groups |
| Playbook | YAML file describing the automation tasks |
| Execution environment | Container image that packages Ansible and required dependencies |
| Playbook artifact | Saved record of a Navigator playbook run, when enabled |

### Typical workflow

1. Install the required packages and Navigator.
2. Prepare an inventory with reachable managed hosts.
3. Configure Navigator settings.
4. Write and validate a playbook.
5. Run it with `ansible-navigator`.
6. inspect the output and recap.
7. Review artifacts and logs if needed.

---

## 3. Install the Required Packages

The commands in the original lab are:

```bash
sudo dnf install -y python-pip podman
pip3 install ansible-navigator --user
```

### Explanation

- `sudo` runs a command with administrative privileges.
- `dnf install` installs packages using DNF.
- `-y` automatically answers yes to package-manager confirmation prompts.
- `python-pip` provides the pip package installer on systems where that package name is available.
- `podman` installs the container engine.
- `pip3 install ansible-navigator --user` installs Navigator into the current user's Python environment rather than system-wide.

**Platform note:** Package names and installation methods differ by operating system and repository configuration. The commands above are the commands recorded in this session. If a package cannot be found, check the OS release and enabled repositories rather than repeatedly running the same command.

### Verify the installation

```bash
ansible-navigator --version
podman --version
podman images
```

`podman images` lists container images already present on the machine.

The session showed this image:

```text
REPOSITORY                                   TAG      IMAGE ID      CREATED      SIZE
ghcr.io/ansible/community-ansible-dev-tools  latest   8f181bd154f3  10 days ago  1.41 GB
```

The image list is specific to the lab machine and can differ on another system.

### If the command is not found

Because `pip --user` installs executables in a user-specific directory, the shell may not automatically find `ansible-navigator`.

Check the user installation location:

```bash
python3 -m pip show ansible-navigator
python3 -m site --user-base
```

For Bash, a common user executable directory is `~/.local/bin`. Check whether it is in `PATH`:

```bash
echo "$PATH"
ls -l ~/.local/bin/ansible-navigator
```

If the executable exists but is not found, temporarily add the directory for the current shell:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

To make the change persistent for Bash, add the same export line to `~/.bashrc`, then reload it:

```bash
source ~/.bashrc
```

Use the path that matches the actual location on your system.

---

## 4. Start Ansible Navigator

Run:

```bash
ansible-navigator
```

Navigator opens an interactive terminal interface. The welcome page lists helpful commands, including:

| Navigator command | Purpose |
|---|---|
| `:collections` | Explore available collections |
| `:config` | Explore the current Ansible configuration |
| `:doc <plugin>` | Read documentation for a module or plugin |
| `:help` | Display the help page |
| `:images` | Explore execution-environment images |
| `:inventory -i <inventory>` | Explore an inventory |
| `:log` | Review the Navigator application log |
| `:lint <file or directory>` | Lint Ansible/YAML files; availability may be experimental |
| `:open` | Open the current page in the editor |
| `:replay` | Explore a previous run using a playbook artifact |
| `:run <playbook> -i <inventory>` | Run a playbook interactively |
| `:settings` | Review Navigator settings |
| `:quit` | Quit Navigator |

For example, from the Navigator interface:

```text
:help
:config
:collections
:images
:inventory -i inventory
:doc ansible.builtin.ping
:quit
```

**Important:** These colon-prefixed commands are entered inside the Navigator interface. They are not ordinary shell commands. At the normal shell prompt, use commands such as `ansible-navigator run ping.yml`.

---

## 5. Prepare the Inventory

The lab inventory is:

```ini
[suman]
node1
node2

[ubuntu]
node3
```

Save it in a file named `inventory`.

### Understand the inventory

- `[suman]` defines a host group named `suman`.
- `node1` and `node2` belong to that group.
- `[ubuntu]` defines another group named `ubuntu`.
- `node3` belongs to that group.
- `all` is Ansible's implicit group that includes all inventory hosts.

The group name is simply a label; Ansible does not verify that the hosts in a group run that operating system. If `node3` is Ubuntu, the group name is descriptive. If the machine uses another OS, update the group name to avoid confusion.

### Confirm the inventory

```bash
cat inventory
ansible-inventory -i inventory --list
ansible-inventory -i inventory --graph
```

The short graph view should show both groups and their hosts.

### Host connection information

The original inventory uses host aliases (`node1`, `node2`, and `node3`). These names must resolve through DNS or `/etc/hosts`, and SSH access must be configured.

If needed, an inventory can define connection details explicitly:

```ini
[suman]
node1 ansible_host=192.0.2.11 ansible_user=ansible
node2 ansible_host=192.0.2.12 ansible_user=ansible

[ubuntu]
node3 ansible_host=192.0.2.13 ansible_user=ansible
```

The IP addresses above are **documentation-only examples** from the reserved `192.0.2.0/24` range. Replace them with your real lab addresses. Do not copy them as if they were your actual hosts.

---

## 6. Configure `ansible-navigator.yml`

The configuration from the session is:

```yaml
---
ansible-navigator:
  ansible:
    inventory:
      entries:
        - inventory

  execution-environment:
    container-engine: podman
    enabled: false
    image: quay.io/ansible/ansible-navigator-demo-ee:latest

  logging:
    level: debug

  playbook-artifact:
    enable: true
    save-as: playbook-artifacts/{playbook_name}-artifact-{time_stamp}.json
```

Save this as `ansible-navigator.yml` in the project directory from which you run Navigator.

### Explanation, line by line

#### `ansible:`

This section contains Ansible-related settings used by Navigator.

#### `inventory.entries`

```yaml
inventory:
  entries:
    - inventory
```

This tells Navigator to use the `inventory` file in the current project directory. If the file is stored elsewhere, provide the correct path.

#### `execution-environment`

```yaml
execution-environment:
  container-engine: podman
  enabled: false
  image: quay.io/ansible/ansible-navigator-demo-ee:latest
```

- `container-engine: podman` selects Podman as the container engine for execution environments.
- `enabled: false` disables execution-environment use in this configuration.
- `image` specifies the image to use if execution-environment mode is enabled.

Because `enabled` is `false`, the image setting does not mean that this playbook run is using that image. To test an execution environment, first confirm the image exists or can be pulled, then enable EE mode intentionally and verify compatibility with your environment.

#### `logging.level`

```yaml
logging:
  level: debug
```

Debug logging provides more diagnostic detail. It can be useful for troubleshooting but may produce a large amount of output.

#### `playbook-artifact`

```yaml
playbook-artifact:
  enable: true
  save-as: playbook-artifacts/{playbook_name}-artifact-{time_stamp}.json
```

- `enable: true` enables saving a playbook-run artifact.
- `save-as` defines a naming pattern for the artifact.
- `{playbook_name}` and `{time_stamp}` are placeholders expanded by Navigator.

Make sure the target directory exists if your setup requires it:

```bash
mkdir -p playbook-artifacts
```

Check the exact artifact path after a run; the actual path depends on Navigator version and configuration.

### Inspect Navigator's effective configuration

From the shell:

```bash
ansible-navigator --help
ansible-navigator --version
```

From inside the Navigator interface, use:

```text
:config
:settings
```

The interface can help you determine which settings Navigator has loaded.

---

## 7. Create the Ping Test Playbook

The playbook from the session is:

```yaml
---
- name: ping test playbook
  hosts: all
  tasks:
    - name: Testing ping
      ansible.builtin.ping:
      register: result

    - name: print
      ansible.builtin.debug:
        msg: "{{ result }}"
```

Save it as `ping.yml`.

### Understand each part

#### Playbook start

```yaml
---
```

This marks the start of a YAML document.

#### Play name

```yaml
- name: ping test playbook
```

The play name describes what the play does. The leading hyphen starts a list item, because a playbook is a list of plays.

#### Target hosts

```yaml
hosts: all
```

This runs the play against every host in the selected inventory. In this lab, that means `node1`, `node2`, and `node3`.

#### Tasks

```yaml
tasks:
```

A task is an individual automation action.

#### Ansible ping module

```yaml
- name: Testing ping
  ansible.builtin.ping:
  register: result
```

`ansible.builtin.ping` tests whether Ansible can communicate with the managed host and execute its module. It is **not** the same as sending an ICMP network ping. It normally relies on the configured connection method (commonly SSH) and a usable Python interpreter on the target.

`register: result` saves the task result in a variable called `result` for later tasks in the same play and host context.

#### Debug module

```yaml
- name: print
  ansible.builtin.debug:
    msg: "{{ result }}"
```

The debug module prints the contents of `result`. The expression `{{ result }}` is a Jinja2 expression that inserts the variable's value.

### A more concise alternative

If you only need to test connectivity, the registered result and debug task are optional:

```yaml
---
- name: Test Ansible connectivity
  hosts: all
  gather_facts: false
  tasks:
    - name: Check Ansible ping
      ansible.builtin.ping:
```

`gather_facts: false` skips the automatic fact-gathering step. This is useful for a quick connectivity check; omit it when you want facts for later tasks.

---

## 8. Run the Playbook with Ansible Navigator

### Interactive mode

The session runs:

```bash
ansible-navigator run ping.yml
```

Navigator shows a summary table with task counts and status. In the lab output, the play was marked complete with no failed or unreachable hosts.

In the interactive view, the session notes that you can press `0` to inspect the play's output. The exact keys and screen layout can vary by Navigator version.

### Standard output mode

To display traditional Ansible output in the terminal, use:

```bash
ansible-navigator run ping.yml -m stdout
```

Here, `-m stdout` selects the standard-output mode instead of the interactive TUI.

This is useful when you want readable terminal logs, want to paste output into notes, or are troubleshooting a task.

### Run with an explicit inventory

If you do not want to rely on the configuration file's inventory setting:

```bash
ansible-navigator run ping.yml -i inventory -m stdout
```

### Validate before running

```bash
ansible-playbook -i inventory ping.yml --syntax-check
```

If your Navigator setup supports the equivalent command workflow, use the appropriate Navigator options for your version. A syntax check catches YAML and playbook-structure errors but does not prove that every target is reachable or that every task will succeed.

---

## 9. Understand the Successful Output

The session output included:

```text
TASK [Testing ping] ************************************************************
ok: [node3]
ok: [node2]
ok: [node1]
```

This means the ping task completed successfully on each host.

The registered result printed by the debug task included:

```yaml
msg:
  changed: false
  failed: false
  ping: pong
```

### What the result fields mean

| Field | Meaning |
|---|---|
| `changed: false` | The task did not report a system change |
| `failed: false` | The task did not fail |
| `ping: pong` | The Ansible ping module returned its expected success response |

The ping module is designed to test Ansible module execution, so a successful result normally reports `pong` and no change.

### Understand the play recap

The session's recap was:

```text
PLAY RECAP *********************************************************************
node1 : ok=3 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
node2 : ok=3 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
node3 : ok=3 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

| Recap field | Meaning |
|---|---|
| `ok` | Number of tasks that completed successfully for that host, including successful unchanged tasks |
| `changed` | Number of tasks that reported changes |
| `unreachable` | Tasks that could not connect to the host |
| `failed` | Tasks that failed |
| `skipped` | Tasks skipped due to conditions or other skip behavior |
| `rescued` | Failures handled by a rescue block |
| `ignored` | Failures ignored using task settings |

In this example, `ok=3` includes fact gathering, the ping task, and the debug task. The exact task count can differ if facts are disabled or the playbook changes.

**Good result:** `unreachable=0` and `failed=0` for all intended hosts.

---

## 10. Understand the Python Interpreter Warning

The output included warnings similar to:

```text
[WARNING]: Host 'node3' is using the discovered Python interpreter
at '/usr/bin/python3.14' ...
```

This is an **interpreter discovery warning**, not a task failure. Ansible found a Python interpreter on the managed host and is warning that a future interpreter installation could change which interpreter is discovered.

The session showed Python paths for the hosts:

- `node1`: `/usr/bin/python3.12`
- `node2`: `/usr/bin/python3.12`
- `node3`: `/usr/bin/python3.14`

These paths are what the lab reported at that time; your paths may be different.

### Check Python on a managed host

```bash
ansible all -i inventory -m command -a "python3 --version"
ansible all -i inventory -m command -a "command -v python3"
```

The `command` module does not use shell pipes or shell expansions. For commands that require shell syntax, use `ansible.builtin.shell` only when necessary.

### Set the interpreter explicitly if appropriate

For a host that has a confirmed Python path, you can set it in inventory:

```ini
node1 ansible_python_interpreter=/usr/bin/python3
```

Or set it for a group:

```ini
[suman:vars]
ansible_python_interpreter=/usr/bin/python3
```

**Important:** Use a path that actually exists on that target. Do not force `/usr/bin/python3` or another path without verifying it. Interpreter configuration is often unnecessary if automatic discovery is working correctly.

---

## 11. Troubleshooting Checklist

### A. `ansible-navigator: command not found`

Check the installation and user executable directory:

```bash
python3 -m pip show ansible-navigator
python3 -m site --user-base
ls -l ~/.local/bin/ansible-navigator
echo "$PATH"
```

If the executable is under `~/.local/bin`, add that directory to `PATH` as described earlier.

### B. Inventory host cannot be resolved

Check the names and DNS resolution:

```bash
cat inventory
getent hosts node1
getent hosts node2
getent hosts node3
```

If name resolution fails, fix DNS or add correct lab addresses to `/etc/hosts` or the inventory.

### C. SSH connection fails

Test SSH to a target using the correct account:

```bash
ssh ansible@node1
```

Use the actual username and hostname for your environment. Check SSH keys, permissions, firewall rules, network reachability, and `ansible_user` settings.

### D. Ansible ping reports `UNREACHABLE`

Run a targeted test with verbose output:

```bash
ansible all -i inventory -m ansible.builtin.ping -vvv
```

Look for SSH errors, name-resolution failures, authentication problems, or timeouts.

### E. YAML syntax error

Run:

```bash
ansible-playbook -i inventory ping.yml --syntax-check
```

Check indentation. YAML uses spaces; avoid tabs. Ensure module names and task indentation are correct.

### F. Podman image does not appear

List images:

```bash
podman images
```

Remember that the current configuration has `execution-environment.enabled: false`, so a configured image may not be used by the run. If you intend to use an execution environment, check the image name, registry access, Podman availability, and Navigator configuration.

### G. No artifact is saved where expected

Check whether artifact saving is enabled and confirm the configured path:

```bash
grep -A4 -n "playbook-artifact" ansible-navigator.yml
find . -maxdepth 3 -type f -name '*artifact*.json'
```

Create the configured directory if necessary:

```bash
mkdir -p playbook-artifacts
```

The exact output location depends on Navigator's version and effective configuration.

---

## 12. Useful Ansible Navigator Commands

### Basic commands

```bash
ansible-navigator --version
ansible-navigator --help
ansible-navigator
ansible-navigator run ping.yml
ansible-navigator run ping.yml -m stdout
ansible-navigator run ping.yml -i inventory -m stdout
```

### Inventory and connectivity

```bash
ansible-inventory -i inventory --list
ansible-inventory -i inventory --graph
ansible all -i inventory --list-hosts
ansible all -i inventory -m ansible.builtin.ping
ansible suman -i inventory -m ansible.builtin.ping
ansible ubuntu -i inventory -m ansible.builtin.ping
```

### Configuration and troubleshooting

```bash
ansible-config dump --only-changed
ansible-config view
ansible-playbook -i inventory ping.yml --syntax-check
ansible-navigator run ping.yml -i inventory -m stdout -vv
ansible all -i inventory -m command -a "hostname"
ansible all -i inventory -m command -a "python3 --version"
```

### Podman commands

```bash
podman --version
podman info
podman images
podman ps -a
```

### Check logs and artifacts

Inside Navigator:

```text
:log
:settings
:config
:images
```

From the shell:

```bash
find . -maxdepth 3 -type f -name '*artifact*.json'
find . -maxdepth 3 -type f -iname '*log*'
```

### Quick reference

| Goal | Command |
|---|---|
| Open Navigator | `ansible-navigator` |
| Run playbook in TUI | `ansible-navigator run ping.yml` |
| Show standard output | `ansible-navigator run ping.yml -m stdout` |
| Choose inventory | `ansible-navigator run ping.yml -i inventory -m stdout` |
| List inventory graph | `ansible-inventory -i inventory --graph` |
| Ping all hosts | `ansible all -i inventory -m ansible.builtin.ping` |
| Check syntax | `ansible-playbook -i inventory ping.yml --syntax-check` |
| List container images | `podman images` |

---

## 13. Practice Tasks

Try to solve each task yourself before checking the solutions.

### Task 1 — Verify all inventory hosts

**Goal:** Display all hosts in the inventory and confirm that each host responds to Ansible.

**Requirements:**
1. Display the inventory graph.
2. Run the Ansible ping module against all hosts.

### Task 2 — Target one group

**Goal:** Run a ping test against only the `suman` group.

**Requirements:**
1. Do not run the test against `node3`.
2. Use the inventory file explicitly.

### Task 3 — Create a hostname playbook

**Goal:** Create `hostname.yml` that runs against all hosts and displays each host's hostname.

**Requirements:**
1. Use `ansible.builtin.command`.
2. Register the command result.
3. Print the hostname result with `ansible.builtin.debug`.

### Task 4 — Create a Python-version playbook

**Goal:** Create `python-version.yml` that reports the Python 3 version on each host.

**Requirements:**
1. Run on all hosts.
2. Register the result.
3. Display the result.
4. Use Navigator's standard output mode to run it.

### Task 5 — Validate a playbook

**Goal:** Check the syntax of `ping.yml` before running it.

**Requirements:**
1. Run a syntax check.
2. Run the playbook with Navigator in standard output mode.

### Task 6 — Inspect Navigator configuration

**Goal:** Review the configuration and confirm the inventory file is set.

**Requirements:**
1. Open Navigator.
2. Use its configuration view.
3. Quit Navigator.

### Task 7 — Find saved artifacts

**Goal:** Locate playbook artifacts generated by Navigator.

**Requirements:**
1. Ensure artifact saving is enabled in the YAML configuration.
2. Run `ping.yml`.
3. Search for generated JSON artifact files.

---

## 14. Practice Task Solutions

### Solution 1 — Verify all inventory hosts

```bash
ansible-inventory -i inventory --graph
ansible all -i inventory -m ansible.builtin.ping
```

Expected result: the graph shows the `suman` and `ubuntu` groups, and all three hosts return a successful ping response.

### Solution 2 — Target only the `suman` group

```bash
ansible suman -i inventory -m ansible.builtin.ping
```

Expected hosts: `node1` and `node2`. `node3` should not be targeted by this command.

### Solution 3 — `hostname.yml`

Create the file:

```yaml
---
- name: Display managed hostnames
  hosts: all
  gather_facts: false
  tasks:
    - name: Run hostname command
      ansible.builtin.command: hostname
      register: hostname_result
      changed_when: false

    - name: Display hostname
      ansible.builtin.debug:
        msg: "Hostname on {{ inventory_hostname }} is {{ hostname_result.stdout }}"
```

Run it:

```bash
ansible-playbook -i inventory hostname.yml
```

Or with Navigator:

```bash
ansible-navigator run hostname.yml -i inventory -m stdout
```

`changed_when: false` is appropriate here because reading a hostname does not change the system. `hostname_result.stdout` contains the command's standard output.

### Solution 4 — `python-version.yml`

Create the file:

```yaml
---
- name: Display Python 3 version
  hosts: all
  gather_facts: false
  tasks:
    - name: Check Python version
      ansible.builtin.command: python3 --version
      register: python_result
      changed_when: false

    - name: Print Python version
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ python_result.stdout }}"
```

Run it:

```bash
ansible-navigator run python-version.yml -i inventory -m stdout
```

Note: the command assumes `python3` is available on every target. If a host does not have it in its `PATH`, the task will fail on that host and should be investigated.

### Solution 5 — Validate and run `ping.yml`

```bash
ansible-playbook -i inventory ping.yml --syntax-check
ansible-navigator run ping.yml -i inventory -m stdout
```

The syntax check should report that the playbook syntax is OK. The actual run then tests host connectivity and module execution.

### Solution 6 — Inspect Navigator configuration

At the shell:

```bash
ansible-navigator
```

Inside the Navigator interface:

```text
:config
:settings
:quit
```

Review the displayed configuration for the inventory entry and other settings. The exact presentation may differ between versions.

### Solution 7 — Find saved artifacts

Check the configuration:

```bash
grep -A4 -n "playbook-artifact" ansible-navigator.yml
mkdir -p playbook-artifacts
```

Run the playbook:

```bash
ansible-navigator run ping.yml -i inventory -m stdout
```

Search for JSON artifacts:

```bash
find . -type f -name '*artifact*.json'
```

If no artifact appears, review Navigator's effective configuration and log, and check whether the configured artifact path is being applied by your installed version.

---

## 15. Extra Challenge — Build a Reusable Connectivity Check

Create `connectivity-report.yml`:

```yaml
---
- name: Generate a simple connectivity report
  hosts: all
  gather_facts: false
  tasks:
    - name: Test Ansible connectivity
      ansible.builtin.ping:
      register: ping_result

    - name: Display a readable result
      ansible.builtin.debug:
        msg: >-
          Host {{ inventory_hostname }} returned
          {{ ping_result.ping | default('no pong response') }}.
```

Check and run it:

```bash
ansible-playbook -i inventory connectivity-report.yml --syntax-check
ansible-navigator run connectivity-report.yml -i inventory -m stdout
```

**What to observe:**
- Each managed host should show `pong` if the task succeeds.
- If the ping task fails, Ansible normally stops executing later tasks for that host unless error handling is configured. The debug task may therefore not run for a failed host.
- The `default` filter provides a fallback if the variable exists but its `ping` field is missing; it does not automatically rescue a failed task.

---

## 16. Best Practices

1. **Keep inventory accurate.** Hostnames must resolve, and the connection username must be correct.
2. **Use fully qualified module names.** For example, use `ansible.builtin.ping` rather than relying on a short module name.
3. **Validate YAML before execution.** A syntax check is quick and catches many structural mistakes.
4. **Use standard output when debugging.** `-m stdout` makes task-by-task output easy to inspect.
5. **Use `register` only when later tasks need the result.** Avoid unnecessary variables and debug output in production playbooks.
6. **Use `changed_when: false` for read-only command tasks when appropriate.** This keeps recap output meaningful.
7. **Treat warnings separately from failures.** Investigate interpreter warnings, but check `failed` and `unreachable` status to understand whether the run actually failed.
8. **Do not enable an execution environment without checking the image and dependencies.** The session's configuration disables EE mode.
9. **Protect logs and artifacts.** They may contain hostnames, configuration details, or other operational information.
10. **Keep secrets out of plain-text playbooks.** Use Ansible Vault or an approved secrets-management system when credentials are needed.

---

## 17. Session 26 Summary

In this session, you installed Ansible Navigator and Podman, explored Navigator's interactive commands, prepared a three-host inventory, and configured `ansible-navigator.yml`. You created and ran `ping.yml` using both the interactive interface and standard-output mode.

The successful output showed `pong` for `node1`, `node2`, and `node3`, with zero failed and unreachable hosts. You also learned that Python interpreter discovery warnings are not necessarily task failures, and how to inspect inventory, configuration, output, and saved artifacts.

### Final revision checklist

- [ ] I can explain the purpose of Ansible Navigator.
- [ ] I can open and exit the Navigator interface.
- [ ] I can explain the main `ansible-navigator.yml` settings.
- [ ] I can read an inventory with groups and hosts.
- [ ] I can run a playbook using `ansible-navigator run`.
- [ ] I understand `-m stdout`.
- [ ] I understand `register`, `debug`, and the ping result.
- [ ] I can interpret `ok`, `changed`, `unreachable`, and `failed`.
- [ ] I can perform a syntax check and troubleshoot basic connectivity.
- [ ] I can locate playbook artifacts and review Navigator logs.

**End of Session 26 — Ansible Navigator**
