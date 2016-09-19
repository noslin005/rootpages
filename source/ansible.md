# Ansible 2

* [Introduction](#introduction)
* [Configuration](#configuration)
  * [Inventory](#configuration---inventory)
    * [Variables](#configuration---inventory---variables)
* [Command Usage](#command-usage)
* [Playbooks](#playbooks)
  * [Directory Structure](#playbooks---directory-structure)
  * [Functions](#playbooks---functions)
    * [Async](#playbooks---functions---async)
    * [Ignore Errors](#playbooks---functions---ignore-errors)
    * [Include]
    * [Loops](#playbooks---functions---loops)
    * [Prompts](#playbooks---functions---prompts)
    * [Roles](#playbooks---functions---roles)
    * [Run Once](#playbooks---functions---run-once)
    * [Register](#playbooks---functions---register)
    * [Set Fact](#playbooks---functions---set-fact)
    * [Tags](#playbooks---functions---tags)
    * [When](#playbooks---functions---when)
  * [Modules](#playbooks---modules)
    * [Command and Shell](#playbooks---modules---command-and-shell)
    * [Copy Files and Templates](#playbooks---modules---copy-files-and-templates)
    * [Cron](#playbooks---modules---cron)
    * [Debug](#playbooks---modules---debug)
    * [Git](#playbooks---modules---git)
    * [MySQL Database]
    * [MySQL User]
    * [Service](#playbooks---modules---service)
    * [Package Managers](#playbooks---modules---package-managers)
      * [Yum](#playbooks---modules---package-managers---yum)
      * [Apt]

## Introduction
Ansible is a utility for managing server deployments and updates. The project is well known for the ease of deploying updated configuration files, it's backwards compatible nature, as well as helping to automate infrastructures. [1] Ansible uses SSH to connect to server's remotely. If an Ansible task is ever re-run on a server, it will verify if the task is truly necessary (ex., if a package already exists). [2] Ansible can be used to create [Playbooks](#playbooks) to automate deployment of configurations and/or services.

Sources:

1. Hochstein, *Ansible Up & Running*, 2.
2. "An Ansible Tutorial."

## Configuration
### Configuration - Inventory
Default file: /etc/ansible/hosts

The hosts file is referred to as the "inventory" for Ansible. Here servers and groups of servers are defined. Ansible can then be used to execute commands and/or Playbooks on these hosts.

The general syntax is:
```
[GROUPNAME]
SERVER1NAME
SERVER2NAME
```

Groups are created by using brackets "[" and "]" to specify the name. In this example, the group name is "dns-us" and contains three servers.
```
[dns-us]
dns-us1
dns-us2
dns-us3
```

A group can also be created from other groups by using the ":children" tag.
```
[dns-global:children]
dns-us
dns-ca
dns-mx
```

Variables are created for a host and/or group using the tag ":vars". Then any custom variable can be defined and associated with a string. A host specificially can also have it's variables defined on the same line as it's Ansible inventory variables. A few examples are listed below. These can also be defined in seperate files as explained in [Inventory - Variables](#inventory---variables).
```
examplehost ansible_user=toor ansible_host=192.168.0.1 custom_var_here=True
```
```
[examplegroup:vars]
domain_name=examplehost.tld
domain_ip=192.168.7.7
```

There are a large number of customizations that can be used to suit most server's access requirements.

Common Inventory Options:
* ansible_host = The IP address or hostname of the server.
* ansible_port = A custom SSH port (i.e., if not using the standard port 22).
* ansible_connection = These options specify how to log in to execute tasks. Valid options are "ssh" to remotely connect to the server (this is the default behavior), "local" to run as the current user, and "chroot" to change into a different main root file system.
* ansible_user = The SSH user.
* ansible_pass = The SSH user's password. This is very insecure to keep passwords in plain text files so it is recommended to use SSH keys or pass the "--ask-pass" option to ansible when running tasks.
* ansible_ssh_private_key_file = Specify the private SSH key to use for accessing the server(s).
* ansible_ssh_common_args = Append additionally SSH command-line arguments.
* ansible_python_interpreter = This will force Ansible to run on remote systems using a different Python binary. Ansible only supports Python 2 so on server's where only Python 3 is available, such as Arch Linux, a custom install of Python 2 can be used instead. [1]
* ansible_become = Set to "true" or "yes" to become a different user than the ansible_user once logged in.
  * ansible_become_method = Pick a method for switching users. Valid options are: sudo, su, pbrun, pfexec, doas, or dzdo.
  * ansible_become_user = Specify the user to become.
  * ansible_become_pass = Optionally ues a password to change users. [5]


Here is an example of common Ansible options that can be used.
```
localhost ansible_connection=local
dns1 ansible_host=192.168.1.53 ansilbe_port=2222 ansible_become=true ansible_become_user=root ansible_become_method=sudo
dns2 ansible_host=192.168.1.54
/home/user/ubuntu1604 ansible_connection=chroot
```

### Configuration - Inventory - Variables

Variables that Playbooks will use can be defined for specific hosts and/or groups.
* /etc/ansible/host_vars/ = Create a file named after the host.
* /etc/ansible/group_vars/ = Create a file named after the group of hosts.
* all = This file can exist under the above directories to define global variables for all hosts and/or groups.

The files in these directories can have any name and do not require an extension. It is assumed that they are YAML files. Here is an example for a host variable file.
```
---
domain_name: examplehost.tld
domain_ip: 192.168.10.1
hello_string: Hello World!
```

In the Playbook and/or template files, these variables can then be referenced when enclosed by double braces "{{" and "}}". [2]
```
Hello world from {{ domain_name }}!
```

Varialbes from other hosts or groups can also be refereneced.
```
{{ groupvars['<GROUPNAME>']['<VARIABLE>'] }}
{{ hostvars['<HOSTNAME>']['<VARIABLE>'] }}
```

Variables can be defined as a list or nested lists.

Syntax:
```
<VARIABLE>: [ '<ITEM1>', '<ITEM2>', '<ITEM3>' ]
```
```
<VARIABLE>:
 - [ [ '<ITEMA>', '<ITEMB>' ] ]
 - [ [ '<ITEM1>', '<ITEM2>' ] ]
```

Examples:
```
colors: [ 'blue', 'red', 'green' ]
```
```
cars:
 - [ 'sports', 'sedan' ]
 - [ 'suv', 'pickup' ]
```

Lists can be called by their array position, starting at "0." Alternatively they can be called by the subvariable name.

Syntax:
```
{{ item.0 }}
```
```
{{ item.0.<SUBVARIABLE> }}
```

Example:
```
# variables
members:
 - name: Frank
   contact:
    - address: "111 Puppet Drive"
    - phone: "1111111111"
```
```
- debug: msg="Contact {{ item.name }} at {{ item.contact.phone }}"
 with_items:
  - {{ members }}
```

[4]

The order that variables take precendence in is listed below. The bottom locations get overriden by anything above them.

* extra vars
* task vars
* block vars
* role and include vars
* set_facts
* registered vars
* play vars_files
* play vars_prompt
* play vars
* host facts
* playbook host_vars
* playbook group_vars
* inventory host_vars
* inventory group_vars
* inventory vars
* role defaults

[2]

Sources:

1. "Ansible Inventory"
2. "Ansible Variables."
3. "Ansible Best Practices."
4. "Ansible Loops."
5. "Ansible Become (Privlege Escalation)"

## Command Usage
Refer to Root Page's "Linux Commands" guide in the "Deployment" section.

## Playbooks
Playbooks organize tasks into one or more YAML files. It can be a self-contained file or a large project organized in a directory. Official examples can he found here at [https://github.com/ansible/ansible-examples](https://github.com/ansible/ansible-examples).

### Playbooks - Directory Structure

A Playbook can be self-contained entirely into one file. However, especially for large projects, each segment of the Playbook should be split into seperate files and directories.

* Layout:
```
├── group_vars/
├── host_vars/
├── roles/
│   └── general/
│       ├── defaults/
│       │   └── main.yml
│       ├── files/
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       └── vars/
│           └── main.yml
└── site.yml
```

* Layout Explained:
```
├── group_vars/ = Group specific variables. The "all" file can be used to define global variables for all hosts.
├── host_vars/ = Host specific variables.
├── roles/ = This directory should contain all of the different roles.
│   └── general/ = A role name. This can be anything.
│       ├── defaults/ = Define default variables. If any variables are defined elsewhere, these will be overriden.
│       │   └── main.yml = Each main.yml file is executed as the first file. Additional seperation of operations can be split into different files that can be accessed via "include:" statements.
│       ├── files/ = Store static files that are not modified.
│       ├── handlers/ = Specify alias commands that can be called using the "notify:" method.
│       │   └── main.yml
│       ├── meta/ = Specify role depdencies. These can be other roles and/or Playbooks.
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml = The tasks' main file is executed first for the entire role.
│       ├── templates/ = Store dynamic files that will be generated based on variables.
│       └── vars/ = Define role-specific variables.
│           └── main.yml
└── site.yml = This is typically the default Playbook file to execute. Any name and any number of Playbook files can be used here to include different roles.
```

* Examples:
  * site.yml = This is generally the main Playbook file. It should include all other Playbook files required if more than one is used. [2]

```
# FILE: site.yml
---
include: nginx.yml
include: php-fpm.yml
```
```
# FILE: nginx.yml
---
- hosts: webnodes
  roles:
    - common
    - nginx
```

  * roles/`<ROLENAME>`/vars/main.yml = Global variables for a role.

```
---
memcache_hosts=192.168.1.11,192.168.1.12,192.168.1.13
ldap_ip=192.168.10.1
```

  * group_vars/ and host_vars/ = These files define variables for hosts and/or groups. Details about this can be found in the [Variables](#configuration---inventory---variables) section.

  * templates/ = Template configuration files for services. The files in here end with a ".j2" suffix to signify that it uses the Jinja2 template engine. [1]

```
<html>
<body>My domain name is {{ domain }}</body>
</html>
```

Sources:

1. "An Ansible Tutorial."
2. “Ansible Best Practices.”


## Playbooks - Functions

### Playbooks - Functions - Async

The "async" function can be used to start a detached task on a remote system. Ansible will then poll the server periodically to see if the task is complete (by default, it checks every 10 seconds). Optionally a custom poll time can be set.

Syntax:
```
async: <SECONDS_TO_RUN>
```

Example #1:
```
- command: bash /usr/local/bin/example.sh
  async: 15
  poll: 5
```

[1]

Sources:

1. "Ansible Asynchronous Actions and Polling."

### Playbooks - Functions - Loops

Loops are a great way to reuse modules with multiple items. Essentially these are "for" loops. These loops can even utilize flattened lists to run an action on multiple items at once.

Variable syntax:
```
{{ item }}
```

Loop syntax:
```
with_items:
  - <ITEM1>
  - <ITEM2>
  - <ITEM3>
```

Example:
```
package: name={{ item }} state=latest
with_items:
 - nginx
 - php-fpm
 - mysql
```

Flattened example:
```
---
# variables
web_services: [ 'nginx', 'httpd', 'mysql' ]
```
```
service: name={{ item }} state=restarted
with_flattened:
 - "{{ web_services }}"
```

[1]

Sources:

1. "Ansible Loops."

### Playbooks - Functions - Ignore Errors

Playbooks, by default, will stop running if it fails to run a command. If it's okay to continue then the "ignore_errors" option can be appeneded below the command. This will allow the Playbook to continue onto the next step.

```
ignore_errors: yes
```

Sources:

1. "Ansible Error Handling In Playbooks."

### Playbooks - Functions - Prompts

Prompts can be used to assign a user's input as a variable.

Syntax:
```
vars_prompt:
  - name: "<VARIABLE>"
    prompt: "<PROMPT TEXT>"
```

Options:
 * confirm = Prompt the user twice and then verify that the input is the same.
 * encrypt = Encrypt the text.
   * md5_crypt
   * sha256_crypt
   * sha512_crypt
 * salt = Specify a string to use as a salt for encrypting.
 * salt_size = Specify the length to use for a randomly generated salt. The default is 8.

Examples:
```
vars_prompt:
  - name: "zipcode"
    prompt: "Enter your zipcode."
```
```
vars_prompt:
  - name: "pw"
    prompt: "Password:"
    encrypt: "sha512_crypt"
    salt_size: 12
```

[1]

Sources:

1. "Ansible Prompts."

### Playbooks - Functions - Run Once

In some situations a command should only need to be run on one node. A good example is when using a MariaDB Galera cluster where database changes will get synced to all nodes.

Syntax:
```
run_once: true
```

This can also be assigned to a specific node.
```
run_once:
delegate_to: <HOST>
```

[1]

Sources:

1. "Ansible Delegation, Rolling Updates, and Local Actions."

### Playbooks - Functions - Roles

A Playbook consists of roles. Each role that needs to be run needs to be specified. [1] It's important to note that individual roles cannot call other roles. Instead, an "include" statement could be used to find the relative path. [2]

Syntax:
```
roles:
  - <ROLE1>
  - <ROLE2>
```

Example:
```
roles:
  - common
  - httpd
  - sql
```

Sources:

1. "Ansible Playbook Roles and Include Statements."
2. "Ansible: Include Role in a Role?"

### Playbooks - Functions - Register

The output of commands can be saved to a variable. The attributes that are saved are:

* changed = If something was ran, this would be set to "true."
* stdout = The standard output of the command.
* stdout_lines = The standard output of the command is seperated by the newline characters into a list.
* stderr = The stanard error of the command.

[1]

The result of an action is saved as one of these three values. They can then be referenced later.
* succeeded
* failed
* skipped

[3]

Syntax:
```
register: cmd_output
```

Example #1:
```
- command: echo Hello World
  register: hello
- debug: msg="We heard you"
  when: "'Hello World' in hello.stdout"
```
[2]

Example #2:
```
- copy: src=example.conf dest=/etc/example.conf
  register: copy_example
- debug: msg="Copying example.conf failed."
  when: copy_example|failed
```

[3]

Sources:

1. "Ansible - some random useful things."
2. "Ansible Error Handling In Playbooks."
3. "Ansible Conditionals."

### Playbooks - Functions - Set Fact

New variables can be defined set the "set_fact" module. These are added to the available variables/facts tied to a inventory host.

Syntax:
```
set_fact:
  <VARIABLE_NAME1>: <VARIABLE_VALUE1>
  <VARIABLE_NAME2>: <VARIABLE_VALUE2>
```

Example:
```
- set_fact:
    is_installed: True
    sql_server: mariadb
```

Sources:

1. "Ansible Set host facts from a task."

### Playbooks - Functions - Tags

Each task in a tasks file can have a tag associted to it. This should be appended to the end of the task. This is useful for debugging and seperating tasks into specific groups. Here is the syntax:

Single tag syntax:
```
tags: <TAG>
```

Multiple tags syntax:
```
tags:
 - <TAG1>
 - <TAG2>
 - <TAG3>
```
Run only tasks with the defined tasks:
```
# ansible-playbook --tags <TAG1,TAG2,TAG3,etc.>
```
Example:
```
# head webserver.yml
---
- package: name=nginx state=latest
  tags:
   - yum
   - rpm
   - nginx
```
```
# ansible-playbook --tags yum webserver.yml webnode1
```

[1]

Sources:

1. "Ansible Tags."

### Playbooks - Functions - When

The "when" function can be uesd to specify that a sub-task should only run if the condition returns turn. This is similar to an "if" statement in programming langauges. It is usually the last line to a sub-task.

"When" Example:
```
- package: name=httpd state=latest
  when: ansible_os_family == "CentOS"
```
"Or" example:
```
when: ansible_os_family == "CentOS" or when: ansible_os_family == "Debian"
```
"And" example, using a cleaner style:
```
when: (ansible_os_family == "Fedora") and
      (ansible_distribution_major_version == "25")
```

[1]

Sources:

1. "Ansible Conditionals."

## Playbooks - Modules

### Playbooks - Modules - Command and Shell

Both the command and shell modules provide the ability to run command line programs. The big difference is that shell provides a full shell enviornment where operand redirection and pipping works, along with loading up all of the shell variables. Conversely, command will not load a full shell environment so it will lack in features and functionality but it makes up for that by being faster and more efficent. [1]

Common options:
* executable = Set the executable shell binary.
* chdir = Change directories before running the command.

Example:
```
- shell: echo "Hello world" >> /tmp/hello_world.txt
  args:
    executable: /bin/bash
```

Sources:

1. "Ansible Command Module."
2. "Ansible Shell Module."

### Playbooks - Modules - Copy Files and Templates

The copy, file and template modules provide ways to managing and configuring various files. The file module is used to handle file creation/modification [1], templates are to be used when Ansible needs to fill in the variables [2] and copy is used for copying files and folders. Most of the attributes are the same between the three modules.

Common options:
* src = Define the source file or template. If a full path is not given, Ansible will check in the roles/`<ROLENAME>`/files/ directory for a file or roles/`<ROLENAME>`/templates/ for a template.
* dest (or path) = This is the full path to where the file should be copied to on the destination server.
* remote_src = If set to True, the source file will be found on the server Ansible is running tasks on (not the local machine). The default is False.
* owner = Set the user owner.
* group = Set the group owner.
* mode = Set the octal or symbloc permissions. If using octal, it has to be four digits. The first digit is generally the flag "0" to indicate no special permissions.
* setype = Set SELinux file permissions.
* state = Specify the state the file should be created in.
  * file = Copy the file.
  * link = Create a soft link shortcut.
  * hard = Create a hard link reference.
  * touch = Create an empty file.
  * directory = Create all subdirectories in the destination folder.
  * absent = Delete destination folders. [1]

* Example:
  * Copy a template from roles/`<ROLE>`/templates/ and set the permissions for the file.
```
template: src=example.conf.j2 dst=/etc/example/example.conf mode=0644 owner=root group=nobody
```

Sources:

1. "Ansible File Module."
2. "Ansible Template Module."

### Playbooks - Modules - Debug
The debug module is used for helping facilitate troubleshooting. It prints out specified information to standard output.

* Options:
  * msg = Display a message.
  * var = Display a variable.
  * verbosity = Show more verbose information. The higher the number, the more verbose the information will be. [1]

* Example:
  * Print Ansible's hostname of the current server that the script is being run on.
```
debug: msg=The inventory host name is {{ inventory_hostname }}
```

Sources:

1. "Ansible Debug Module."

### Playbooks - Modules - Cron

The cron module is used to manage crontab entries. Crons are scheduled/automated tasks that run on Unix-like systems.

Options:
* user = Modify the specified user's crontab.
* job = Provide a command to run when the cron reaches the correct
* minute
* hour
* weeekday = Specify the weekday as a number 0 through 6 where 0 is Sunday and 6 is Saturday.
* month
* day = Specify the day number in the 30 day month.
* backup = Backup the existing crontab. The "backup_file" variable provides the backed up file name.
  * yes
  * no
* state
  * present = add the crontab
  * absent = remove an existing entry
* special_time
  * reboot
  * yearly or annually
  * monthly
  * weekly
  * daily
  * hourly

Example #1:
```
cron: job="/usr/bin/wall This actually works" minute="*/1" user=ubuntu
```

Example #2:
```
cron: job="/usr/bin/yum -y update" weekday=0 hour=6 backup=yes
```

[1]

Sources:

1. "Ansible cron - Manage cron.d and crontab entries."

### Playbooks - Modules - Git

Git is a utlity used for provisioning and versioning software. Ansible has built-in support for handling most Git-related tasks.

Options:
* repo = The full path of the repository.
* dest = The path to place/use the repository
* update = Pull the latest version from the Git server. The default is "yes."
* version = Switch to a different branch or tag.
* ssh_opts = If using SSH, specify custom SSH options.
* force = Overriden local changes. The default is "yes."

Source:

1. "Ansible Git Module"

### Playbooks - Modules - Service
The service module is used to handle system services.

Options:
* name = Specify the service name.
* enabled = Enable the service to start on boot or not. Valid options are "yes" or "no."
* sleep = When restarted a service, specify the amount of time (in seconds) to wait before starting a service after stopping it.
* state = Specify what state the service should be in.
  * started = Start the service.
  * stopped = Stop the service.
  * restarted = Stop and then start the service.
  * reloaded = If supported by the service, it will reload it's configuration file without restarting it's main thread. [1]

Exmaple:
* Restart the Apache service "httpd."
```
service: name=httpd state=restarted sleep=3
```

Sources:

1. "Ansible Service Module."

### Playbooks - Modules - Package Managers

Ansible has the ability to add, remove, or update software packages. Almost every popular package manager is supported. [1] This can generically be handled by the "package" module or the specific module for the operating system's package manager.

Options:

* name = Specify the package name.
* state = Specify how to change the package state.
  * present = Install the package.
  * latest = Update the package (or install, if necessary).
  * absent = Uninstall the package.
* use = Specify the package manager to use.
  * auto = Automatically detect the package manager to use. This is the default.
  * apt = Use Debian's Apt package manager.
  * yum = Use Red Hat's yum package manager. [2]

Example:
  * Update the MariaDB package.
```
package: name=mariadb state=latest
```

Sources:

1. "Ansible Packaging Modules."
2. "Ansible Generic OS package manager."

### Playbooks - Modules - Package Managers - Yum

There are two commands to primarily handel Red Hat's Yum package manager: "yum" and "yum_repository."

* yum options:
  * name = Specify the package name.
  * state = Specify the package state.
    * {present|installed|latest} = Any of these will install the package.
    * {absent|removed} = Any of these will uninstall the package.
  * enablerepo = Temporarily enable a repository.
  * disablerepo = Temporarily disable a repository.
  * disable_gpg_check = Disable the GPG check. The default is "no".
  * conf_file = Specify a Yum configuration file to use. [1]

* yum example:
  * Install the "wget" package with the EPEL repository enabled and disable GPG validation checks.
```
yum: name=wget state=installed enablerepo=epel disable_gpg_check=yes
```

* yum_repository options:
  * name = Specify a name for the repository. This is only required if the file is being created (state=present) or deleted (state=absent).
  * baseurl = Provide the URL of the repository.
  * mirrorlist = Provide a URL to a mirrorlist repository instead of the baseurl.
  * description = Required. Provide a description of the repository.
  * enabled = Enable the repository permanently to be active. The default is "yes."
  * exclude = List packages that should be excluded from being accessed from this repository.
  * gpgcheck = Validate the RPMs with a GPG check. The default is "no."
  * gpgkey = Specify a URL to the GPG key.
  * state = Specify a state for the repository file.
    * present = Install the Yum repository file. This is the default.
    * absent = Delete the repository file. [2]

* yum_repository example:
  * Install the RepoForge Yum repository.
```
yum_repository: name=repoforge baseurl=http://apt.sw.be/redhat/el7/en/x86_64/rpmforge/ enabled=no description="Third-party RepoForge packages"
```

Sources:

1. "Ansible Yum Module."
2. "Ansible Yum Repository Module."


---
Bibliography:

* Lorin Hochstein *Ansible Up & Running* (Sebastopol: O'Reilly Media, Inc., 2015).
* "An Ansible Tutorial." Servers for Hackers. August 26, 2014. Accessed June 24, 2016. https://serversforhackers.com/an-ansible-tutorial
* "Intro to Playbooks." Ansible Documentation. June 22, 2016. Accessed June 24, 2016.  http://docs.ansible.com/ansible/playbooks_intro.html
* "Ansible Frequently Asked Questions." Ansible Documentation. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/faq.html
* "Ansible Inventory." Ansible Docs. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/intro_inventory.html
* "Ansible Variables." Ansible Documentation. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/playbooks_variables.html
* "Ansible Best Practices." Ansible Documentation. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/playbooks_best_practices.html
* "Ansible File Module." Ansible Documentation. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/file_module.html
* "Ansible Template Module." Ansible Documentation. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/template_module.html
* "Ansible Service Module." Ansible Docs. June 22, 2016. Accessed July 9, 2016. http://docs.ansible.com/ansible/service_module.html
* "Ansible Packaging Modules." Ansible Documentation. June 22, 2016. Access July 10, 2016. http://docs.ansible.com/ansible/list_of_packaging_modules.html
* "Ansible Generic OS package manager" Ansible Documentation. June 22, 2016. Access July 10, 2016. http://docs.ansible.com/ansible/package_module.html
* "Ansible Yum Module." Ansible Documentation. June 22, 2016. Accessed July 10, 2016. http://docs.ansible.com/ansible/yum_module.html
* "Ansible Yum Repository Module." Ansible Documentation. June 22, 2016. Accessed July 10, 2016. http://docs.ansible.com/ansible/yum_repository_module.html
* "Ansible Command Module." Ansible Documentation. June 22, 2016. Accessed July 10, 2016. http://docs.ansible.com/ansible/yum_repository_module.html
* "Ansible Shell Module." Ansible Documentation. June 22, 2016. Accessed July 10, 2016. http://docs.ansible.com/ansible/yum_repository_module.html
* "Ansible Debug Module." Ansible Documentation. June 22, 2016. Accessed July 17, 2016. http://docs.ansible.com/ansible/debug_module.html
* "Ansible Git Module." Ansible Documentation. June 22, 2016. Accessed July 30, 2016. http://docs.ansible.com/ansible/git_module.html
* "Ansible Tags." Ansible Documentation. August 05, 2016. Accessed August 13, 2016. http://docs.ansible.com/ansible/playbooks_tags.html
* "Ansible Prompts." Ansible Documentation. August 05, 2016. Accessed August 13, 2016. http://docs.ansible.com/ansible/playbooks_prompts.html
* "Ansible Using Lookups." Ansible Documentation. August 05, 2016. Accessed August 13, 2016. http://docs.ansible.com/ansible/playbooks_lookups.html
* "Ansible Loops." Ansible Documentation. August 05, 2016. Accessed August 13, 2016. http://docs.ansible.com/ansible/playbooks_loops.html
* "Ansible Conditionals." Ansible Documentation. August 24, 2016. Accessed August 27th, 2016. http://docs.ansible.com/ansible/playbooks_conditionals.html
* "Ansible Error Handling In Playbooks." Ansible Documentation. August 24, 2016. Accessed August 27th, 2016. http://docs.ansible.com/ansible/playbooks_error_handling.html
* "Ansible - some random useful things." Code Poets. August 4, 2014. Accessed August 27th, 2016. https://codepoets.co.uk/2014/ansible-random-things/
* "Ansible Become (Privlege Escalation)." Ansible Documentation. August 24, 2016. Accessed August 27th, 2016. http://docs.ansible.com/ansible/become.html
* "Ansible Delegation, Rolling Updates, and Local Actions." Ansible Documentation. September 1, 2016. Accessed September 11th, 2016. http://docs.ansible.com/ansible/playbooks_delegation.html
* "Ansible Asynchronous Actions and Polling." Ansible Documentation. September 1, 2016. Accessed September 11th, 2016. http://docs.ansible.com/ansible/playbooks_async.html
* "Ansible Set host facts from a task." Ansible Documentation. September 1, 2016. Accessed September 11th, 2016. http://docs.ansible.com/ansible/set_fact_module.html
* "Ansible Playbook Roles and Include Statements." Ansible Documentation. September 1, 2016. Accessed September 11th, 2016. http://docs.ansible.com/ansible/playbooks_roles.html
* "Ansible: Include Role in a Role?" StackOverflow. October 24, 2014. http://stackoverflow.com/questions/26551422/ansible-include-role-in-a-role
* "Ansible cron - Manage cron.d and crontab entries." Ansible Documentation. September 13, 2014. Accessed September 15th, 2016. http://docs.ansible.com/ansible/cron_module.html