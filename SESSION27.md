# SESSION 27 — Automating RHCSA Tasks with Ansible and Regular Expressions

> **Topic:** Using regular expressions with `grep` and Ansible's `lineinfile` module to manage configuration files, with a practical example for changing SELinux configuration.

---

## Learning Objectives

By the end of this session, you should be able to:

- Explain what a regular expression (regex) is and why it is useful.
- Search text using `grep`, including case-insensitive and extended-regex searches.
- Understand common regex symbols such as `^`, `$`, `*`, `+`, and `|`.
- Use Ansible's `lineinfile` module with `regexp` to manage a configuration line.
- Use Ansible variables to make playbooks reusable.
- Override a playbook variable from the command line using `-e`.
- Validate a playbook and verify the resulting configuration.

## 1. Introduction

Regular expressions are patterns used to match text. They are useful when a file contains many lines and you need to find a particular line or replace it reliably.

In this session, the examples use:

- `grep` to search text files on the command line.
- Ansible's `lineinfile` module to ensure that a matching line in `/etc/selinux/config` has a desired value.
- Ansible variables to avoid hard-coding the desired SELinux configuration value.

The lab examples target `node1` and use privilege escalation (`become: true`) because `/etc/selinux/config` is a system configuration file.

## 2. Lab Prerequisites

Before starting, confirm that:

1. Ansible is installed on the control node.
2. The managed host `node1` is defined in your inventory.
3. The control node can connect to `node1`.
4. The Ansible user has the required privilege-escalation permissions.
5. You have permission to edit the SELinux configuration file.

Useful checks:

```bash
ansible --version
ansible-inventory --list
ansible node1 -m ping
```

> Replace `node1` with the host or inventory group used in your own lab if needed.

## 3. Regular Expressions: Basic Idea

A regular expression describes a text pattern. A search tool or module checks text against that pattern.

For example, the regex `^SELINUX=` means:

- `^` — the beginning of a line.
- `SELINUX` — the literal text `SELINUX`.
- `=` — a literal equals sign.

It matches a configuration line such as:

```text
SELINUX=enforcing
```

It does not mean that the value must be `enforcing`; the pattern only identifies the line beginning with `SELINUX=`.

### Common regex symbols

| Symbol | Meaning | Example |
|---|---|---|
| `^` | Beginning of a line | `^Val` matches a line starting with `Val` |
| `$` | End of a line | `a$` matches a line ending in `a` |
| `*` | Zero or more occurrences of the preceding pattern | `o*` matches zero or more `o` characters |
| `+` | One or more occurrences of the preceding pattern in extended regex | `o+` matches one or more `o` characters |
| `\|` in extended-regex notation | Alternative, meaning “or” | `i|a` matches either `i` or `a` |
| `.` | Any single character in most regex modes | `s.m` can match `sam` |
| `[abc]` | One character from the listed set | `[123]` matches `1`, `2`, or `3` |

**Important:** Regex syntax can differ between basic and extended regular expressions. In `grep -E`, `|` and `+` are special operators without a preceding backslash. In traditional basic `grep`, some operators require escaping.

## 4. Practising with `grep`

`grep` searches files and prints lines matching a pattern.

### 4.1 Create a sample file

```bash
cat > names.txt <<'EOF'
sampa1
sampa2
sampa3
sampa4
sampa5
EOF

cat names.txt
```

Expected content:

```text
sampa1
sampa2
sampa3
sampa4
sampa5
```

### 4.2 Search for exact text within lines

```bash
grep 'sampa2' names.txt
```

Expected output:

```text
sampa2
```

By default, `grep` matches a pattern anywhere in a line. This is a substring search, not necessarily a match of the entire line.

### 4.3 Ignore case with `-i`

```bash
grep -i 'samPa2' names.txt
grep -i 'samPa' names.txt
```

Expected results:

- The first command matches `sampa2`.
- The second command matches all five lines because each contains `sampa`, regardless of letter case.

### 4.4 Search for a digit

```bash
grep '1' names.txt
grep '3' names.txt
```

The first command prints `sampa1`; the second prints `sampa3`.

### 4.5 Match either of two patterns

Create another file:

```bash
cat > list.txt <<'EOF'
tania
laoura
EOF
```

Using extended regular expressions:

```bash
grep -E 'i|a' list.txt
```

This matches lines containing either `i` or `a`. Both sample lines contain `a`, so both are printed.

With basic `grep`, the equivalent alternation can be written:

```bash
grep 'i\|a' list.txt
```

The exact support for escaped operators depends on the grep implementation; GNU `grep` supports this form.

### 4.6 Understand `*` and `+`

Create a test file:

```bash
cat > list2.txt <<'EOF'
ll
lol
lool
loool
EOF
```

Zero or more `o` characters:

```bash
grep -E 'o*' list2.txt
```

This pattern can match zero `o` characters, so it can match every line, including `ll`. In fact, `o*` can match an empty string, which is why it is usually too broad for a useful search by itself.

One or more `o` characters:

```bash
grep -E 'o+' list2.txt
```

Expected output:

```text
lol
lool
loool
```

The line `ll` is excluded because it contains no `o`.

### 4.7 Match the beginning and end of lines

Create a third file:

```bash
cat > list3.txt <<'EOF'
Tania
Laura
Valentina
Fleur
Floor
EOF
```

Lines ending in lowercase `a`:

```bash
grep 'a$' list3.txt
```

Expected output:

```text
Tania
Laura
Valentina
```

Lines beginning with `Val`:

```bash
grep '^Val' list3.txt
```

Expected output:

```text
Valentina
```

### 4.8 Helpful `grep` options

| Command | Purpose |
|---|---|
| `grep -i 'pattern' file` | Ignore letter case |
| `grep -n 'pattern' file` | Show matching line numbers |
| `grep -v 'pattern' file` | Show lines that do not match |
| `grep -E 'pattern' file` | Use extended regular expressions |
| `grep -w 'word' file` | Match a whole word |
| `grep -c 'pattern' file` | Count matching lines |
| `grep -r 'pattern' directory/` | Search recursively through files |

Examples:

```bash
grep -n 'sampa' names.txt
grep -v 'sampa3' names.txt
grep -c 'sampa' names.txt
grep -E '^sampa[1-3]$' names.txt
```

The last command matches complete lines containing `sampa1`, `sampa2`, or `sampa3`.

## 5. Using Ansible to Change SELinux Configuration

The session's main automation example uses Ansible's `lineinfile` module to manage the `SELINUX` setting in `/etc/selinux/config`.

### 5.1 What does `lineinfile` do?

The `ansible.builtin.lineinfile` module ensures that a particular line exists in a file. When `regexp` is provided, Ansible searches for a line matching that regular expression and replaces the matching line with the specified `line`.

Important parameters:

- `path` — file to edit.
- `regexp` — regex used to identify the existing line.
- `line` — desired replacement line.
- `become: true` — run tasks with elevated privileges where configured.

For this lab, the pattern is:

```yaml
regexp: '^SELINUX='
```

It identifies a line starting with `SELINUX=` regardless of the current value.

### 5.2 Playbook: `selinux.yml`

Create the file:

```bash
vim selinux.yml
```

Add:

```yaml
---
- name: Change SELinux configuration using a regular expression
  hosts: node1
  become: true

  tasks:
    - name: Set SELinux mode in the configuration file
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=permissive'
```

Save the file, then validate its syntax:

```bash
ansible-playbook selinux.yml --syntax-check
```

If the syntax is valid, Ansible reports the playbook name without a syntax error.

Run the playbook:

```bash
ansible-playbook selinux.yml
```

A successful task may show `changed` the first time because the configuration line was updated. A later run should generally show `ok` if the desired line is already present.

### 5.3 Verify the configuration

Inspect the file on the managed host:

```bash
ansible node1 -b -m command -a "grep '^SELINUX=' /etc/selinux/config"
```

Check SELinux status:

```bash
ansible node1 -b -m command -a "sestatus"
```

The lab output showed:

```text
Current mode: enforcing
Mode from config file: permissive
```

This difference is important: changing `/etc/selinux/config` changes the **configured mode**, but does not necessarily change the **current running mode immediately**.

On a typical RHEL system, the configured mode is applied during boot. To change the current mode temporarily, use the appropriate SELinux command and follow your system's policy and operational requirements. Do not assume that editing the configuration file alone immediately changes the running mode.

**Security note:** SELinux `permissive` mode logs policy denials but does not enforce them. Keep `enforcing` in production unless you have an approved reason to change it. Use permissive mode carefully for controlled troubleshooting.

## 6. Using a Variable in the Playbook

Hard-coding a value makes a playbook less flexible. A variable lets you reuse the same playbook with different desired values.

### 6.1 Playbook: `selinux2.yml`

Create or edit the file:

```bash
vim selinux2.yml
```

Use:

```yaml
---
- name: Manage SELinux configuration with a variable
  hosts: node1
  become: true

  vars:
    status: permissive

  tasks:
    - name: Set SELinux configuration
      ansible.builtin.lineinfile:
        path: /etc/selinux/config
        regexp: '^SELINUX='
        line: "SELINUX={{ status }}"
```

Explanation:

- `vars:` defines play-level variables.
- `status: permissive` is the default value for this playbook.
- `{{ status }}` is Ansible's variable interpolation syntax.
- The `line` value is rendered using the current value of `status`.

Check and run it:

```bash
ansible-playbook selinux2.yml --syntax-check
ansible-playbook selinux2.yml
```

### 6.2 Override the variable from the command line

You can override the playbook's default variable with `-e` (extra variables):

```bash
ansible-playbook selinux2.yml -e status=enforcing
```

This run uses `enforcing` instead of the default `permissive`.

Verify the configured value:

```bash
ansible node1 -b -m command -a "grep '^SELINUX=' /etc/selinux/config"
```

Expected configuration line after the override:

```text
SELINUX=enforcing
```

The command-line value takes precedence over the play-level default for this run.

## 7. Important Difference: Configured Mode vs Current Mode

The session's `sestatus` output demonstrates a common troubleshooting point.

| `sestatus` field | Meaning |
|---|---|
| `Current mode` | SELinux mode currently active in the running system |
| `Mode from config file` | Mode configured in `/etc/selinux/config` |

For example:

```text
Current mode: enforcing
Mode from config file: permissive
```

means the system is currently enforcing SELinux, but the configuration file specifies permissive mode for the next boot. A configuration edit does not prove that the live mode changed.

Always verify both the file and the running status when troubleshooting.

## 8. Ansible Idempotency

Ansible modules are designed to describe the desired state. With `lineinfile`, once the matching line has the requested value, running the same playbook again should not make another change.

Example workflow:

```bash
ansible-playbook selinux2.yml -e status=enforcing
ansible-playbook selinux2.yml -e status=enforcing
```

- First run: likely `changed` if the file needed updating.
- Second run: likely `ok` if the file already has the desired line.

This property is called **idempotency**: repeated runs converge on the same desired state rather than continually changing the system.

## 9. Troubleshooting

### Problem 1: The playbook reports a syntax error

Check indentation and YAML structure. YAML indentation matters; use spaces rather than tabs.

```bash
ansible-playbook selinux.yml --syntax-check
```

### Problem 2: The task fails with permission denied

Ensure the play has privilege escalation:

```yaml
become: true
```

Also confirm that the Ansible account is allowed to use privilege escalation on the managed host.

### Problem 3: The playbook runs but the wrong line is changed

Check the file and the pattern:

```bash
ansible node1 -b -m command -a "grep -n 'SELINUX' /etc/selinux/config"
```

The pattern `^SELINUX=` matches only lines beginning exactly with `SELINUX=`. It does not match a commented line such as `#SELINUX=enforcing` or a line with spaces before `SELINUX`.

### Problem 4: The configured value differs from the current mode

Use:

```bash
ansible node1 -b -m command -a "sestatus"
```

Remember that the configuration file and running SELinux state are different things.

### Problem 5: A variable override does not seem to work

Make sure the variable name is correct and pass the value using:

```bash
ansible-playbook selinux2.yml -e status=enforcing
```

Then inspect `/etc/selinux/config` on the managed host.

### Problem 6: Python interpreter discovery warning

Ansible may display a warning that it discovered a Python interpreter on a managed host. This is a warning, not necessarily a task failure. If needed, configure the intended interpreter in inventory after checking which Python version is supported on the target system.

## 10. Useful Commands — Quick Revision

### Text and regex practice

```bash
cat names.txt
grep 'sampa2' names.txt
grep -i 'SAMPA' names.txt
grep -n 'sampa' names.txt
grep -v 'sampa3' names.txt
grep -E '^sampa[1-3]$' names.txt
grep -E 'i|a' list.txt
grep -E 'o+' list2.txt
grep 'a$' list3.txt
grep '^Val' list3.txt
```

### Ansible checks and execution

```bash
ansible --version
ansible-inventory --list
ansible node1 -m ping
ansible-playbook selinux.yml --syntax-check
ansible-playbook selinux.yml
ansible-playbook selinux2.yml --syntax-check
ansible-playbook selinux2.yml
ansible-playbook selinux2.yml -e status=enforcing
```

### Verify SELinux configuration and status

```bash
ansible node1 -b -m command -a "grep '^SELINUX=' /etc/selinux/config"
ansible node1 -b -m command -a "sestatus"
```

## 11. Practice Tasks

Try these tasks on your lab before looking at the solutions.

1. Create `users.txt` with the names `sumit`, `suman`, `sampa`, `tania`, and `tanya`.
2. Use `grep -i` to search for `SUMAN` in `users.txt`.
3. Use a regex to find lines beginning with `tan`.
4. Use a regex to find lines ending in `a`.
5. Use `grep -E` to match either `suman` or `sampa`.
6. Create a playbook called `selinux-audit.yml` that runs against `node1` and displays the configured `SELINUX` line without changing it.
7. Modify `selinux2.yml` so the default variable is `enforcing`.
8. Run the playbook with `-e status=permissive` and verify the configured value.
9. Run the same playbook twice with the same variable and observe the `changed`/`ok` result.
10. Explain why `sestatus` may report `Current mode: enforcing` while the configuration file says `SELINUX=permissive`.

## 12. Practice Solutions

### Solution 1 — Create `users.txt`

```bash
cat > users.txt <<'EOF'
sumit
suman
sampa
tania
tanya
EOF
```

### Solution 2 — Case-insensitive search

```bash
grep -i 'SUMAN' users.txt
```

Expected match:

```text
suman
```

### Solution 3 — Lines beginning with `tan`

```bash
grep '^tan' users.txt
```

Expected matches:

```text
tania
tanya
```

### Solution 4 — Lines ending with `a`

```bash
grep 'a$' users.txt
```

Expected matches:

```text
suman
sampa
tania
tanya
```

### Solution 5 — Match either `suman` or `sampa`

```bash
grep -E '^(suman|sampa)$' users.txt
```

Expected matches:

```text
suman
sampa
```

The parentheses group the alternatives, and `^` / `$` require the whole line to match.

### Solution 6 — Audit the SELinux configuration

Create `selinux-audit.yml`:

```yaml
---
- name: Audit SELinux configuration
  hosts: node1
  become: true

  tasks:
    - name: Read the configured SELinux line
      ansible.builtin.command:
        cmd: grep '^SELINUX=' /etc/selinux/config
      register: selinux_config
      changed_when: false

    - name: Display configured SELinux mode
      ansible.builtin.debug:
        var: selinux_config.stdout
```

Run it:

```bash
ansible-playbook selinux-audit.yml
```

`changed_when: false` prevents the read-only command from being reported as a change.

### Solution 7 — Set the default to enforcing

In `selinux2.yml`, use:

```yaml
vars:
  status: enforcing
```

### Solution 8 — Override the default with permissive

```bash
ansible-playbook selinux2.yml -e status=permissive
ansible node1 -b -m command -a "grep '^SELINUX=' /etc/selinux/config"
```

Expected configuration line:

```text
SELINUX=permissive
```

### Solution 9 — Test idempotency

```bash
ansible-playbook selinux2.yml -e status=enforcing
ansible-playbook selinux2.yml -e status=enforcing
```

Observe the task result on each run. The first run may show `changed`; the second should show `ok` if the desired configuration is already present.

### Solution 10 — Explain the mode difference

`/etc/selinux/config` stores the configured mode for boot. `sestatus` also reports the mode active in the current running system. Editing the file alone does not necessarily change the active mode immediately, so the two values can differ until the system state is changed or the configuration is applied at boot.

## 13. Exam / Interview Questions

**Q1. What is a regular expression?**  
A regular expression is a pattern used to search for or match text.

**Q2. What does `^` mean in a regex?**  
It matches the beginning of a line.

**Q3. What does `$` mean?**  
It matches the end of a line.

**Q4. What is the difference between `grep -i` and `grep -E`?**  
`-i` ignores case; `-E` enables extended regular-expression syntax.

**Q5. What does `o+` mean in extended regex?**  
It matches one or more occurrences of `o`.

**Q6. What does `o*` mean?**  
It matches zero or more occurrences of `o`, so it can also match an empty string.

**Q7. What is Ansible's `lineinfile` module used for?**  
It ensures a particular line is present in a file and can replace a line selected by a regular expression.

**Q8. What does `regexp: '^SELINUX='` match?**  
A line beginning with `SELINUX=`.

**Q9. Why use `become: true`?**  
It enables privilege escalation for tasks that require elevated permissions, subject to the managed host's configuration.

**Q10. What does `-e status=enforcing` do?**  
It passes an extra variable named `status` with the value `enforcing` to the playbook run.

**Q11. What is idempotency in Ansible?**  
Repeatedly applying the same desired configuration should leave the system in that state without unnecessary additional changes.

**Q12. Why can `sestatus` show a different mode from `/etc/selinux/config`?**  
The file records the configured mode, while `sestatus` also reports the currently active mode.

## 14. Session Summary

In this session, you practised regular-expression searches with `grep` and applied regex matching in Ansible. The main automation pattern was:

```yaml
ansible.builtin.lineinfile:
  path: /etc/selinux/config
  regexp: '^SELINUX='
  line: "SELINUX={{ status }}"
```

The pattern identifies the existing `SELINUX=` line, while the `line` parameter defines the desired value. A variable makes the playbook reusable, and `-e status=enforcing` lets you override that value at runtime.

Remember to syntax-check playbooks, verify changes on the managed host, and distinguish the SELinux mode configured for boot from the mode currently active. Use SELinux permissive mode only when appropriate for your lab or approved troubleshooting procedure.
