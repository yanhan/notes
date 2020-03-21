# About

Some learnings from Ansible Up & Running by Lorin Hochstein.


## Terminology

- Playbook: a list of Plays
- Play: a set of hosts and set of tasks to run on those hosts. This is what we personally think of as "the top level".


## ansible.cfg

Specifiies some default configuration and variables.

Different locations for this file (from highest to lowest precedence):

- `ANSIBLE_CONFIG` environment variable
- `./ansible.cfg` (in same directory)
- `~/.ansible.cfg`
- `/etc/ansible/ansible.cfg`


## Handlers

- Handlers only run after all tasks are run.
- They only run once, even if notified multiple times.
- They run in order of their definition in the yaml file, not the order of notification.
- They are only fired if the task which fires them has a change of state.


## Inventory file

The inventory file is in INI format.

Basic example:
```
host1 param1 param2
host2 param1 param2
```

The parameters are **behavioral inventory parameters**. A lot of them are SSH settings.

Concretely:
```
vagrant1 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2220
vagrant2 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2221
```

Built in host groups: `all`, `*`. Both refer to the same thing.

### Grouping hosts together

Use INI headers. Hosts can overlap. Example:
```
[production]
web
mobile_api
elasticsearch

[stg]
email
web_stg
es_stg

[web]
web
web_stg

[elasticsearch]
elasticsearch
es_stg
```

You can specify groups that consist of other groups:
```
[app_one:children]
web
elasticsearch
```

You can also specify host and group variables in the inventory file, like so:
```
[all:vars]
office_cidr = "10.50.0.0/16"

[production:vars]
db_host = "db.examplecompany.com"
```

### Dynamic inventory

These are normally scripts that are run to get the inventory. You have to add executable permissions to the file for Ansible to get the dynamic inventory by running the file instead of reading it.

The output of the script must be in JSON and has a specific format. For more information, please go read the docs.

### Adding hosts and groups to inventory at runtime

Check out the `add_host` and `group_by` modules.

### Patterns for specifying hosts

|Type|Example|
|---|---|
|specific group|`hosts: db`|
|all hosts|`hosts: all`|
|Union of 2 groups (use colon sign)|`hosts: stg:prod`|
|Intersection (use colon ampersand)|`hosts: stg:&db`|
|Exclusion|`hosts: stg:!db`|
|Wildcard|`hosts: *.domain.com`|
|Range of numbers|`hosts: db[5:8]`|
|Regex (must start with tilde)|`hosts: ~web\d\.domain\.net`|

### Limit hosts to run tasks on

Use the `-l` flag to `ansible-playbook` in combination with one of the above pattern types:
```
ansible-playbook -l stg:&db
```

This will run on hosts which are belong to both group `stg` and `db`.


## Variables

`gather_facts`: facts are a special type of variable. You can use the `setup` module to view the facts of a host, like so:
```
ansible servername -m setup
```

Filtering of facts is possible:
```
ansible servername -m setup -a 'filter=ansible_eth*'
```

Modules can return facts. If a module returns facts, there is no need to register a variable to get the facts; simply run the module and the facts will be populated. Eg. `ec2_facts` module.

### Built in variables

|variable name|Explanation|
|---|---|
|`hostvars`|dict. keys are host names, values are dicts of variable names -> values|
|`inventory_hostname`|name of current host|
|`group_names`|list of groups that the current host is a member of|
|`groups`|dict. keys are group names. Each value is a list of host names for that group|
|`play_hosts`|list of inventory hostnames activate in current play|
|`ansible_version`|dict with ansible version info|


## `local_action`

There is some confusion on how to supply arguments / the module name for this. The simplest way to use the `module` argument to supply the module name. The rest of the arguments to that module can be supplied as before. Eg.
```
- name: Create config file
  local_action:
    module: file
    path: /etc/myapp.conf
    owner: root
    group: root
    mode: "0640"
```


## Lookups

Read data from various sources. eg. file, password, pipe, env, template, csvfile, dnstxt, redis\_kv, etcd.

Example usage:
```
arg="{{ lookup('filename', '/path/to/file') }}"
```

To write your own lookup plugin, read the source code for lookup plugins that ship with Ansible. May be at `/usr/share/ansible_plugins/lookup_plugins`.


## Loop types

`with_items`, `with_lines`, `with_fileglob`, `with_first_found`, `with_dict`, `with_flattened`, `with_indexed_items`, `with_nested`, `with_random_choice`, `with_sequence`, `with_subelements`, `with_together`, `with_inventory_hostnames`

Lookup the official docs for more details.


## Speeding up Ansible

### Method 1: Pipelining

This is disabled by default. To enable it, modify `ansible.cfg` to include:
```
[defaults]
pipelining = True
```

and ensure that every user used by Ansible does not have `requiretty` enabled in the sudoers file. Remember to use `visudo -cf %s` to validate the sudoers file.


### Method 2: Turn off `gather_facts` if you are not using any facts.

As mentioned in the heading.

### Method 3: Fact caching

Warning: There may be stale data and this may lead to some debugging difficulties. To clear the fact cache, pass `--flush-cache` to `ansible-playbook`.

In `ansible.cfg`, add the following:
```
[defaults]
gathering = smrt
fact_caching_timeout = 86400
# specify fact caching implementation, see below for an example
fact_caching = ...
```

These are the fact caching implementations: JSON files, Redis, memcached.

Example for JSON files:
```
fact_caching = jsonfile
fact_caching_connection = /path/to/the/file
```

### Method 4: Parallelism

Use the `ANDSIBLE_FORKS` environment variable to specify the number of hosts to connect to in parallel.


## List of useful tips and tricks

### Passing environment variables to tasks

This is applicable for any task. Just add the `environment` block. Eg:
```
- name: My task name
  module_name:
    arg1: value1
    arg2: value2
  environment:
    envvar1: value1
    envvar2: value2
```

### `creates` arg

An argument of the `command` module. Indicates that the command will create a file.

This specifies that a file will be created by the task.

### `wait_for` module

Use case: wait for machine's SSH port (or other port) to be ready.
```
- name: wait for SSH server
  local_action: wait_for port=22 host="{{ inventory_hostname }}" search_regex=OpenSSH
```

### `delegate_to`

Use this to run a task on a different host other than the current host.
```
- name: my task
  module_name:
    arg1: value1
  delegate_to: dbhost
```

### Run play X hosts at a time and abort once certain percentage of hosts fail

Use `serial` and `max_fail_percentage`.

### Running task once

To run a task once even if there are multiple hosts, use `run_once: True`

### Changing criteria of change of state / task failure

Use `changed_when` and `failed_when`.
```
- name: task name
  module_name:
    arg1: value1
  register: result
  failed_when: "error found at line" in result.stdout
```

### filters for task return values

These Jinja2 filters are: `failed`, `changed`, `success`, `skipped`. For instance:
```
failed_when: task1_output|failed
```

### Passing variables to roles in a play

```
roles:
  - role: myrole
    var1: value1
    var2: value2
```

### Dependent roles

Specify that a role is dependent on another role. A file is required at `roles/rolename/meta/main.yml` with such contents:
```
dependencies:
  - { role: rolewedependon }
```

### Validate files

The `copy` and `template` modules have a `validate` argument to validate files before the final copy. This is very useful for sudoers file.


## Debugging

### `debug` module

Used to output the value of a variable.

### `assert` module

Fails if a condition is not met

### Linting

```
ansible-playbook --syntax-check <playbook>
```

### List hosts but do not run tasks

```
ansible-playbook --list-hosts <playbook>
```

### Check mode

Kind of like dry run. However, it may be inaccurate due to conditionals:
```
ansible-playbook --check <playbook>
```

### diff

Shows diffs of files which will be changed on a run:
```
ansible-playbook --diff <playbook>
```

### Listing tasks in playbook

```
ansible-playbook --list-tasks <playbook>
```

### Prompt before running each task

```
ansible-playbook --step <playbook>
```

### Start at specific task instead of beginning

```
ansible-playbook --start-at-task="task name" <playbook>
```

### Skip tags

```
ansible-playbook --skip-tags=tagOne,tagTwo <playbook>
```


## Digression

`xip.io` is a service that redirects DNS names to an IP address that is prefixed in the DNS name. Eg. `192.168.99.10.xip.io` redirects to `192.168.99.10`.
