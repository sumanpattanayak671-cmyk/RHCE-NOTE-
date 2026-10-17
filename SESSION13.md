# SESSION 13 — Ansible Vault

## 1. What is Ansible Vault?

**Ansible Vault** is an Ansible feature used to encrypt sensitive information so that it is not readable in plain text. It is useful when automation files contain secrets such as:

- Passwords and secret variables
- API tokens and application credentials
- Private keys or other confidential configuration values
- Database credentials

Vault encrypts files or individual data files that Ansible can decrypt when authorized. The encrypted file can remain encrypted while Ansible runs a playbook, provided that Ansible receives the correct Vault password.

> **Important:** Vault protects data at rest. It does not automatically make every value safe at runtime, and it does not replace careful permissions, secret handling, or secure logging.

## 2. Encrypt an entire YAML playbook

Suppose `vault.yml` contains a playbook with a variable named `pw`:

```yaml
---
- name: Ansible vault playbook
  hosts: all
  vars:
    pw: redhat
  tasks:
    - name: creating user account and setting the password
      user:
        name: testuser
        password: "{{ pw }}"
```

The example above is formatted with valid YAML indentation. The original notes contain a few indentation/spacing issues; the original source is preserved at the end of this document.

### Encrypt the file

```bash
ansible-vault encrypt vault.yml
```

Ansible asks you to enter and confirm a Vault password. After successful encryption, the file contents are replaced by encrypted text.

To inspect the encrypted file:

```bash
cat vault.yml
```

An encrypted file begins with a header similar to:

```text
$ANSIBLE_VAULT;1.1;AES256
```

The following lines are encrypted data. They are not the original YAML playbook and should not be edited manually.

### What does encryption do?

- It encrypts the contents of `vault.yml`.
- The file can be stored or transferred without exposing its plaintext contents, assuming the Vault password is kept secure.
- You need the correct Vault password to view, edit, or run the encrypted file.

## 3. Run an encrypted playbook without decrypting it permanently

If you run an encrypted playbook without supplying the Vault password:

```bash
ansible-playbook vault.yml
```

you may receive an error such as:

```text
ERROR! Attempting to decrypt but no vault secrets found
```

This means Ansible needs the Vault secret but has not been given a way to obtain it.

Run the playbook and enter the Vault password when prompted:

```bash
ansible-playbook vault.yml --ask-vault-pass
```

Ansible decrypts the contents in memory as needed for execution; the file itself remains encrypted on disk. You do **not** need to run `ansible-vault decrypt` just to execute it.

### Understanding the run output

- `ok` means a task ran successfully and did not report a change.
- `changed` means Ansible made or reported a change on that managed node.
- `unreachable=0` means the nodes were reachable.
- `failed=0` means no task failed.

## 4. View an encrypted file

To display the decrypted contents without changing the encrypted file on disk:

```bash
ansible-vault view vault.yml
```

Enter the Vault password when prompted. The command prints the plaintext contents to the terminal, but it does not permanently decrypt the file.

Use care when viewing secrets in shared terminals, recorded sessions, or screen-sharing environments.

## 5. Edit an encrypted file

To edit an encrypted file safely:

```bash
ansible-vault edit vault.yml
```

Enter the Vault password. Ansible opens the file in the configured text editor and re-encrypts it when you save and exit.

This is preferable to manually trying to change the encrypted ciphertext.

## 6. Decrypt a file permanently

To convert an encrypted file back into plaintext:

```bash
ansible-vault decrypt vault.yml
```

Enter the Vault password. After successful decryption, the file is no longer protected by Vault encryption.

Verify its contents with:

```bash
cat vault.yml
```

**Security warning:** A decrypted playbook can expose secrets to anyone who can read the file. Decrypt only when necessary, restrict file permissions, and re-encrypt the file when appropriate.

## 7. Encrypt only the secret variables in a separate file

Instead of encrypting the entire playbook, a common approach is to keep ordinary automation code readable and store sensitive variables in a separate Vault-encrypted YAML file.

### Step 1: Create an encrypted variable file

```bash
ansible-vault create /home/suman1/crypt.yml
```

Enter and confirm a Vault password. An editor opens. Put the secret variable in valid YAML:

```yaml
pw: "1234"
```

Save and exit. The file is encrypted when created.

To view its plaintext contents when authorized:

```bash
ansible-vault view /home/suman1/crypt.yml
```

### Step 2: Reference the encrypted file from the playbook

Example `pass.yml`:

```yaml
---
- name: Ansible vault playbook
  hosts: all
  vars_files:
    - /home/suman1/crypt.yml
  tasks:
    - name: creating user account and setting the password
      user:
        name: sagarika
        password: "{{ pw }}"
```

`vars_files` tells Ansible to load variables from the specified file. Because `crypt.yml` is encrypted, Ansible also needs the Vault password.

### Step 3: Execute the playbook

Without a Vault password, Ansible may report that no Vault secrets were found:

```bash
ansible-playbook pass.yml
```

Provide the password interactively:

```bash
ansible-playbook pass.yml --ask-vault-pass
```

This approach separates secret data from the playbook logic, making it easier to review the playbook without exposing the secret variable.

## 8. Change the Vault password (rekey)

If you need to change the password used to encrypt a Vault file, use:

```bash
ansible-vault rekey /home/suman1/crypt.yml
```

Ansible asks for:
1. The current Vault password
2. The new Vault password
3. Confirmation of the new password

After success, use the new password for future Vault operations. Rekeying changes the password protecting the file; it does not mean the secret value inside the YAML variable has changed.

## 9. Supply the Vault password using a password file

The notes demonstrate using a file named `pass.txt` containing the Vault password, then passing that file to Ansible:

```bash
cat pass.txt
123
```

Run the playbook using:

```bash
ansible-playbook ping_vault.yml --vault-password-file pass.txt
```

Ansible reads the Vault password from the specified file rather than prompting interactively.

### Security considerations for password files

- Do not commit a Vault password file to a public or shared Git repository.
- Restrict access to the file, for example with suitable ownership and permissions.
- Store it separately from the encrypted secrets when possible.
- Remember that anyone who can read the password file may be able to decrypt the Vault files it protects.

For example, access permissions can be restricted to the file owner:

```bash
chmod 600 pass.txt
```

This is an additional security measure, not a substitute for securely managing the password file itself.

## 10. Important note about Linux user passwords

The source examples pass a plain-text variable directly to the `user` module's `password` parameter:

```yaml
password: "{{ pw }}"
```

Ansible warns that this value does not appear to be hashed. On Linux systems, the `user` module generally expects a **hashed password string**, not a plain-text password. Vault encrypts the variable file, but it does **not** hash the Linux account password for you.

A safer pattern is to supply a suitable password hash. Generate it using a trusted method on the control node, then store the resulting hash in the Vault-encrypted variable file. For example, a playbook might use:

```yaml
password: "{{ linux_password_hash }}"
```

Here, `linux_password_hash` must contain a valid hash in the format accepted by the target operating system. Do not put the original plain-text password in the `password` parameter and assume Vault encryption makes it a valid Linux password hash.

## 11. Command reference

| Command | Purpose |
|---|---|
| `ansible-vault encrypt vault.yml` | Encrypt an existing file |
| `ansible-vault create crypt.yml` | Create a new encrypted file |
| `ansible-vault view vault.yml` | View plaintext without permanently decrypting the file |
| `ansible-vault edit vault.yml` | Edit an encrypted file |
| `ansible-vault decrypt vault.yml` | Permanently decrypt a file |
| `ansible-vault rekey crypt.yml` | Change the Vault password protecting a file |
| `ansible-playbook vault.yml --ask-vault-pass` | Run a playbook and prompt for the Vault password |
| `ansible-playbook playbook.yml --vault-password-file pass.txt` | Run using a password file |

Use the actual file paths and names in your environment.

## 12. Common errors and fixes

### Error: `Attempting to decrypt but no vault secrets found`

**Cause:** An encrypted playbook or variable file is being loaded, but Ansible has not received a Vault password.

**Fix:** Use `--ask-vault-pass` or `--vault-password-file`:

```bash
ansible-playbook vault.yml --ask-vault-pass
```

### Error: `Invalid vars_file ... Attempting to decrypt but no vault secrets found`

**Cause:** A file referenced by `vars_files` is encrypted, and Ansible cannot decrypt it because no Vault secret was provided.

**Fix:**

```bash
ansible-playbook pass.yml --ask-vault-pass
```

### Warning: Password does not appear to have been hashed

**Cause:** A plain-text value was supplied to the Linux `user` module's `password` parameter.

**Fix:** Supply a valid password hash. Vault encryption protects the stored variable; it does not convert a plain-text password into a Linux password hash.

## 13. Quick revision

- **Ansible Vault** encrypts sensitive files and variables.
- `encrypt` encrypts an existing file.
- `create` creates a new encrypted file.
- `view` displays plaintext without permanently decrypting the file.
- `edit` modifies an encrypted file securely.
- `decrypt` removes Vault encryption from the file.
- `rekey` changes the Vault password.
- `--ask-vault-pass` prompts for the Vault password during playbook execution.
- `--vault-password-file` obtains the Vault password from a file.
- `vars_files` loads variables from an external YAML file.
- Vault encryption and Linux password hashing are different operations.

## 14. Practice questions

1. What is Ansible Vault?
2. Why is Ansible Vault used?
3. How do you encrypt an existing YAML file?
4. What happens if you execute an encrypted playbook without providing the Vault password?
5. What is the difference between `ansible-vault view` and `ansible-vault decrypt`?
6. How can you edit an encrypted file?
7. How do you change the Vault password?
8. How do you use an encrypted variable file with `vars_files`?
9. What is the purpose of `--ask-vault-pass`?
10. What security risks are associated with storing the Vault password in a file?
11. Why does encrypting a variable with Vault not automatically make it a valid Linux account password?
12. What does `changed=1` mean in a play recap?

---

# Original source text — preserved

The following section reproduces the source notes as provided, including the original command lines, output, spacing, and examples.

Session 13

Ansible Vault 

Ansible Vault is a feature of Ansible used to encrypt and protect sensitive data, such as passwords, SSH keys, and secret variables. It ensures that confidential information remains secure in playbooks and variable files.

----------------------------------------------------
suppose i have a yml file called vault.yml that contain

---
- name: Ansible vault playbook
  hosts: all
  vars:
    pw: redhat
  tasks:
    -name: creating useraccount and setting the password
     user:
       name: testuser
        password: "{{ pw }}"


to encrypt it 

suman1@ansible-server:~/playbook$ ansible-vault encrypt vault.yml
New Vault password:
Confirm New Vault password:
Encryption successful
suman1@ansible-server:~/playbook$ cat vault.yml
$ANSIBLE_VAULT;1.1;AES256
61343932353736363332346234353532613566363335613437653135326165613762366535373637
3632336539613035656537363635613162383765333865620a623061626436626364336436356261
36616465616131376665636464336139353337383531393136396466366161353536303563353361
3437383136343936660a393733373134336532393136396331666234663039646139633138343065
30613038343530373836626566663331633937623630616565356561613564663835313938376130
31386431323462333266396562366661643564336665376237336164363466323430333832663337
64303665626433306265353638323766366164346436326639383563313762396266366535356431
33363036313138363138636465353533653464343763356134316439626632343535666563396465
66616638393837643161646165386462386439653038643330313338356565623830306635646533
64316433623561643737336231626435376266303863306135633865306564396363333936613834
65356661313937326235366339636535333830346532646261653531636537653766616539653964
30343063363461613063623236356666303463653561316236393463623637613031663638363039
66336266313161646539393735356138666637643663396663336266353564613732633539313536
6532383263343463643765613236383730653534663035373934
suman1@ansible-server:~/playbook$

suman1@ansible-server:~/playbook$ ansible-playbook vault.yml
ERROR! Attempting to decrypt but no vault secrets found

---------------------------------------
to edit it --->
suman1@ansible-server:~/playbook$ ansible-vault edit vault.yml
Vault password:
-----------------------------------------
to see the content

suman1@ansible-server:~/playbook$ ansible-vault view vault.yml
Vault password:
---
- name: Ansible vault playbook
  hosts: all
  vars:
    pw: redhat
  tasks:
    -name: creating useraccount and setting the password
     user:
       name: testuser
        password: "{{ pw }}"

-----------------------------------------------------
how to execute it without decrypting--->

suman1@ansible-server:~/playbook$ ansible-playbook vault.yml --ask-vault-pass
Vault password:

PLAY [Ansible vault playbook] ******************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [node3]
ok: [node1]
ok: [node2]

TASK [creating useraccount and setting the password] *******************************************************************************************************
[WARNING]: The input password appears not to have been hashed. The 'password' argument must be encrypted for this module to work properly.
changed: [node3]
changed: [node2]
changed: [node1]

PLAY RECAP *************************************************************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node3                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

-------------------------------------------------------

to decrypt it 

suman1@ansible-server:~/playbook$ ansible-vault decrypt vault.yml
Vault password:
Decryption successful

suman1@ansible-server:~/playbook$ cat vault.yml
---
- name: Ansible vault playbook
  hosts: all
  vars:
    pw: redhat
  tasks:
    - name: creating useraccount and setting the password
      user:
        name: testuser
        password: "{{ pw }}"
suman1@ansible-server:~/playbook$
-------------------------------------------------------- 

=========================================================
======================================================
---------------------------------------------------------

if we want to decrypt only password by using vault we need to create a separate file who store the password

suman1@ansible-server:~/playbook$ ansible-vault create /home/suman1/crypt.yml

suman1@ansible-server:~/playbook$ ansible-vault view /home/suman1/crypt.yml
Vault password:
pw: 1234

suman1@ansible-server:~/playbook$ cat pass.yml
---
- name: Ansible vault playbook
  hosts: all
  vars_files:
    - /home/suman1/crypt.yml
  tasks:
    - name: creating user account and setting the password
      user:
        name: sagarika
        password: "{{ pw }}"
...

suman1@ansible-server:~/playbook$ ansible-playbook pass.yml
ERROR! Invalid vars_file '/home/suman1/crypt.yml'.. Attempting to decrypt but no vault secrets found
suman1@ansible-server:~/playbook$ ansible-playbook pass.yml --ask-vault-pass
Vault password:

PLAY [Ansible vault playbook] ******************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [node2]
ok: [node1]
ok: [node3]

TASK [creating user account and setting the password] ******************************************************************************************************
[WARNING]: The input password appears not to have been hashed. The 'password' argument must be encrypted for this module to work properly.
changed: [node3]
changed: [node2]
changed: [node1]

PLAY RECAP *************************************************************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node3                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
---------------------------------------------------------------------------------------------------------------------------------------------------------------



for changing vault password---------------->

suman1@ansible-server:~/playbook$ ansible-vault rekey /home/suman1/crypt.yml
Vault password:
New Vault password:
Confirm New Vault password:
Rekey successful

----------------------------------------------------------------------------------------------------------------------------------------------------------

suman1@ansible-server:~/playbook$ cat pass.txt
123
suman1@ansible-server:~/playbook$ ansible-playbook ping_vault.yml --vault-password-file pass.txt

PLAY [ping test playbook] **********************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************
ok: [node3]
ok: [node1]
ok: [node2]

TASK [ping] ************************************************************************************************************************************************
ok: [node3]
ok: [node1]
ok: [node2]

PLAY RECAP *************************************************************************************************************************************************
node1                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node2                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
node3                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

suman1@ansible-server:~/playbook$
---------------------------------------------------------------------------------------------------------------------------------------------------------------


 





















