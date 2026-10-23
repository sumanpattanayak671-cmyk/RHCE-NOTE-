# Session 23: Publishing Ansible Roles on GitHub and Ansible Galaxy

> **Goal:** Convert an Apache HTTPD configuration into a reusable Ansible role, test it, publish the role to GitHub, and import it into Ansible Galaxy.

## Table of Contents

1. [Learning Objectives](#learning-objectives)
2. [What Is an Ansible Role?](#what-is-an-ansible-role)
3. [The Original Apache Playbook](#the-original-apache-playbook)
4. [Create the Role Structure](#create-the-role-structure)
5. [Organize Tasks into Separate Files](#organize-tasks-into-separate-files)
6. [Create the Jinja2 Template](#create-the-jinja2-template)
7. [Add a Handler](#add-a-handler)
8. [Add Role Metadata](#add-role-metadata)
9. [Run the Role from a Playbook](#run-the-role-from-a-playbook)
10. [Test and Understand the Output](#test-and-understand-the-output)
11. [Install Git](#install-git)
12. [Initialize the Git Repository](#initialize-the-git-repository)
13. [Connect the Local Repository to GitHub](#connect-the-local-repository-to-github)
14. [Stage and Commit the Role](#stage-and-commit-the-role)
15. [Push the Role to GitHub](#push-the-role-to-github)
16. [Import the Role into Ansible Galaxy](#import-the-role-into-ansible-galaxy)
17. [Install the Published Role](#install-the-published-role)
18. [Important Corrections and Best Practices](#important-corrections-and-best-practices)
19. [Troubleshooting](#troubleshooting)
20. [Quick Revision](#quick-revision)
21. [Practice Questions](#practice-questions)

---

## 1. Learning Objectives

After completing this session, you should be able to:

- Explain why Ansible roles are useful.
- Create an Ansible role directory structure with `ansible-galaxy init`.
- Separate installation, service, and configuration tasks into individual YAML files.
- Use a Jinja2 template to generate a host-specific web page.
- Use handlers to restart a service only when a related task changes something.
- Test a role with `ansible-playbook`.
- Track role files using Git and publish them to GitHub.
- Import a GitHub-hosted role into Ansible Galaxy.
- Install a role using `ansible-galaxy role install`.

## 2. What Is an Ansible Role?

An **Ansible role** is a reusable way to organize automation content. Instead of writing every task inside one large playbook, a role groups related tasks, templates, files, handlers, variables, defaults, and metadata into a predictable directory structure.

For example, an Apache role can handle:

1. Installing the `httpd` package.
2. Starting and enabling the HTTPD service.
3. Deploying an HTML page from a template.
4. Allowing HTTP traffic through `firewalld`.

### Why use roles?

- **Reusability:** Use the same role on multiple servers or projects.
- **Organization:** Keep installation, service, and configuration tasks separate.
- **Maintainability:** Update one part of the role without making a large playbook difficult to manage.
- **Sharing:** Publish the role in a Git repository and make it available through Ansible Galaxy.

### Key terms

| Term | Meaning |
|---|---|
| Playbook | A YAML file that describes the automation to run on hosts. |
| Role | A structured collection of reusable Ansible automation content. |
| Task | One unit of work, such as installing a package. |
| Handler | A task normally triggered by a `notify` statement when another task reports a change. |
| Template | A Jinja2 file that Ansible renders using variables and facts. |
| Git | A version-control system that tracks changes to files. |
| GitHub | A hosting service for Git repositories. |
| Ansible Galaxy | A service for discovering and sharing Ansible roles and collections. |

---

## 3. The Original Apache Playbook

The session begins with a Jinja2 template and a playbook that configures Apache directly.

### 3.1 Create `index.j2`

Original template:

```html
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>
```

The expression `{{ ansible_hostname }}` is a Jinja2 variable. Ansible gathers this fact from the managed host, so each server can receive a page containing its own hostname.

For example, a page rendered on a host whose hostname is `node1` would display:

```html
<h1>The Apache web server is running on node1</h1>
```

The exact value depends on the managed host's gathered facts.

### 3.2 Create `apache.yml`

The original playbook:

```yaml
---
- name: Configure HTTPD and deploy a host-specific page
  hosts: suman
  become: true

  tasks:
    - name: Install HTTPD
      ansible.builtin.dnf:
        name: httpd
        state: latest

    - name: Start and enable HTTPD
      ansible.builtin.service:
        name: httpd
        state: started
        enabled: true

    - name: Render the web page
      ansible.builtin.template:
        src: /home/ansible/index.j2
        dest: /var/www/html/index.html
        owner: root
        group: root
        mode: "0644"

    - name: Allow HTTP through firewalld
      ansible.posix.firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: true

    - name: Restarting apachi HTTPD Service
      ansible.builtin.service:
        name: httpd
        state: restarted
```

### 3.3 Understand the important lines

- `hosts: suman` targets the inventory group or host pattern named `suman`. Make sure it exists in your inventory.
- `become: true` runs privileged tasks with privilege escalation, commonly as root.
- `ansible.builtin.dnf` manages packages on DNF-based systems.
- `state: latest` requests the latest available package version; `state: present` is often preferable when you only need the package installed without upgrading it.
- `ansible.builtin.service` manages service state.
- `ansible.builtin.template` renders a Jinja2 template on the controller and copies the result to the managed host.
- `mode: "0644"` sets read/write permission for the owner and read permission for group and others.
- `ansible.posix.firewalld` manages firewalld rules. The `ansible.posix` collection must be available on the controller, and firewalld must be installed and running on the target when required by the module.

> **Improvement:** The original playbook restarts HTTPD unconditionally. In a role, a handler is usually better: notify it only when a configuration or template task changes the server's files.

---

## 4. Create the Role Structure

The session creates an Apache role under `/etc/ansible/roles`.

### 4.1 Move to the roles directory

```bash
cd /etc/ansible/roles/
```

### 4.2 Initialize the role

The session's command:

```bash
sudo ansible-galaxy init /etc/ansible/roles/apache --offline
```

This creates the role skeleton without fetching content from the internet. If you are already in `/etc/ansible/roles`, the shorter form is:

```bash
sudo ansible-galaxy init apache --offline
```

A role skeleton looks like this:

```text
apache/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── README.md
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
└── vars/
    └── main.yml
```

### 4.3 Purpose of each directory

| Directory/file | Purpose |
|---|---|
| `tasks/main.yml` | Main entry point for the role's tasks. |
| `tasks/` | Task files such as `install.yml`, `service.yml`, and `config.yml`. |
| `handlers/main.yml` | Handlers that tasks can notify. |
| `templates/` | Jinja2 templates rendered by the `template` module. |
| `files/` | Static files copied without template rendering. |
| `defaults/main.yml` | Default variables intended to be easy to override. |
| `vars/main.yml` | Role variables with higher precedence than defaults. Use carefully when override flexibility is important. |
| `meta/main.yml` | Role metadata, including Galaxy information and optional dependencies. |
| `tests/` | Optional test inventory and test playbook. |
| `README.md` | Documentation explaining the role, requirements, variables, and usage. |

The role skeleton is only a starting point. You must put your actual tasks, templates, and metadata in the relevant files.

---

## 5. Organize Tasks into Separate Files

The session divides the work into three task files:

- `install.yml` — install Apache.
- `service.yml` — start and enable Apache.
- `config.yml` — render the page and configure the firewall.

### 5.1 `tasks/install.yml`

```yaml
---
- name: Install HTTPD
  ansible.builtin.dnf:
    name: httpd
    state: latest
```

**Explanation:** This task ensures the `httpd` package is installed or updated to the latest available version.

### 5.2 `tasks/service.yml`

```yaml
---
- name: Start and enable HTTPD
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

**Explanation:**

- `state: started` ensures the service is running.
- `enabled: true` configures the service to start automatically at boot.

### 5.3 `tasks/config.yml`

The session uses the following configuration tasks:

```yaml
---
- name: Render the web page
  ansible.builtin.template:
    src: templates/index1.j2
    dest: /var/www/html/index.html
    owner: root
    group: root
    mode: "0644"

- name: Allow HTTP through firewalld
  ansible.posix.firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true
```

**Explanation:**

1. The `template` task reads `templates/index1.j2` from the role.
2. Ansible renders the Jinja2 expressions and writes the result to `/var/www/html/index.html`.
3. The destination file is owned by root and has mode `0644`.
4. The firewalld task enables the predefined `http` service permanently and applies the change immediately.

**Important:** `src: templates/index1.j2` is resolved relative to the role's template directory by the `template` module. The file should therefore be located at `apache/templates/index1.j2`.

### 5.4 `tasks/main.yml`

The session's main task file imports the three task files:

```yaml
---
# tasks file for /etc/ansible/roles/apache
- ansible.builtin.import_tasks: install.yml
- ansible.builtin.import_tasks: service.yml
- ansible.builtin.import_tasks: config.yml
```

`import_tasks` is a static import. Ansible processes the imported task files as part of the playbook's task structure.

**Remember the spelling:** use `import_tasks` (plural), not `import_task`.

---

## 6. Create the Jinja2 Template

Create the template at:

```text
/etc/ansible/roles/apache/templates/index1.j2
```

Content:

```html
<html>
  <body>
    <center>
      <h1>The Apache web server is running on {{ ansible_hostname }}</h1>
    </center>
  </body>
</html>
```

### How it works

1. Ansible connects to a managed host and gathers facts.
2. The template module evaluates `{{ ansible_hostname }}` using that host's facts.
3. The rendered HTML is copied to `/var/www/html/index.html`.
4. When each host is configured, the page can display that host's hostname.

If fact gathering is disabled, or the required fact is unavailable, ensure the fact is gathered or provide an appropriate variable.

---

## 7. Add a Handler

The session defines a handler in `handlers/main.yml`:

```yaml
---
# handlers file for /etc/ansible/roles/apache
- name: Restarting apachi HTTPD Service
  ansible.builtin.service:
    name: httpd
    state: restarted
```

This defines a handler, but **a handler does not run merely because it exists**. A task must notify it.

### Recommended pattern

Add `notify` to the template task in `tasks/config.yml`:

```yaml
- name: Render the web page
  ansible.builtin.template:
    src: index1.j2
    dest: /var/www/html/index.html
    owner: root
    group: root
    mode: "0644"
  notify: Restarting apachi HTTPD Service
```

The handler runs at the end of the relevant task section if the template task reports a change. If the file is already correct, the template task normally reports `ok`, and the handler does not run.

For a static HTML page, an Apache restart is usually unnecessary; a reload or no service action may be sufficient. This handler is retained to explain the pattern used in the session.

> You can rename the handler to something clearer, such as `Restart HTTPD`, but the name in `notify` must exactly match the handler's name.

---

## 8. Add Role Metadata

The session edits:

```text
/etc/ansible/roles/apache/meta/main.yml
```

Example from the session:

```yaml
---
galaxy_info:
  author: SUMAN
  description: Apache httpd server config
  company: student at BWU
```

This metadata describes the role. For a role intended for Galaxy, review the generated metadata and fill in the fields required by the Galaxy version and role format you use. Useful fields may include the role name, supported platforms, license, and minimum Ansible version.

Metadata does not replace the README. The README should explain what the role does, how to install it, what variables it accepts, and how to use it.

---

## 9. Run the Role from a Playbook

The session creates `/etc/ansible/setup.yml`:

```yaml
---
- name: Apache Server Setup Playbook through roles
  hosts: suman
  become: true

  roles:
    - apache
```

### Explanation

- `hosts: suman` selects the target inventory group or host pattern.
- `become: true` enables privilege escalation.
- `roles: - apache` asks Ansible to execute the role named `apache`.
- Ansible finds `tasks/main.yml` and runs the task files imported from it in order.

Because the role is stored under `/etc/ansible/roles/apache`, Ansible can normally discover it through the standard role path. If your role is stored elsewhere, configure `roles_path` or specify an appropriate role path.

### Run the playbook

```bash
cd /etc/ansible
ansible-playbook setup.yml
```

Before running it, verify that:

- The inventory contains the `suman` host pattern or group.
- Managed hosts are reachable.
- Privilege escalation works.
- The `ansible.posix` collection is installed on the controller.
- The target systems support the selected package and service modules.
- Firewalld is configured as expected on the target hosts.

### Output shown in the session

The session shows facts gathered and Apache tasks completing on `node1` and `node2`, with zero failures. The recap reports `ok=5` and `changed=0` for each host.

An `ok` result means a task completed without reporting a change. A `changed` result means Ansible changed the target system. Zero failures indicates that no task failed during that run; it does not by itself prove that the web page is reachable from every network.

---

## 10. Test and Understand the Output

### 10.1 Check syntax

```bash
ansible-playbook setup.yml --syntax-check
```

This checks playbook syntax. It does not verify every runtime condition, network path, or remote service.

### 10.2 Run the playbook

```bash
ansible-playbook setup.yml
```

### 10.3 Check HTTPD on a managed host

```bash
ansible suman -b -m command -a "systemctl is-active httpd"
```

This checks whether HTTPD is active on hosts matching `suman`.

### 10.4 Check the generated page

On a managed host:

```bash
cat /var/www/html/index.html
```

Or, if HTTP connectivity is available, test the page from a machine that can reach the web server:

```bash
curl http://SERVER_IP/
```

Replace `SERVER_IP` with the actual address of a managed web server.

### 10.5 Check the firewall

On a target using firewalld:

```bash
sudo firewall-cmd --list-services
```

The `http` service should appear in the relevant zone if the role enabled it there. Ensure the target is using the expected firewalld zone and that network routing and upstream firewalls also allow the connection.

---

## 11. Install Git

The session installs Git on the Ansible controller:

```bash
sudo dnf install -y git
```

### Why Git is needed

Git tracks changes to the role's files. It allows you to create commits and push the role to a remote GitHub repository.

The session's package output shows Git version `2.52.0-1.el10` being installed on a RHEL 10 system, along with dependencies.

A subscription warning also appears:

```text
This system is not registered with an entitlement server.
```

The package installation nevertheless completes in the shown output. Repository access and registration requirements depend on the system's repository configuration; do not assume every unregistered RHEL system will have the same package access.

Verify Git:

```bash
git --version
```

---

## 12. Initialize the Git Repository

The session moves into the role directory:

```bash
cd /etc/ansible/roles/apache
pwd
```

Expected directory:

```text
/etc/ansible/roles/apache
```

Initialize Git:

```bash
sudo git init
```

The session's output says the repository was initialized with the branch name `master`.

### Optional: use `main` as the branch name

Git may choose a different initial branch depending on its configuration. If you want to use `main`, rename the branch after initialization:

```bash
sudo git branch -M main
```

If you use `main`, use that same branch name in the GitHub push command. The session uses `master`, so its later commands use `master`.

### Inspect the directory

```bash
ls -lha
```

The session shows the role files and directories, including `README.md`, `apache.yml`, `defaults/`, `handlers/`, `meta/`, `tasks/`, `templates/`, `tests/`, and `vars/`, along with the `.git` directory.

---

## 13. Connect the Local Repository to GitHub

### 13.1 Create a GitHub repository

The session uses this repository URL:

```text
https://github.com/sumanpattanayak671-cmyk/apache.git
```

For a new role, create a repository on GitHub and copy its HTTPS or SSH URL. If you already have a repository, use its existing URL.

### 13.2 Add the remote

The session runs:

```bash
sudo git remote add origin https://github.com/sumanpattanayak671-cmyk/apache.git
```

`origin` is the conventional name for the remote repository.

Verify it:

```bash
git remote -v
```

If `origin` already exists, do not add it again. Inspect it and update it if necessary:

```bash
git remote set-url origin https://github.com/USERNAME/REPOSITORY.git
```

Replace the example URL with your own repository URL.

---

## 14. Stage and Commit the Role

### 14.1 Check the status

```bash
sudo git status
```

Before files are staged, Git lists them as untracked or modified.

### 14.2 Stage the role files

The session uses:

```bash
sudo git add .
```

This stages files in the current directory and its subdirectories, subject to ignore rules. Review the staged files before committing.

```bash
sudo git status
```

### 14.3 Configure the commit identity

The session uses:

```bash
sudo git config user.name "sumanpattanayak671-cmyk"
sudo git config user.email "sumanpattanayak671@gmail.com"
```

These commands configure the identity for the repository owned or accessed by the user running them with `sudo`. Git configuration under `sudo` can differ from the normal user's configuration.

If you work as your regular account, you can configure the repository identity without `sudo`:

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

Use an identity you intend to associate with your commits.

### 14.4 Create a commit

The session's successful commit command is:

```bash
sudo git commit -m "apache"
```

The `-m` option requires a commit message. A more descriptive message could be:

```bash
sudo git commit -m "Add reusable Apache HTTPD role"
```

A commit records the staged snapshot in local Git history. It does not upload files to GitHub yet.

---

## 15. Push the Role to GitHub

The session pushes the `master` branch:

```bash
sudo git push origin master
```

If your branch is named `main`, use:

```bash
git push -u origin main
```

The session's output confirms that Git uploaded the objects and created the remote branch `master`.

### Authentication and personal access tokens

The session describes using a GitHub personal access token (PAT) when Git prompts for a password over HTTPS.

GitHub generally does not accept an account password for Git-over-HTTPS authentication. If prompted for a password, use an appropriately scoped token, or configure SSH authentication.

**Token safety:**

- Treat a PAT like a password.
- Never put it in your playbook, repository, README, shell script, or a public screenshot.
- Do not paste it into a chat.
- Give it only the permissions needed for the repository.
- Store it securely and revoke it if exposed.

For an organization or repository with stricter policies, use the authentication method permitted by that account.

### Verify the push

Open your GitHub repository in a browser and confirm that the role files and commit appear there. Do not assume the push worked until you see the repository updated or Git reports success.

---

## 16. Import the Role into Ansible Galaxy

The session's workflow is:

1. Open Ansible Galaxy.
2. Navigate to the roles import workflow.
3. Select the option to import a role from GitHub.
4. Choose the `apache` repository.
5. Start the import.

After import, the role can be associated with a Galaxy namespace and role name. Exact interface labels and requirements may change, so follow the current Galaxy interface and its validation messages.

The session then uses this role identifier:

```text
sumanpattanayak671-cmyk.apache
```

A Galaxy import makes the role discoverable through Galaxy. It does not automatically guarantee that the role passes every test or works on every operating system. Keep the GitHub repository maintained and document supported platforms and requirements.

---

## 17. Install the Published Role

The session shows:

```bash
ansible-galaxy role install sumanpattanayak671-cmyk.apache
```

This asks Ansible Galaxy to install the role under the name `sumanpattanayak671-cmyk.apache`.

### Check installed roles

```bash
ansible-galaxy role list
```

### Use an installed role in a playbook

Once installed in a location Ansible searches, a playbook can refer to the role by its installed role name or directory name, depending on the installed path and role layout.

Example:

```yaml
---
- name: Configure Apache using an installed role
  hosts: webservers
  become: true

  roles:
    - sumanpattanayak671-cmyk.apache
```

Ensure the inventory contains the `webservers` group, and verify the exact installed role directory/name before using it. You can also install to a specific directory with the appropriate `ansible-galaxy role install` options and configure `roles_path` accordingly.

---

## 18. Important Corrections and Best Practices

These notes preserve the session's approach while highlighting points to check before using the role in a real environment.

### 18.1 Keep the role directory clean

The session copies the original playbook into the role directory:

```bash
sudo cp -r /home/ansible/apache.yml /etc/ansible/roles/apache/
```

This is useful as a reference, but a role repository normally needs its task files, templates, metadata, tests, and README more than a separate standalone playbook. Keep `apache.yml` only if you want it as a documented example or test playbook.

### 18.2 Prefer a handler over an unconditional restart

Instead of restarting Apache on every run, notify a handler from a task that changes a relevant configuration file. This helps keep the playbook idempotent and avoids unnecessary service interruptions.

### 18.3 Prefer predictable package state when appropriate

The session uses:

```yaml
state: latest
```

This can upgrade Apache whenever a newer package is available. For repeatable configuration management, consider:

```yaml
state: present
```

Choose based on whether the goal is simply to ensure installation or to keep the package updated.

### 18.4 Check role dependencies

The role uses `ansible.posix.firewalld`. Install the collection on the controller if it is not present:

```bash
ansible-galaxy collection install ansible.posix
```

The `firewalld` service and required Python bindings must also be available as required on the managed hosts.

### 18.5 Improve the template

The original template uses the HTML `<center>` element. For a modern HTML page, you could use CSS instead. The original template is retained here because it is the content used in the session.

### 18.6 Avoid unnecessary `sudo` and ownership problems

A role under `/etc/ansible/roles` may be owned by root, which can lead to needing `sudo` for Git operations. Be consistent about which account owns the repository. Avoid mixing root-owned Git configuration and normal-user Git configuration unintentionally.

### 18.7 Write a useful README

A public role should explain:

- What the role configures.
- Supported operating systems and requirements.
- How to install it.
- How to use it in a playbook.
- Variables and their defaults.
- Ports and services it changes.
- How to test it.
- Known limitations.

### 18.8 Check secrets before pushing

Before running `git add .`, check that the repository does not contain passwords, private keys, PATs, inventory files with sensitive data, or other secrets. A `.gitignore` file can help exclude local-only files, but it is not a substitute for reviewing staged content.

---

## 19. Troubleshooting

### Problem 1: `ansible-galaxy init` fails

Check that Ansible is installed and the command is available:

```bash
ansible-galaxy --version
```

Check the target directory's permissions. Use `sudo` only when needed for the chosen role location.

### Problem 2: `import_task` gives an error

Incorrect:

```yaml
- import_task: install.yml
```

Correct:

```yaml
- ansible.builtin.import_tasks: install.yml
```

The module name is `import_tasks`, with an **s** at the end.

### Problem 3: Ansible cannot find the role

Check the role directory and configured role paths:

```bash
ansible-config dump --only-changed | grep -i roles_path
ls -la /etc/ansible/roles/apache
```

Ensure the playbook refers to the correct role name and that the role is in a searched directory.

### Problem 4: The template file cannot be found

Confirm the file exists:

```bash
ls -l /etc/ansible/roles/apache/templates/index1.j2
```

For a task inside the role, use the role template file name or its path relative to the role's `templates/` directory, for example:

```yaml
src: index1.j2
```

### Problem 5: The firewall task fails

Check:

```bash
sudo systemctl status firewalld
sudo firewall-cmd --state
```

Also verify that the `ansible.posix` collection is installed and that the managed host meets the module's requirements.

### Problem 6: Git says `remote origin already exists`

Inspect the remote:

```bash
git remote -v
```

Update it if needed:

```bash
git remote set-url origin https://github.com/USERNAME/REPOSITORY.git
```

### Problem 7: Git push is rejected

Possible causes include authentication failure, an incorrect branch name, or existing commits on the remote repository. Check:

```bash
git branch --show-current
git remote -v
git status
```

Resolve the actual cause before retrying. If the remote contains a README or initial commit not present locally, fetch and reconcile histories rather than force-pushing blindly.

### Problem 8: Ansible Galaxy cannot import the repository

Check that:

- The GitHub repository is accessible to Galaxy.
- The role layout is valid.
- The repository contains the required role metadata.
- The import log reports no validation errors.
- The role's GitHub branch contains the latest commit.

Follow the current Galaxy import instructions if the interface or requirements have changed.

### Problem 9: The web page does not open

Check the service, page, and firewall:

```bash
sudo systemctl status httpd
sudo cat /var/www/html/index.html
sudo firewall-cmd --list-services
```

Then test connectivity using the correct server IP. A local firewall rule alone cannot fix missing routes, a blocked upstream firewall, or an incorrect IP address.

---

## 20. Quick Revision

### Complete workflow

```text
Create a standalone Apache playbook and Jinja2 template
                    |
                    v
Initialize the apache role with ansible-galaxy
                    |
                    v
Split tasks into install.yml, service.yml, config.yml
                    |
                    v
Add the template, handler, metadata, and main.yml imports
                    |
                    v
Create setup.yml and run ansible-playbook
                    |
                    v
Install Git and initialize the role repository
                    |
                    v
Stage files and create a commit
                    |
                    v
Add the GitHub remote and push the branch
                    |
                    v
Import the GitHub repository into Ansible Galaxy
                    |
                    v
Install the published role with ansible-galaxy
```

### Commands to remember

```bash
# Initialize a role
ansible-galaxy init apache --offline

# Check playbook syntax
ansible-playbook setup.yml --syntax-check

# Run the playbook
ansible-playbook setup.yml

# Install Git on RHEL
sudo dnf install -y git

# Initialize Git in the role directory
git init

# Inspect files and changes
git status

# Stage and commit
git add .
git commit -m "Add reusable Apache HTTPD role"

# Connect to GitHub
git remote add origin https://github.com/USERNAME/apache.git

# Push (use the branch name that actually exists)
git push -u origin main

# Import/install roles
ansible-galaxy role install NAMESPACE.ROLE_NAME

# List installed roles
ansible-galaxy role list

# Install the required collection
ansible-galaxy collection install ansible.posix
```

### Important distinctions

- **Git commit** saves a snapshot locally.
- **Git push** sends local commits to a remote repository.
- **GitHub** hosts the repository.
- **Ansible Galaxy import** makes a role available through Galaxy's role-discovery workflow.
- **`ansible-galaxy role install`** downloads/installs a role onto an Ansible controller.
- **`tasks/main.yml`** is the role's main task entry point.
- **`handlers/main.yml`** contains handlers; a task must notify a handler for it to run.
- **`templates/`** is for Jinja2 templates; **`files/`** is for static files.

---

## 21. Practice Questions

### Short-answer questions

1. What is an Ansible role?
2. Why is it useful to split a role into multiple task files?
3. What is the purpose of `ansible-galaxy init`?
4. What is the purpose of `tasks/main.yml`?
5. What is the difference between `templates/` and `files/`?
6. What does `become: true` do?
7. What is the purpose of `ansible_hostname` in the template?
8. What is a handler, and how is it triggered?
9. What does `git init` do?
10. What is the difference between `git commit` and `git push`?
11. What is the purpose of a Git remote named `origin`?
12. Why should a personal access token be kept secret?
13. What does `ansible-galaxy role install` do?
14. Why should a public role contain a README?
15. What does `state: started` mean for a service?

### Practical exercises

1. Create a role named `apache` using `ansible-galaxy init`.
2. Create `install.yml`, `service.yml`, and `config.yml`.
3. Import these task files from `tasks/main.yml` using `import_tasks`.
4. Create a Jinja2 template that prints each managed host's hostname.
5. Add a handler and notify it from a template task.
6. Run `ansible-playbook setup.yml --syntax-check`.
7. Run the playbook and verify HTTPD and the generated HTML file on each host.
8. Initialize a Git repository, stage the role, and create a commit.
9. Publish the repository to GitHub using secure authentication.
10. Import the repository into Ansible Galaxy and install the role on a test controller.

---

**End of Session 23 — Publishing Ansible Roles on GitHub and Ansible Galaxy**
