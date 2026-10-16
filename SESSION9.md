# SESSION 9 — Ansible Variables

## 1. Session overview

This session covers Ansible variables and their use in playbooks:
- Built-in variables: `inventory_hostname` and `hostvars`
- User-defined variables with `vars`
- String variables
- Importing variables using `vars_files`
- List/array variables and indexing
- Mapping/dictionary data with `loop`
- Interactive input using `vars_prompt`
- Hiding prompted input with `private: true`

The original examples use inventory groups `liu` and `suman`, with managed nodes such as `node1` and `node2`.

## 2. Create a playbook directory

```bash
mkdir playbook
cd playbook
```

These commands create a dedicated directory and move into it so the session's playbooks can be organized together.

## 3. Built-in variables

### `inventory_hostname`

```yaml
---
- hosts: liu
  tasks:
    - name: ansible built-in variables use
      debug:
        msg: "{{ inventory_hostname }}"
```

`inventory_hostname` returns the inventory name of the host on which the task is running. The `debug` module displays the value. `{{ ... }}` is Jinja2 variable-expression syntax.

### `hostvars`

```yaml
---
- hosts: liu
  tasks:
    - name: ansible built-in variables use
      debug:
        msg: "{{ hostvars }}"
```

`hostvars` is a dictionary of variables associated with inventory hosts. Printing the entire structure may produce extensive output; for troubleshooting, it is often more useful to inspect a particular host or variable.

## 4. User-defined variables with `vars`

```yaml
---
- hosts: liu
  vars:
    salutations: "hello everyone"
  tasks:
    - name: example of user define variable.
      debug:
        msg: "{{ salutations }}"
```

- `vars` defines variables for the play.
- `salutations` is the variable name.
- `hello everyone` is its value.
- The debug task prints the variable's value.

## 5. Three variable data types in the notes

1. **String** — a text value, for example `httpd` or `/var/www/html`.
2. **List/array** — an ordered sequence of values, such as package names.
3. **Mapping/dictionary** — key-value pairs, such as a user's `name` and `uid`.

## 6. String variables

Example based on `string_ex.yml`:

```yaml
---
- name: example of string type variable
  hosts: suman
  become: true
  vars:
    apache: httpd
    apache_root: /var/www/html
  tasks:
    - name: installing httpd server
      ansible.builtin.yum:
        name: "{{ apache }}"
        state: present

    - name: starting and enabling the httpd service
      ansible.builtin.service:
        name: "{{ apache }}"
        state: started
        enabled: true

    - name: creating the web-root directory
      ansible.builtin.file:
        path: "{{ apache_root }}"
        state: directory

    - name: creating index.html inside the web root
      ansible.builtin.copy:
        content: "hello everyone"
        dest: "{{ apache_root }}/index.html"
```

Explanation:
- `apache: httpd` stores the package/service name in a variable.
- `apache_root: /var/www/html` stores the web-root path.
- `name: "{{ apache }}"` reuses the variable instead of repeating `httpd`.
- `dest: "{{ apache_root }}/index.html"` builds the destination path using the variable.
- `state: present` ensures the package is installed.
- `state: started` ensures the service is running; `enabled: true` enables it at boot.

Package installation, service management, and writing under `/var/www/html` generally require administrative privileges, so this explanatory version includes `become: true`.

## 7. Import variables with `vars_files`

The notes define `/home/suman1/playbook/myvars.yml` like this:

```yaml
apache: httpd
apache_root: /var/www/html
```

Create/edit it with:

```bash
vim /home/suman1/playbook/myvars.yml
```

Then load it into a playbook:

```yaml
---
- name: test file for importing
  hosts: suman
  become: true
  vars_files:
    - /home/suman1/playbook/myvars.yml
  tasks:
    - name: installing httpd server
      ansible.builtin.yum:
        name: "{{ apache }}"
        state: present

    - name: starting and enabling the httpd service
      ansible.builtin.service:
        name: "{{ apache }}"
        state: started
        enabled: true

    - name: creating the web-root directory
      ansible.builtin.file:
        path: "{{ apache_root }}"
        state: directory

    - name: creating index.html inside the web root
      ansible.builtin.copy:
        content: "hello everyone"
        dest: "{{ apache_root }}/index.html"
```

`vars_files` loads variables from the listed YAML file. This avoids repeating common settings in multiple playbooks.

## 8. List variables for package installation

```yaml
---
- name: example of list variable
  hosts: suman
  become: true
  vars:
    mypakgs:
      - ftp
      - vsftpd
      - zsh
  tasks:
    - name: installing packages
      ansible.builtin.yum:
        name: "{{ mypakgs }}"
        state: present
```

A list uses one dash per item. The package module can receive the list and ensure each package is installed. The variable spelling `mypakgs` follows the original notes; for new work, a name like `my_packages` is clearer. Package names must exist in the configured repositories.

## 9. List/array indexing

```yaml
---
- name: use of array
  hosts: suman
  become: true
  vars:
    mypackg:
      - httpd
      - ftp
      - vsftpd
      - zsh
      - telnet
  tasks:
    - name: installing telnet by using array
      ansible.builtin.yum:
        name: "{{ mypackg[4] }}"
        state: present
```

Indexes start at zero:

| Index | Value |
|---:|---|
| `mypackg[0]` | `httpd` |
| `mypackg[1]` | `ftp` |
| `mypackg[2]` | `vsftpd` |
| `mypackg[3]` | `zsh` |
| `mypackg[4]` | `telnet` |

Therefore, `[4]` selects the fifth item. The index must exist in the list.

## 10. Mapping/dictionary variables and `loop`

```yaml
---
- name: mapping examples
  hosts: suman
  become: true
  tasks:
    - name: create new users
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        state: present
      loop:
        - name: su
          uid: 2022
        - name: sui
          uid: 2023
        - name: hello
          uid: 2024
```

Each loop item is a mapping containing `name` and `uid`. `item.name` and `item.uid` access the corresponding fields. Ansible runs the user task once for each entry, ensuring the users exist with the specified UIDs.

Check that the UIDs are not already in use before running this on real systems. User management requires suitable privileges.

## 11. Prompt for input with `vars_prompt`

The original notes demonstrate both visible and hidden input:

```yaml
---
- name: user input and hiding information
  hosts: suman
  vars_prompt:
    - name: user_name
      prompt: please enter your username.
      private: false
    - name: password
      prompt: enter your pass
      private: true
  tasks:
    - debug:
        msg: "the username is {{ user_name }} and password is {{ password }}"
```

- `vars_prompt` asks for values when the playbook starts.
- `name` defines the variable that receives the input.
- `prompt` is the text displayed to the operator.
- `private: false` shows typed input.
- `private: true` hides typed input, which is appropriate for a password.

**Security warning:** The original example prints the password using `debug`, so the secret may appear in terminal output or logs. Do not print passwords in real automation. A safer version displays only the username:

```yaml
---
- name: prompt for username and password
  hosts: suman
  vars_prompt:
    - name: user_name
      prompt: Please enter your username
      private: false
    - name: password
      prompt: Please enter your password
      private: true
  tasks:
    - name: Display only the username
      ansible.builtin.debug:
        msg: "The username is {{ user_name }}"
```

For secrets used repeatedly, consider Ansible Vault rather than storing secret values directly in a playbook.

## 12. Running the prompt playbook

```bash
ansible-playbook prompt_var.yml
```

Ansible asks for the username and password before executing the play. Because the password uses `private: true`, its characters are hidden while typing. The captured output in the original notes prints the password afterwards; that demonstrates variable interpolation but is not safe practice.

## 13. Useful commands

```bash
# Create and enter the playbook directory
mkdir playbook
cd playbook

# Edit a playbook
vim system_variable.yml

# Check playbook syntax
ansible-playbook --syntax-check system_variable.yml

# Run a playbook
ansible-playbook system_variable.yml

# Run the prompt example
ansible-playbook prompt_var.yml
```

## 14. Quick revision

| Term | Meaning |
|---|---|
| `inventory_hostname` | Inventory name of the current host |
| `hostvars` | Dictionary of variables associated with inventory hosts |
| `vars` | Defines variables in a play |
| `vars_files` | Loads variables from an external file |
| String | A text value |
| List/array | Ordered collection; indexes start at zero |
| Mapping/dictionary | Key-value data structure |
| `loop` | Repeats a task for each item |
| `item.name` | Reads the `name` field of the current loop item |
| `vars_prompt` | Requests values interactively at playbook start |
| `private: false` | Shows input while typing |
| `private: true` | Hides input while typing |
| `debug` | Prints messages/values; do not use it to expose secrets |

## 15. Original Session 9 notes — preserved verbatim

The complete source text is included below to retain all original commands, examples, output, spelling, and formatting.

```text
Session 9
variables
----------------------------------
we will write oure play books in a playbook dir

mkdir playbook
cd playbook


# built in variables
-----------
suman1@ansible-server:~/playbook$ cat system_variable.yml
---
- hosts: liu
  tasks: 
    - name: ansible built-in variables use
      debug:
        msg: "{{ inventory_hostname }}"
--------------

suman1@ansible-server:~/playbook$ cat system_variable.yml
---
- hosts: liu
  tasks: 
    - name: ansible built-in variables use
      debug:
        msg: "{{ hostvars }}"
--------------
suman1@ansible-server:~/playbook$ cat hello.yml 
---
- hosts: liu
  vars:
    salutations: hello everyone
  tasks:
    - name: example of user define variable.
      debug:
        msg: "{{ salutations }}"
...

suman1@ansible-server:~/playbook$ 
----------------
----------------
types of Ansible Playbook Variables: 
	A) String type variable
	B) Listing/Array type variable
	C) mapping type variable
----------------------------------
A) string type variable :-

suman1@ansible-server:~/playbook$ cat string_ex.yml 
---
- name: example of string type variable 
  hosts: suman
  vars:
    apache: httpd
    apache_root: /var/www/html
  tasks:
    - name: installing httpd server
      yum:
        name: "{{ apache }}"
        state: installed
    - name: starting & enabling the httpd service.
      service:
        name: "{{ apache }}"
        state: started
        enabled: true
    - name: creating "{{ apache_root }}" directory
      file:
        path: "{{ apache_root }}"
        state: directory
    - name: creatinf index.html page inside "{{ apache_root }}" 
      copy:
        content: "hello everyone"
        dest: "{{ apache_root }}/index.html"
suman1@ansible-server:~/playbook$ 

-----------------------------------------------------------
-----------------------------------------------------------
Importing Variables in ansible Playbook 

suman1@ansible-server:~/playbook$ cat /home/suman1/playbook/myvars.yml 
apache: httpd
apache_root: /var/www/html


suman1@ansible-server:~/playbook$ cat string_var2.yml 
---
- name: test file for importing 
  hosts: suman
  vars_files:
    - /home/suman1/playbook/myvars.yml
  tasks:
    - name: installing httpd server
      yum:
        name: "{{ apache }}"
        state: installed
    - name: starting & enabling the httpd service.
      service:
        name: "{{ apache }}"
        state: started
        enabled: true
    - name: creating "{{ apache_root }}" directory
      file:
        path: "{{ apache_root }}"
        state: directory
    - name: creatinf index.html page inside "{{ apache_root }}" 
      copy:
        content: "hello everyone"
        dest: "{{ apache_root }}/index.html"

suman1@ansible-server:~/playbook$ 

---------------------------------------------------------------------------
-------------------------------------------------------------------------
------------------------------------------------------------------------
B) list/Array type variables examples


list --->

suman1@ansible-server:~/playbook$ cat list_var.yml 
---
- name: example of listing variable or array variable
  hosts: suman
  vars:
    mypakgs:
      - ftp
      - vsftpd
      - zsh
  tasks:
    - name: installing packages
      yum:
        name: "{{ mypakgs }}"
        state: installed
...

Array ------>

suman1@ansible-server:~/playbook$ cat array_var.yml 
---
- name: use of array 
  hosts: suman
  vars:
    mypackg:
      - httpd
      - ftp
      - vsftpd
      - zsh
      - telnet
  tasks:
    - name: installing telnet by using array 
      yum:
        name: "{{ mypackg [4] }}"
        state: installed
...
suman1@ansible-server:~/playbook$ 

---------------------------------------
--------------------------------
-------------------------------
--------------------------
mapping or dictionary type variable examples

---------------->

suman1@ansible-server:~/playbook$ cat mapping_var.yml 
---
- name: mapping examples
  hosts: suman
  tasks:
    - name: create new users.
      user: 
        name: '{{ item.name }}'
        uid: '{{ item.uid }}'
        state: present
      loop:
        - name: su
          uid: 2022
        - name: sui
          uid: 2023
        - name: hello
          uid: 2024
suman1@ansible-server:~/playbook$ 
--------------------------------------------
user input and hiding message 

suman1@ansible-server:~/playbook$ cat prompt_var.yml 
---
- name: user input and hiding information
  hosts: suman
  vars_prompt:
    - name: user_name
      prompt: please enter your username.
      private: no
    - name: password
      prompt: enter your pass
      private: yes
  tasks:
    - debug: 
        msg: "the username is {{ user_name }} and password is {{ password }}"

        
suman1@ansible-server:~/playbook$ 

suman1@ansible-server:~/playbook$ ansible-playbook prompt_var.yml
please enter your username.: hello
enter your pass: 

PLAY [user input and hiding information] ****************************************************************************

TASK [Gathering Facts] **********************************************************************************************
ok: [node1]
ok: [node2]

TASK [debug] ********************************************************************************************************
ok: [node1] => {
    "msg": "the username is hello and password is 1234"
}
ok: [node2] => {
    "msg": "the username is hello and password is 1234"
}

PLAY RECAP **********************************************************************************************************
node1                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

suman1@ansible-server:~/playbook$ 

















```
