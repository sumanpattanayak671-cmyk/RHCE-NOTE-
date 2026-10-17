# SESSION 12 — Registering Variables in Ansible

## Contents
1. What registering a variable means
2. How registering works
3. Practical command-output example
4. Registered variable attributes
5. Using `register` with `when`
6. `stdout` and `stdout_lines`
7. Registering variables with loops
8. Registering variables with `shell`
9. Checking service status
10. Comparing `register` and `set_fact`
11. Important rules
12. Practice task and final revision

---

## Original Session Text

The original text from `SESSION 12.txt` is preserved below.

---

SESSION 12

---------------------------------

register variables in ansible 

1. What is registering a variable in Ansible?
In Ansible, registering a variable means storing the result of a task in a variable so that you can use that result in later tasks.
For example, suppose you execute a command on a managed node to check whether the httpd package is installed. You want to save the command's output and use it in another task.
You can use the register keyword.
---
- name: Register variable example
  hosts: all
  tasks:
    - name: Check hostname
      command: hostname
      register: host_output

    - name: Display hostname
      debug:
        var: host_output


Here:
- command: hostname executes the command on the managed node.
- register: host_output saves the task's result.
- debug: var=host_output displays the stored result.
Remember: register stores the result of a task, not just the output text.

2. How does registering a variable work?
Step 1: Ansible executes a task
Example: command: hostname

Step 2: The result is registered
register: host_output

Step 3: Ansible stores the result
Output, return code, status and other task information

Step 4: Use the registered variable
Display it, check a condition, or use it in another task


3. Practical example: Register command output
Create a file called register.yml on your Ansible control node.
---
- name: Demonstrate registering variables
  hosts: all
  gather_facts: false

  tasks:
    - name: Check hostname
      command: hostname
      register: host_result

    - name: Display hostname
      debug:
        msg: "The hostname is {{ host_result.stdout }}"


Run the playbook:
ansible-playbook register.yml


Suppose a managed node's hostname is node1. The relevant output will look like this:
TASK [Check hostname]
changed: [node1]

TASK [Display hostname]
ok: [node1] => {
    "msg": "The hostname is node1"
}


Notice this line:
register: host_result


It stores the result in host_result.
This expression:
{{ host_result.stdout }}


extracts the standard output of the command, which is node1.

4. Important attributes of a registered variable
A registered variable contains a dictionary of information about the task result. For command tasks, some of the most useful attributes are:
Attribute	Meaning	Example
stdout	Standard output as a string	node1
stdout_lines	Standard output as a list of lines	["node1"]
stderr	Standard error output	Error message, if any
rc	Return code of the command	0 means success for most commands
changed	Whether Ansible reports a change	true or false
failed	Whether the task failed	true or false
skipped	Whether the task was skipped	true or false
The exact attributes available depend on the module being used. These are common examples, especially for command and shell.
Example: Display individual attributes
---
- name: Explore registered variable attributes
  hosts: all
  gather_facts: false

  tasks:
    - name: Check operating system uptime
      command: uptime
      register: uptime_result

    - name: Display command output
      debug:
        var: uptime_result.stdout

    - name: Display return code
      debug:
        var: uptime_result.rc

    - name: Display complete result
      debug:
        var: uptime_result


To see only the command's output:
- name: Show uptime
  debug:
    msg: "{{ uptime_result.stdout }}"


To see the return code:
- name: Show return code
  debug:
    msg: "{{ uptime_result.rc }}"


To display the complete registered result:
- name: Show all available result information
  debug:
    var: uptime_result



5. Using registered variables with when
One of the most important uses of register is conditional task execution.
You can run a task only when a previous task's result meets a condition.
Example 1: Check whether a command succeeded
---
- name: Check command result
  hosts: all
  gather_facts: false

  tasks:
    - name: Check hostname
      command: hostname
      register: result

    - name: Display success message
      debug:
        msg: "Hostname command executed successfully"
      when: result.rc == 0


Explanation:
- result.rc contains the command's return code.
- result.rc == 0 checks whether the command succeeded.
- The debug task runs only when the condition is true.
Example 2: Check whether HTTPD is installed
For RHEL systems, use the RPM database to check whether the package is installed.
---
- name: Check HTTPD package
  hosts: all
  gather_facts: false

  tasks:
    - name: Check installed HTTPD package
      command: rpm -q httpd
      register: httpd_check
      changed_when: false
      failed_when: false

    - name: Display package status
      debug:
        msg: "HTTPD is installed"
      when: httpd_check.rc == 0

    - name: Display missing package status
      debug:
        msg: "HTTPD is not installed"
      when: httpd_check.rc != 0


Why use these two options?
- changed_when: false tells Ansible that this check does not change the system.
- failed_when: false prevents the play from stopping just because the package is absent.
The return code is then used to decide which message to display.
Note that a nonzero return code can indicate other errors, too, so production playbooks should distinguish a missing package from an unexpected RPM database problem if that distinction matters.

6. Using registered variables with stdout and stdout_lines
Suppose you execute:
- name: List files
  command: ls /etc
  register: files_result


You can access the result in two ways.
Using stdout:
- name: Display file listing
  debug:
    var: files_result.stdout


This stores the command output as a single string, possibly containing newline characters.
Using stdout_lines:
- name: Display file listing line by line
  debug:
    var: files_result.stdout_lines


This provides a list of individual lines, which is useful for looping.
For example:
- name: Display each file
  debug:
    msg: "{{ item }}"
  loop: "{{ files_result.stdout_lines }}"


This displays each line as a separate loop item.

7. Registering variables with loops
You can also register the results of tasks that run multiple times using a loop.
---
- name: Register loop results
  hosts: all
  gather_facts: false

  tasks:
    - name: Check multiple commands
      command: "{{ item }}"
      loop:
        - hostname
        - whoami
        - uptime
      register: command_results
      changed_when: false

    - name: Display all results
      debug:
        var: command_results


When a task uses a loop and register, the result is generally stored in a results list.
Each item in that list contains the result of one loop iteration.
For example, the structure resembles:
command_results:
  changed: false
  results:
    - item: hostname
      stdout: node1
      rc: 0
    - item: whoami
      stdout: root
      rc: 0
    - item: uptime
      stdout: "14:30:00 up 2 days"
      rc: 0


The output above is illustrative; actual uptime text, user names, and result fields vary.
To display the output from every command:
- name: Display each command result
  debug:
    msg: "{{ item.stdout }}"
  loop: "{{ command_results.results }}"


Important: A registered variable from a loop is structured differently from a variable registered by a single task. The individual results are inside .results.

8. Registering variables with the shell module
The shell module is useful when you need shell features such as pipes, redirection, or shell operators.
Example:
---
- name: Register shell output
  hosts: all
  gather_facts: false

  tasks:
    - name: Count running processes
      shell: "ps -e | wc -l"
      register: process_count
      changed_when: false

    - name: Display process count
      debug:
        msg: "Number of listed processes: {{ process_count.stdout }}"


Here, process_count.stdout contains the output from the pipeline.
Use command when you do not need shell functionality. It is generally the safer, simpler choice.

9. Registering a variable for service status
Suppose you want to check the HTTPD service on your RHEL managed nodes.
---
- name: Check HTTPD service
  hosts: all
  gather_facts: false

  tasks:
    - name: Check HTTPD status
      command: systemctl is-active httpd
      register: httpd_status
      changed_when: false
      failed_when: false

    - name: Display active status
      debug:
        msg: "HTTPD service is running"
      when: httpd_status.stdout == "active"

    - name: Display inactive status
      debug:
        msg: "HTTPD service is not active"
      when: httpd_status.stdout != "active"


The systemctl is-active command returns active when the service is running. If the service is inactive or missing, the output and return code may differ.
The registered variable allows later tasks to inspect that result.

10. Registering variables with set_fact
This is a common source of confusion.
register and set_fact are different.
Feature	register	set_fact
Purpose	Saves the result of a task	Defines a variable from a value or expression
Typical use	Save command/module output	Calculate or build a variable
Example	register: result	set_fact: total: 10
Example combining both:
---
- name: Compare register and set_fact
  hosts: all
  gather_facts: false

  tasks:
    - name: Get hostname
      command: hostname
      register: host_result
      changed_when: false

    - name: Create a new variable from the result
      set_fact:
        custom_message: "Managed node is {{ host_result.stdout }}"

    - name: Display custom variable
      debug:
        var: custom_message


The first task registers the command result. The second task creates a new variable using that result.

11. Important rules to remember
1. register is written at the task level, alongside command, shell, copy, service, or another task module.
2. You can use a registered variable in subsequent tasks within the same play.
3. Registered results are host-specific: each managed host gets its own result.
4. A registered variable is available even if its task fails, provided the task ran and produced a result.
5. If a task is skipped, the registered result may have a skipped attribute rather than the normal output attributes.
6. Registering a result does not automatically make a playbook fail or succeed; Ansible's task status and failure-handling rules still apply.
7. Registered variables are generally available only for the current playbook run, not automatically saved permanently across future runs.
8. Be careful when displaying full registered results because they may contain sensitive information, such as command output or credentials.

12. Practice task for your Ansible lab
Try this exercise on your control node and managed nodes.
Objective: Check whether HTTPD is installed, save the result, and display the appropriate message.
Your task
- [ ] Execute rpm -q httpd on all managed nodes.
- [ ] Register the result in a variable named package_result.
- [ ] Use package_result.rc to determine whether the package is installed.
- [ ] Display a suitable message using the debug module.
  
  Show solution

Final revision
The most important concept is this:
- name: Execute a task
  command: hostname
  register: result

- name: Use the stored result
  debug:
    msg: "{{ result.stdout }}"


In one sentence: Registering variables in Ansible lets you capture task results and reuse them for displaying information, checking conditions, processing command output, and controlling subsequent tasks.