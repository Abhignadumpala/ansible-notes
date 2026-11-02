# Ansible Cheat Sheet

[← Back to all topics](README.md)

## About this cheat sheet

All the everyday Ansible commands on one page, grouped by job. Each section opens with a one-line definition. The full explanations are in the topic notes.

| Section | Covers |
|---|---|
| [1. Key words](#1-key-words) | play, task, module, role, vars, register, notify, handler |
| [2. Install](#2-install) | Amazon Linux, RHEL, Ubuntu, pip |
| [3. Ad-hoc commands](#3-ad-hoc-commands) | ping, commands, facts, copy, packages, services |
| [4. Playbook commands](#4-playbook-commands) | run, check, limit, tags, start-at-task |
| [5. Command-line options](#5-command-line-options) | `-i -u -k -b -K -f -T -t -e -l -v` |
| [6. Output colours](#6-output-colours) | what green, yellow, cyan and red mean |
| [7. Inventory commands](#7-inventory-commands) | list, graph, host vars, patterns |
| [8. Variables on the command line](#8-variables-on-the-command-line) | `-e`, vars from a file |
| [9. Vault (secrets)](#9-vault-secrets) | encrypt, edit, view, decrypt |
| [10. Galaxy (roles and collections)](#10-galaxy-roles-and-collections) | init, install, list |
| [11. Documentation](#11-documentation) | `ansible-doc` |
| [12. Troubleshooting](#12-troubleshooting) | verbosity, syntax, step, facts |

---

## 1. Key words

| Word | One-line meaning |
|---|---|
| **play** | a set of tasks for a set of hosts (`hosts:` + `tasks:`) |
| **task** | one action on the hosts; runs one module |
| **module** | a reusable program that does one job (`copy`, `user`, `service`) |
| **role** | a standard folder of tasks, handlers, vars, templates and files for reuse |
| **vars** | define variables for the play |
| **register** | save a task's result in a variable |
| **notify** | tell a handler to run if this task **changed** something |
| **handler** | a task that runs only when notified, once, at the end of the play |
| **become** | run as root (sudo) |

**Common modules:**

| Module | Does |
|---|---|
| `package` | install software on any OS (picks `dnf` / `apt`) |
| `dnf` / `yum` | packages on Amazon Linux / RHEL |
| `apt` | packages on Ubuntu / Debian |
| `copy` | copy a file from the control node to the hosts (or write `content:`) |
| `file` | create folders, set permissions, delete files |
| `template` | copy a Jinja2 (`.j2`) file with variables filled in |
| `service` | start / stop / restart / enable a service |
| `user` | create or delete users |
| `command` / `shell` | run a command (`shell` allows pipes and `>`) |
| `cron` | manage scheduled jobs |
| `git` | clone or update a Git repository on the hosts |
| `debug` | print a message or a variable |

## 2. Install

**Definition:** Ansible is installed **only on the control node**.

| OS | Command |
|---|---|
| Amazon Linux 2023 | `sudo dnf install ansible-core -y` |
| RHEL / CentOS | `sudo dnf install ansible-core -y` (older: `yum`) |
| Ubuntu / Debian | `sudo apt update && sudo apt install ansible -y` |
| Any OS with Python (pip) | `python3 -m pip install --user ansible` |
| Check | `ansible --version` |

## 3. Ad-hoc commands

**Definition:** **ad-hoc command** = run one module from the command line, no playbook. Form: `ansible <hosts> -m <module> -a "<args>"`.

| Job | Command |
|---|---|
| check Ansible can log in | `ansible all -m ping` |
| run a command | `ansible web -a "uptime"` (default module is `command`) |
| run a command with a pipe | `ansible web -m shell -a "df -h \| grep /dev"` |
| gather facts | `ansible web1 -m setup` |
| one fact only | `ansible web1 -m setup -a "filter=ansible_distribution*"` |
| copy a file | `ansible all -m copy -a "src=file.txt dest=/tmp/file.txt"` |
| install a package | `ansible web -b -m package -a "name=nginx state=present"` |
| restart a service | `ansible web -b -m service -a "name=nginx state=restarted"` |
| create a user | `ansible all -b -m user -a "name=devuser state=present"` |

> Installing, restarting services and creating users need root: add **`-b`**.

## 4. Playbook commands

**Definition:** **`ansible-playbook`** = runs a playbook file.

| Job | Command |
|---|---|
| run | `ansible-playbook site.yaml` |
| check the YAML only | `ansible-playbook site.yaml --syntax-check` |
| dry run (change nothing) | `ansible-playbook site.yaml --check` |
| dry run + show file differences | `ansible-playbook site.yaml --check --diff` |
| list the tasks | `ansible-playbook site.yaml --list-tasks` |
| list the hosts | `ansible-playbook site.yaml --list-hosts` |
| only some hosts | `ansible-playbook site.yaml -l web1` |
| start from one task | `ansible-playbook site.yaml --start-at-task "Install nginx"` |
| only tasks with a tag | `ansible-playbook site.yaml -t nginx` |
| skip tasks with a tag | `ansible-playbook site.yaml --skip-tags nginx` |
| list the tags | `ansible-playbook site.yaml --list-tags` |

## 5. Command-line options

**Definition:** the same options work for `ansible` and `ansible-playbook`.

```bash
ansible-playbook -i inventory.ini -u ec2-user -b -f 10 -T 20 -t nginx site.yaml
```

| Option | Means |
|---|---|
| `-i <file>` | which inventory |
| `-u <user>` | SSH username |
| `-k` | ask for the SSH password |
| `-b` | become root (sudo) |
| `-K` | ask for the sudo password |
| `-f <n>` | forks: how many hosts to work on **in parallel** (default 5) |
| `-T <sec>` | connection timeout, so unreachable hosts don't hang the run |
| `-t <tag>` | run only tasks with this tag |
| `-e "k=v"` | extra variables (highest priority) |
| `-l <host>` | limit to these hosts |
| `-v` … `-vvvv` | more detail in the output |
| `-M <path>` | where to find your own modules |

## 6. Output colours

**Definition:** Ansible colours each task result so you can read a run at a glance.

| Colour | Status | Means |
|---|---|---|
| 🟢 Green | `ok` | already correct, **nothing changed** |
| 🟡 Yellow | `changed` | Ansible **changed** something |
| 🔵 Cyan | `skipping` | task skipped (its condition was false) |
| 🔴 Red | `failed` / `unreachable` | error, or couldn't connect |
| 🟣 Purple | `[WARNING]` / `[DEPRECATION WARNING]` | a warning; the run continues |

## 7. Inventory commands

**Definition:** **`ansible-inventory`** = shows what Ansible understands from an inventory. It only reads; it doesn't connect.

| Job | Command |
|---|---|
| tree of groups and hosts | `ansible-inventory -i inventory.ini --graph` |
| tree with variables | `ansible-inventory -i inventory.ini --graph --vars` |
| everything as JSON | `ansible-inventory -i inventory.ini --list` |
| one host's variables | `ansible-inventory -i inventory.ini --host web1` |
| which hosts match a pattern | `ansible 'web*' -i inventory.ini --list-hosts` |
| refresh a dynamic inventory's cache | `ansible-inventory -i aws_ec2.yaml --graph --flush-cache` |

**Host patterns** (where you write the group name):

| Pattern | Matches |
|---|---|
| `all` | every host |
| `web*` | names starting with `web` |
| `webservers:databases` | in either group |
| `webservers:!web2` | webservers, except web2 |
| `webservers:&prod` | in both groups |

## 8. Variables on the command line

**Definition:** **`-e`** (`--extra-vars`) = pass variables when running. It beats every other place.

| Job | Command |
|---|---|
| one variable | `ansible-playbook site.yaml -e "app_port=8080"` |
| several | `ansible-playbook site.yaml -e "app_port=8080 app_env=prod"` |
| a value with spaces | `ansible-playbook site.yaml -e "app_name='My App'"` |
| from a file | `ansible-playbook site.yaml -e @vars.yaml` |

## 9. Vault (secrets)

**Definition:** **Ansible Vault** = encrypts files or values (passwords, keys) so they can be stored safely in Git.

| Job | Command |
|---|---|
| create an encrypted file | `ansible-vault create secrets.yaml` |
| encrypt an existing file | `ansible-vault encrypt secrets.yaml` |
| view it | `ansible-vault view secrets.yaml` |
| edit it | `ansible-vault edit secrets.yaml` |
| decrypt it | `ansible-vault decrypt secrets.yaml` |
| change its password | `ansible-vault rekey secrets.yaml` |
| encrypt one value | `ansible-vault encrypt_string 'MyPassw0rd' --name 'db_password'` |
| run a playbook that uses vault | `ansible-playbook site.yaml --ask-vault-pass` |

## 10. Galaxy (roles and collections)

**Definition:** **Ansible Galaxy** = the public hub of ready-made roles and collections. `ansible-galaxy` downloads them and creates role skeletons.

| Job | Command |
|---|---|
| create a new role folder | `ansible-galaxy role init webserver` |
| install a role | `ansible-galaxy role install geerlingguy.nginx` |
| install a collection | `ansible-galaxy collection install amazon.aws` |
| install from a list | `ansible-galaxy install -r requirements.yaml` |
| list installed roles | `ansible-galaxy role list` |
| list installed collections | `ansible-galaxy collection list` |
| search for roles | `ansible-galaxy role search nginx` |

## 11. Documentation

**Definition:** **`ansible-doc`** = offline docs for modules, plugins and keywords.

| Job | Command |
|---|---|
| list all modules | `ansible-doc -l` |
| one module (full, with examples) | `ansible-doc user` |
| one module (short) | `ansible-doc -s user` |
| a playbook keyword | `ansible-doc -t keyword become` |
| list inventory plugins | `ansible-doc -t inventory -l` |

## 12. Troubleshooting

| Problem | Command |
|---|---|
| need more detail | `ansible-playbook site.yaml -vvv` (`-vvvv` shows SSH details) |
| is the YAML valid? | `ansible-playbook site.yaml --syntax-check` |
| what would change? | `ansible-playbook site.yaml --check --diff` |
| go one task at a time | `ansible-playbook site.yaml --step` |
| can Ansible log in? | `ansible <host> -m ping` |
| what does the host look like? | `ansible <host> -m setup` |
| which config file is used? | `ansible --version` (`config file =` line) |
| which settings did I change? | `ansible-config dump --only-changed` |

| Error | Usual cause |
|---|---|
| `UNREACHABLE ... timed out` | security group / firewall blocks port 22, or wrong IP |
| `Permission denied (publickey)` | wrong user, wrong key, or key permissions not `400` |
| `... is undefined` | a `{{ variable }}` isn't defined anywhere |
| `Missing sudo password` / `permission denied` on a task | add `become: true` (and `-K` if sudo needs a password) |
| `could not find expected ':'` | YAML indentation or a missing space after `:` |
