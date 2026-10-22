# SESSION22p1 — Ansible Roles: Detailed Notes

> **Topic:** Ansible roles  
> **Level:** Beginner to intermediate  
> **Lab environment:** Ansible control node with RHEL managed nodes

## Contents

1. What is an Ansible role?
2. Why use roles?
3. Playbook vs. role
4. Role directory structure
5. Purpose of each directory
6. Create and run an HTTPD role
7. Use variables
8. Use handlers
9. Find files and templates in a role
10. Ways to execute a role
11. Use multiple roles
12. Role dependencies
13. Useful commands
14. Best practices
15. Practice exercise
16. Quick revision
17. Original session note

---

---
what is roles explain in details

---

let's understand Roles in Ansible from the basics to practical examples, step by step, in an easy way for your RHEL and Ansible lab.

# 1. What is a Role in Ansible?

An Ansible Role is a structured way to organize automation tasks, variables, files, templates, handlers, and other components into separate directories so that you can reuse and manage them easily.

In simple words:

A role is a reusable collection of everything needed to perform a particular automation job.

For example, suppose you want to configure an HTTPD web server on two RHEL servers. You need to:

1. Install the HTTPD package.
2. Start the HTTPD service.
3. Enable HTTPD at boot.
4. Copy an HTML website.
5. Configure the firewall.
6. Restart HTTPD whenever its configuration changes.

You could write all these tasks in one large playbook. However, as your project grows, that playbook becomes difficult to manage.

Instead, you can create an `httpd` role that contains the tasks, files, handlers, and variables required to configure HTTPD.

Then you can reuse that role on `node1`, `node2`, or any other server.

## 2. Why do we use roles?

Organized code

Keep tasks, variables, templates, and files in separate directories.

Reusability

Use the same HTTPD role in multiple playbooks and environments.

Modularity

Separate jobs into roles such as `httpd`, `firewalld`, `users`, `ntp`, and `database`.

Easy maintenance

Update one role instead of editing the same tasks in several playbooks.

# 3. Playbook vs. Role

| Feature        | Playbook                            | Role                                          |
| -------------- | ----------------------------------- | --------------------------------------------- |
| Purpose        | Defines automation and target hosts | Organizes reusable automation components      |
| Main file      | `site.yml`, for example             | Files such as `tasks/main.yml`                |
| Host selection | Defined in plays                    | Usually selected by the playbook that uses it |
| Reuse          | Possible, but can become repetitive | Designed for reuse                            |
| Structure      | One or more plays                   | Standard directory structure                  |
| Example        | Configure a web server              | Reusable HTTPD configuration                  |

Important: A role does not replace a playbook. A playbook can call one or more roles.

Think of a playbook as the project manager and roles as specialized workers.

# 4. Ansible role directory structure

Let's create an HTTPD role.

First, understand the standard directory structure:

httpd_role/

tasks/

Contains the tasks to execute.

`main.yml`

handlers/

Contains actions triggered by notifications.

`main.yml`

templates/

Contains Jinja2 template files, usually `.j2`.

files/

Contains static files to copy to managed hosts.

vars/

Contains role variables, generally higher precedence.

`main.yml`

defaults/

Contains default variables, which are easy to override.

`main.yml`

meta/

Contains role metadata and dependencies.

`main.yml`

A role may also contain `include_tasks/`-related files within `tasks/`, tests, documentation, and other directories. You do not need to create every standard directory if your role does not need it.

## 5. What does each directory do?

| Directory    | Purpose                           | Example         |
| ------------ | --------------------------------- | --------------- |
| `tasks/`     | Tasks to execute                  | Install HTTPD   |
| `handlers/`  | Tasks triggered by `notify`       | Restart HTTPD   |
| `templates/` | Dynamic configuration files       | `index.html.j2` |
| `files/`     | Static files                      | `index.html`    |
| `vars/`      | Role-specific variables           | Package name    |
| `defaults/`  | Overridable default variables     | HTTPD port      |
| `meta/`      | Role information and dependencies | Required roles  |

The most important files to understand first are `tasks/main.yml`, `defaults/main.yml`, and `handlers/main.yml`.

Ansible automatically recognizes `main.yml` in these standard role directories.

# 6. Create your first role on the Ansible control node

Suppose your lab has this setup:

| Machine          | Purpose             |
| ---------------- | ------------------- |
| `ansible-server` | Control node        |
| `node1`          | Managed RHEL server |
| `node2`          | Managed RHEL server |

We will create a role that installs and starts HTTPD on both managed servers.

### Step 1: Create the role

On your Ansible control node, run:

```
mkdir -p ~/ansible-project
cd ~/ansible-project

ansible-galaxy role init httpd_role
```

This generates the standard role directory structure.

Check it:

```
tree httpd_role
```

If `tree` is not installed, use:

```
find httpd_role -maxdepth 2 -type f
```

You should see files and directories similar to these:

```
httpd_role/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
├── vars/
│   └── main.yml
└── README.md
```

The exact generated structure can vary with your Ansible version.

### Step 2: Write the tasks

Open the tasks file:

```
vim httpd_role/tasks/main.yml
```

Add:

```
---
- name: Install HTTPD package
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: Start and enable HTTPD
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

Save the file.

Explanation:

- `name`: Describes the task.
- `ansible.builtin.dnf`: Manages packages using DNF.
- `name: httpd`: Specifies the package to install.
- `state: present`: Ensures HTTPD is installed.
- `state: started`: Ensures the service is running.
- `enabled: true`: Enables the service at boot.

Using `ansible.builtin.dnf` is suitable for modern RHEL systems.

### Step 3: Create a playbook that calls the role

Go back to the project directory:

```
cd ~/ansible-project
vim webserver.yml
```

Add:

```
---
- name: Configure web servers using a role
  hosts: webservers
  become: true

  roles:
    - httpd_role
```

Here, `roles:` tells Ansible to execute the role named `httpd_role`.

The role directory must be in a location where Ansible can find it. Since `httpd_role` is alongside `webserver.yml`, Ansible can normally resolve it through the playbook's role search path.

### Step 4: Configure the inventory

Create an inventory file:

```
vim inventory
```

Add:

```
[webservers]
node1
node2
```

Ensure `node1` and `node2` are defined in your SSH configuration or use their IP addresses directly. Your Ansible user must also have working SSH access and privilege escalation permissions.

### Step 5: Run the playbook

```
ansible-playbook -i inventory webserver.yml
```

Ansible will:

1. Read the playbook.
2. Find the `httpd_role` role.
3. Load its `tasks/main.yml`.
4. Execute the tasks against `node1` and `node2`.
5. Install HTTPD and ensure that the service is running and enabled.

Check the service:

```
ansible -i inventory webservers -b \
  -m ansible.builtin.command \
  -a "systemctl is-active httpd"
```

Expected output for each host:

```
node1 | CHANGED | rc=0 >>
active

node2 | CHANGED | rc=0 >>
active
```

The status may show `SUCCESS` or `CHANGED` differently depending on the command and Ansible version. The important result is `active`.

# 7. Make the role more powerful using variables

Suppose you want to change the HTTPD package or website directory without editing the tasks every time.

You can use role variables.

Open:

```
vim httpd_role/defaults/main.yml
```

Add:

```
---
httpd_package: httpd
httpd_service: httpd
web_root: /var/www/html
web_message: "Welcome to my Ansible role"
```

Now update `httpd_role/tasks/main.yml`:

```
---
- name: Install web server package
  ansible.builtin.dnf:
    name: "{{ httpd_package }}"
    state: present

- name: Start and enable web server
  ansible.builtin.service:
    name: "{{ httpd_service }}"
    state: started
    enabled: true

- name: Create website page
  ansible.builtin.copy:
    content: |
      <html>
      <body>
        <h1>{{ web_message }}</h1>
      </body>
      </html>
    dest: "{{ web_root }}/index.html"
    owner: root
    group: root
    mode: "0644"
```

Run the playbook again:

```
ansible-playbook -i inventory webserver.yml
```

Test the website from a managed host:

```
curl http://localhost
```

You should see the HTML containing the welcome message.

## How variables work

For example:

```
web_message: "Welcome to my Ansible role"
```

The task uses:

```
content: "{{ web_message }}"
```

Ansible replaces the variable with its value when the task runs.

You can override the default value from the playbook:

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  vars:
    web_message: "Welcome to RHEL Web Server"

  roles:
    - httpd_role
```

This allows you to reuse the same role with different website messages.

Remember: `defaults/main.yml` contains default values. Playbook variables, inventory variables, and command-line extra variables may override them according to Ansible's variable precedence rules. Variables in `vars/main.yml` have higher precedence than role defaults, so use defaults for values that users should be able to customize easily.

# 8. Using handlers inside a role

A handler is a special task that runs when another task notifies it. Handlers are useful for restarting a service only when a configuration change occurs.

Let's configure HTTPD with a custom configuration file.

### Step 1: Create a template

```
vim httpd_role/templates/welcome.conf.j2
```

Add:

```
<VirtualHost *:80>
    ServerName {{ server_name }}
    DocumentRoot {{ web_root }}

    <Directory "{{ web_root }}">
        Require all granted
    </Directory>
</VirtualHost>
```

### Step 2: Add default variables

In `httpd_role/defaults/main.yml`, include:

```
server_name: localhost
web_root: /var/www/html
```

Keep the earlier variables if you are using them.

### Step 3: Create the handler

Open:

```
vim httpd_role/handlers/main.yml
```

Add:

```
---
- name: Restart HTTPD
  ansible.builtin.service:
    name: httpd
    state: restarted
```

### Step 4: Add the configuration task

Add this task to `httpd_role/tasks/main.yml`, after the package installation and service setup:

```
- name: Deploy HTTPD virtual host configuration
  ansible.builtin.template:
    src: welcome.conf.j2
    dest: /etc/httpd/conf.d/welcome.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart HTTPD
```

Run:

```
ansible-playbook -i inventory webserver.yml
```

When the template task changes the configuration file, it notifies the `Restart HTTPD` handler. The handler runs at the end of the play unless you explicitly flush handlers.

If the file is already correct and unchanged, the handler normally does not run.

Important: If you use this virtual host example, verify the HTTPD configuration before restarting it:

```
httpd -t
```

This example uses the standard RHEL HTTPD configuration path. For a production environment, also consider SELinux, firewall rules, hostname resolution, and the desired virtual-host configuration.

# 9. How does Ansible find files inside a role?

One useful feature of roles is that you do not need to specify every full path.

For example, if your role contains:

```
httpd_role/
├── files/
│   └── index.html
├── templates/
│   └── welcome.conf.j2
└── tasks/
    └── main.yml
```

To copy a static file:

```
- name: Copy website page
  ansible.builtin.copy:
    src: index.html
    dest: /var/www/html/index.html
```

Ansible can resolve `index.html` from the role's `files/` directory.

To deploy a template:

```
- name: Configure HTTPD
  ansible.builtin.template:
    src: welcome.conf.j2
    dest: /etc/httpd/conf.d/welcome.conf
```

Ansible can resolve the template from the role's `templates/` directory.

This makes roles portable: you can move the entire role to another project without rewriting all its internal file paths.

# 10. Different ways to execute a role

There are several ways to use roles in Ansible.

## Method 1: Using `roles:`

This is the method we used earlier.

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    - httpd_role
    - firewalld_role
```

Ansible executes the roles in the listed order, subject to dependencies and play execution rules.

Use this approach when you know which roles should be applied to a play.

## Method 2: Using `import_role`

`import_role` statically imports a role into the play.

```
---
- name: Configure web server
  hosts: webservers
  become: true

  tasks:
    - name: Install HTTPD using a role
      ansible.builtin.import_role:
        name: httpd_role
```

The role is processed during playbook parsing. This is useful when you want to organize roles within the `tasks:` section or apply task-level keywords to the imported role.

## Method 3: Using `include_role`

`include_role` dynamically includes a role while the playbook executes.

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Include HTTPD role
      ansible.builtin.include_role:
        name: httpd_role
      when: install_webserver | default(true)
```

Here, the role is included only when the condition is true.

### Difference between the three methods

| Feature               | `roles:`                  | `import_role`              | `include_role`                        |
| --------------------- | ------------------------- | -------------------------- | ------------------------------------- |
| Execution style       | Role-based play execution | Static                     | Dynamic                               |
| Location              | Play level                | Under `tasks:`             | Under `tasks:`                        |
| Conditional execution | Possible                  | Possible                   | Possible                              |
| Useful for            | Standard role execution   | Static role reuse in tasks | Conditional or dynamic role execution |

# 11. Example: Create two roles in one project

Let's imagine you want to configure both HTTPD and firewalld.

Your project could look like this:

```
ansible-project/
├── inventory
├── site.yml
├── httpd_role/
│   ├── tasks/
│   │   └── main.yml
│   └── defaults/
│       └── main.yml
└── firewalld_role/
    └── tasks/
        └── main.yml
```

The HTTPD role installs the web server. The firewalld role opens HTTP traffic.

### `firewalld_role/tasks/main.yml`

```
---
- name: Install firewalld
  ansible.builtin.dnf:
    name: firewalld
    state: present

- name: Start and enable firewalld
  ansible.builtin.service:
    name: firewalld
    state: started
    enabled: true

- name: Allow HTTP traffic
  ansible.posix.firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true
```

The `ansible.posix.firewalld` module belongs to the `ansible.posix` collection. Install it if it is not already available:

```
ansible-galaxy collection install ansible.posix
```

### `site.yml`

```
---
- name: Configure complete web infrastructure
  hosts: webservers
  become: true

  roles:
    - httpd_role
    - firewalld_role
```

Run:

```
ansible-playbook -i inventory site.yml
```

Now one playbook coordinates two independent roles. Each role handles its own responsibility.

# 12. Role dependencies

A role can declare dependencies on other roles using `meta/main.yml`.

For example, suppose the application role requires HTTPD to be installed first.

In `application_role/meta/main.yml`:

```
---
dependencies:
  - role: httpd_role
```

When Ansible processes `application_role`, it also processes its declared dependency.

Dependencies are useful when one role requires another role's setup. Be careful to avoid circular dependencies, such as role A depending on role B while role B depends on role A.

# 13. Useful commands for role management

| Command                                                 | Purpose                                                        |
| ------------------------------------------------------- | -------------------------------------------------------------- |
| `ansible-galaxy role init httpd_role`                   | Generate a role skeleton                                       |
| `ansible-galaxy role list`                              | List roles installed in configured role paths                  |
| `ansible-playbook -i inventory site.yml`                | Execute a playbook that uses roles                             |
| `ansible-playbook -i inventory site.yml --syntax-check` | Check playbook syntax                                          |
| `ansible-playbook -i inventory site.yml --list-tasks`   | Show the tasks Ansible plans to execute                        |
| `ansible-playbook -i inventory site.yml --check`        | Attempt a check-mode run without applying normal changes       |
| `ansible-galaxy install namespace.role_name`            | Install a published role, when the name is valid and available |

Check mode is a simulation, not a guarantee that every module or task can predict all changes.

# 14. Best practices for roles

- Give each role one clear responsibility.
- Use `defaults/main.yml` for configurable default values.
- Use handlers to restart services when configuration changes.
- Use templates for configuration files containing variables.
- Keep secrets out of plain-text role files; use Ansible Vault or another approved secrets-management solution.
- Use descriptive task names.
- Test roles against a lab server before using them in production.
- Keep role dependencies explicit and avoid circular dependencies.
- Use version control, such as Git, to track role changes.
- Use fully qualified module names such as `ansible.builtin.copy`.

# 15. Practice exercise for your RHEL lab

Try building this project yourself.

## HTTPD Role Lab

Beginner to intermediate practice

Your progress

0 of 8

Create an httpd_role using ansible-galaxy.

Install HTTPD using tasks/main.yml.

Start and enable the HTTPD service.

Create a default variable for the website message.

Copy or generate an index.html page.

Create a handler that restarts HTTPD.

Use the role on node1 and node2.

Verify the website with curl and check the service status.

## Final revision

Remember these five points:

1. Role: A reusable collection of Ansible automation components.
2. `tasks/main.yml`: Defines the tasks executed by the role.
3. `defaults/main.yml`: Defines overridable default variables.
4. `handlers/main.yml`: Defines tasks triggered by notifications.
5. `roles:` in a playbook: Calls roles to perform automation on selected hosts.

Once you understand roles, you can organize your RHEL automation into reusable components for HTTPD, firewalld, users, storage, networking, and other system administration tasks.

---


---

# Original Session Note (as supplied)

session22 p1 
--------------------------------ansible role
what is roles explain in details

-------------------


let's understand Roles in Ansible from the basics to practical examples, step by step, in an easy way for your RHEL and Ansible lab.

# 1. What is a Role in Ansible?

An Ansible Role is a structured way to organize automation tasks, variables, files, templates, handlers, and other components into separate directories so that you can reuse and manage them easily.

In simple words:

A role is a reusable collection of everything needed to perform a particular automation job.

For example, suppose you want to configure an HTTPD web server on two RHEL servers. You need to:

1. Install the HTTPD package.
2. Start the HTTPD service.
3. Enable HTTPD at boot.
4. Copy an HTML website.
5. Configure the firewall.
6. Restart HTTPD whenever its configuration changes.

You could write all these tasks in one large playbook. However, as your project grows, that playbook becomes difficult to manage.

Instead, you can create an `httpd` role that contains the tasks, files, handlers, and variables required to configure HTTPD.

Then you can reuse that role on `node1`, `node2`, or any other server.

## 2. Why do we use roles?

Organized code

Keep tasks, variables, templates, and files in separate directories.

Reusability

Use the same HTTPD role in multiple playbooks and environments.

Modularity

Separate jobs into roles such as `httpd`, `firewalld`, `users`, `ntp`, and `database`.

Easy maintenance

Update one role instead of editing the same tasks in several playbooks.

# 3. Playbook vs. Role

| Feature        | Playbook                            | Role                                          |
| -------------- | ----------------------------------- | --------------------------------------------- |
| Purpose        | Defines automation and target hosts | Organizes reusable automation components      |
| Main file      | `site.yml`, for example             | Files such as `tasks/main.yml`                |
| Host selection | Defined in plays                    | Usually selected by the playbook that uses it |
| Reuse          | Possible, but can become repetitive | Designed for reuse                            |
| Structure      | One or more plays                   | Standard directory structure                  |
| Example        | Configure a web server              | Reusable HTTPD configuration                  |

Important: A role does not replace a playbook. A playbook can call one or more roles.

Think of a playbook as the project manager and roles as specialized workers.

# 4. Ansible role directory structure

Let's create an HTTPD role.

First, understand the standard directory structure:

httpd_role/

tasks/

Contains the tasks to execute.

`main.yml`

handlers/

Contains actions triggered by notifications.

`main.yml`

templates/

Contains Jinja2 template files, usually `.j2`.

files/

Contains static files to copy to managed hosts.

vars/

Contains role variables, generally higher precedence.

`main.yml`

defaults/

Contains default variables, which are easy to override.

`main.yml`

meta/

Contains role metadata and dependencies.

`main.yml`

A role may also contain `include_tasks/`-related files within `tasks/`, tests, documentation, and other directories. You do not need to create every standard directory if your role does not need it.

## 5. What does each directory do?

| Directory    | Purpose                           | Example         |
| ------------ | --------------------------------- | --------------- |
| `tasks/`     | Tasks to execute                  | Install HTTPD   |
| `handlers/`  | Tasks triggered by `notify`       | Restart HTTPD   |
| `templates/` | Dynamic configuration files       | `index.html.j2` |
| `files/`     | Static files                      | `index.html`    |
| `vars/`      | Role-specific variables           | Package name    |
| `defaults/`  | Overridable default variables     | HTTPD port      |
| `meta/`      | Role information and dependencies | Required roles  |

The most important files to understand first are `tasks/main.yml`, `defaults/main.yml`, and `handlers/main.yml`.

Ansible automatically recognizes `main.yml` in these standard role directories.

# 6. Create your first role on the Ansible control node

Suppose your lab has this setup:

| Machine          | Purpose             |
| ---------------- | ------------------- |
| `ansible-server` | Control node        |
| `node1`          | Managed RHEL server |
| `node2`          | Managed RHEL server |

We will create a role that installs and starts HTTPD on both managed servers.

### Step 1: Create the role

On your Ansible control node, run:

```
mkdir -p ~/ansible-project
cd ~/ansible-project

ansible-galaxy role init httpd_role
```

This generates the standard role directory structure.

Check it:

```
tree httpd_role
```

If `tree` is not installed, use:

```
find httpd_role -maxdepth 2 -type f
```

You should see files and directories similar to these:

```
httpd_role/
├── defaults/
│   └── main.yml
├── files/
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
├── tests/
│   ├── inventory
│   └── test.yml
├── vars/
│   └── main.yml
└── README.md
```

The exact generated structure can vary with your Ansible version.

### Step 2: Write the tasks

Open the tasks file:

```
vim httpd_role/tasks/main.yml
```

Add:

```
---
- name: Install HTTPD package
  ansible.builtin.dnf:
    name: httpd
    state: present

- name: Start and enable HTTPD
  ansible.builtin.service:
    name: httpd
    state: started
    enabled: true
```

Save the file.

Explanation:

- `name`: Describes the task.
- `ansible.builtin.dnf`: Manages packages using DNF.
- `name: httpd`: Specifies the package to install.
- `state: present`: Ensures HTTPD is installed.
- `state: started`: Ensures the service is running.
- `enabled: true`: Enables the service at boot.

Using `ansible.builtin.dnf` is suitable for modern RHEL systems.

### Step 3: Create a playbook that calls the role

Go back to the project directory:

```
cd ~/ansible-project
vim webserver.yml
```

Add:

```
---
- name: Configure web servers using a role
  hosts: webservers
  become: true

  roles:
    - httpd_role
```

Here, `roles:` tells Ansible to execute the role named `httpd_role`.

The role directory must be in a location where Ansible can find it. Since `httpd_role` is alongside `webserver.yml`, Ansible can normally resolve it through the playbook's role search path.

### Step 4: Configure the inventory

Create an inventory file:

```
vim inventory
```

Add:

```
[webservers]
node1
node2
```

Ensure `node1` and `node2` are defined in your SSH configuration or use their IP addresses directly. Your Ansible user must also have working SSH access and privilege escalation permissions.

### Step 5: Run the playbook

```
ansible-playbook -i inventory webserver.yml
```

Ansible will:

1. Read the playbook.
2. Find the `httpd_role` role.
3. Load its `tasks/main.yml`.
4. Execute the tasks against `node1` and `node2`.
5. Install HTTPD and ensure that the service is running and enabled.

Check the service:

```
ansible -i inventory webservers -b \
  -m ansible.builtin.command \
  -a "systemctl is-active httpd"
```

Expected output for each host:

```
node1 | CHANGED | rc=0 >>
active

node2 | CHANGED | rc=0 >>
active
```

The status may show `SUCCESS` or `CHANGED` differently depending on the command and Ansible version. The important result is `active`.

# 7. Make the role more powerful using variables

Suppose you want to change the HTTPD package or website directory without editing the tasks every time.

You can use role variables.

Open:

```
vim httpd_role/defaults/main.yml
```

Add:

```
---
httpd_package: httpd
httpd_service: httpd
web_root: /var/www/html
web_message: "Welcome to my Ansible role"
```

Now update `httpd_role/tasks/main.yml`:

```
---
- name: Install web server package
  ansible.builtin.dnf:
    name: "{{ httpd_package }}"
    state: present

- name: Start and enable web server
  ansible.builtin.service:
    name: "{{ httpd_service }}"
    state: started
    enabled: true

- name: Create website page
  ansible.builtin.copy:
    content: |
      <html>
      <body>
        <h1>{{ web_message }}</h1>
      </body>
      </html>
    dest: "{{ web_root }}/index.html"
    owner: root
    group: root
    mode: "0644"
```

Run the playbook again:

```
ansible-playbook -i inventory webserver.yml
```

Test the website from a managed host:

```
curl http://localhost
```

You should see the HTML containing the welcome message.

## How variables work

For example:

```
web_message: "Welcome to my Ansible role"
```

The task uses:

```
content: "{{ web_message }}"
```

Ansible replaces the variable with its value when the task runs.

You can override the default value from the playbook:

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  vars:
    web_message: "Welcome to RHEL Web Server"

  roles:
    - httpd_role
```

This allows you to reuse the same role with different website messages.

Remember: `defaults/main.yml` contains default values. Playbook variables, inventory variables, and command-line extra variables may override them according to Ansible's variable precedence rules. Variables in `vars/main.yml` have higher precedence than role defaults, so use defaults for values that users should be able to customize easily.

# 8. Using handlers inside a role

A handler is a special task that runs when another task notifies it. Handlers are useful for restarting a service only when a configuration change occurs.

Let's configure HTTPD with a custom configuration file.

### Step 1: Create a template

```
vim httpd_role/templates/welcome.conf.j2
```

Add:

```
<VirtualHost *:80>
    ServerName {{ server_name }}
    DocumentRoot {{ web_root }}

    <Directory "{{ web_root }}">
        Require all granted
    </Directory>
</VirtualHost>
```

### Step 2: Add default variables

In `httpd_role/defaults/main.yml`, include:

```
server_name: localhost
web_root: /var/www/html
```

Keep the earlier variables if you are using them.

### Step 3: Create the handler

Open:

```
vim httpd_role/handlers/main.yml
```

Add:

```
---
- name: Restart HTTPD
  ansible.builtin.service:
    name: httpd
    state: restarted
```

### Step 4: Add the configuration task

Add this task to `httpd_role/tasks/main.yml`, after the package installation and service setup:

```
- name: Deploy HTTPD virtual host configuration
  ansible.builtin.template:
    src: welcome.conf.j2
    dest: /etc/httpd/conf.d/welcome.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart HTTPD
```

Run:

```
ansible-playbook -i inventory webserver.yml
```

When the template task changes the configuration file, it notifies the `Restart HTTPD` handler. The handler runs at the end of the play unless you explicitly flush handlers.

If the file is already correct and unchanged, the handler normally does not run.

Important: If you use this virtual host example, verify the HTTPD configuration before restarting it:

```
httpd -t
```

This example uses the standard RHEL HTTPD configuration path. For a production environment, also consider SELinux, firewall rules, hostname resolution, and the desired virtual-host configuration.

# 9. How does Ansible find files inside a role?

One useful feature of roles is that you do not need to specify every full path.

For example, if your role contains:

```
httpd_role/
├── files/
│   └── index.html
├── templates/
│   └── welcome.conf.j2
└── tasks/
    └── main.yml
```

To copy a static file:

```
- name: Copy website page
  ansible.builtin.copy:
    src: index.html
    dest: /var/www/html/index.html
```

Ansible can resolve `index.html` from the role's `files/` directory.

To deploy a template:

```
- name: Configure HTTPD
  ansible.builtin.template:
    src: welcome.conf.j2
    dest: /etc/httpd/conf.d/welcome.conf
```

Ansible can resolve the template from the role's `templates/` directory.

This makes roles portable: you can move the entire role to another project without rewriting all its internal file paths.

# 10. Different ways to execute a role

There are several ways to use roles in Ansible.

## Method 1: Using `roles:`

This is the method we used earlier.

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    - httpd_role
    - firewalld_role
```

Ansible executes the roles in the listed order, subject to dependencies and play execution rules.

Use this approach when you know which roles should be applied to a play.

## Method 2: Using `import_role`

`import_role` statically imports a role into the play.

```
---
- name: Configure web server
  hosts: webservers
  become: true

  tasks:
    - name: Install HTTPD using a role
      ansible.builtin.import_role:
        name: httpd_role
```

The role is processed during playbook parsing. This is useful when you want to organize roles within the `tasks:` section or apply task-level keywords to the imported role.

## Method 3: Using `include_role`

`include_role` dynamically includes a role while the playbook executes.

```
---
- name: Configure web servers
  hosts: webservers
  become: true

  tasks:
    - name: Include HTTPD role
      ansible.builtin.include_role:
        name: httpd_role
      when: install_webserver | default(true)
```

Here, the role is included only when the condition is true.

### Difference between the three methods

| Feature               | `roles:`                  | `import_role`              | `include_role`                        |
| --------------------- | ------------------------- | -------------------------- | ------------------------------------- |
| Execution style       | Role-based play execution | Static                     | Dynamic                               |
| Location              | Play level                | Under `tasks:`             | Under `tasks:`                        |
| Conditional execution | Possible                  | Possible                   | Possible                              |
| Useful for            | Standard role execution   | Static role reuse in tasks | Conditional or dynamic role execution |

# 11. Example: Create two roles in one project

Let's imagine you want to configure both HTTPD and firewalld.

Your project could look like this:

```
ansible-project/
├── inventory
├── site.yml
├── httpd_role/
│   ├── tasks/
│   │   └── main.yml
│   └── defaults/
│       └── main.yml
└── firewalld_role/
    └── tasks/
        └── main.yml
```

The HTTPD role installs the web server. The firewalld role opens HTTP traffic.

### `firewalld_role/tasks/main.yml`

```
---
- name: Install firewalld
  ansible.builtin.dnf:
    name: firewalld
    state: present

- name: Start and enable firewalld
  ansible.builtin.service:
    name: firewalld
    state: started
    enabled: true

- name: Allow HTTP traffic
  ansible.posix.firewalld:
    service: http
    permanent: true
    state: enabled
    immediate: true
```

The `ansible.posix.firewalld` module belongs to the `ansible.posix` collection. Install it if it is not already available:

```
ansible-galaxy collection install ansible.posix
```

### `site.yml`

```
---
- name: Configure complete web infrastructure
  hosts: webservers
  become: true

  roles:
    - httpd_role
    - firewalld_role
```

Run:

```
ansible-playbook -i inventory site.yml
```

Now one playbook coordinates two independent roles. Each role handles its own responsibility.

# 12. Role dependencies

A role can declare dependencies on other roles using `meta/main.yml`.

For example, suppose the application role requires HTTPD to be installed first.

In `application_role/meta/main.yml`:

```
---
dependencies:
  - role: httpd_role
```

When Ansible processes `application_role`, it also processes its declared dependency.

Dependencies are useful when one role requires another role's setup. Be careful to avoid circular dependencies, such as role A depending on role B while role B depends on role A.

# 13. Useful commands for role management

| Command                                                 | Purpose                                                        |
| ------------------------------------------------------- | -------------------------------------------------------------- |
| `ansible-galaxy role init httpd_role`                   | Generate a role skeleton                                       |
| `ansible-galaxy role list`                              | List roles installed in configured role paths                  |
| `ansible-playbook -i inventory site.yml`                | Execute a playbook that uses roles                             |
| `ansible-playbook -i inventory site.yml --syntax-check` | Check playbook syntax                                          |
| `ansible-playbook -i inventory site.yml --list-tasks`   | Show the tasks Ansible plans to execute                        |
| `ansible-playbook -i inventory site.yml --check`        | Attempt a check-mode run without applying normal changes       |
| `ansible-galaxy install namespace.role_name`            | Install a published role, when the name is valid and available |

Check mode is a simulation, not a guarantee that every module or task can predict all changes.

# 14. Best practices for roles

- Give each role one clear responsibility.
- Use `defaults/main.yml` for configurable default values.
- Use handlers to restart services when configuration changes.
- Use templates for configuration files containing variables.
- Keep secrets out of plain-text role files; use Ansible Vault or another approved secrets-management solution.
- Use descriptive task names.
- Test roles against a lab server before using them in production.
- Keep role dependencies explicit and avoid circular dependencies.
- Use version control, such as Git, to track role changes.
- Use fully qualified module names such as `ansible.builtin.copy`.

# 15. Practice exercise for your RHEL lab

Try building this project yourself.

## HTTPD Role Lab

Beginner to intermediate practice

Your progress

0 of 8

Create an httpd_role using ansible-galaxy.

Install HTTPD using tasks/main.yml.

Start and enable the HTTPD service.

Create a default variable for the website message.

Copy or generate an index.html page.

Create a handler that restarts HTTPD.

Use the role on node1 and node2.

Verify the website with curl and check the service status.

## Final revision

Remember these five points:

1. Role: A reusable collection of Ansible automation components.
2. `tasks/main.yml`: Defines the tasks executed by the role.
3. `defaults/main.yml`: Defines overridable default variables.
4. `handlers/main.yml`: Defines tasks triggered by notifications.
5. `roles:` in a playbook: Calls roles to perform automation on selected hosts.

Once you understand roles, you can organize your RHEL automation into reusable components for HTTPD, firewalld, users, storage, networking, and other system administration tasks.


------------------------------------------------------------------------------------------------------------------------------
