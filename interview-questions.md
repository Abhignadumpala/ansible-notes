# Ansible Interview Questions

[← Back to all topics](README.md)

## About this page

Short, clear answers, grouped by topic, from basic to advanced. Scenario questions include a working example.

| # | Section |
|---|---|
| ⭐ | [Very Important](#-very-important) |
| 1 | [Basics and architecture](#1-basics-and-architecture) |
| 2 | [Inventory](#2-inventory) |
| 3 | [Modules, ad-hoc commands and idempotency](#3-modules-ad-hoc-commands-and-idempotency) |
| 4 | [Playbooks](#4-playbooks) |
| 5 | [Variables and facts](#5-variables-and-facts) |
| 6 | [Roles](#6-roles) |
| 7 | [Vault and secrets](#7-vault-and-secrets) |
| 8 | [Advanced features](#8-advanced-features) |
| 9 | [Debugging and performance](#9-debugging-and-performance) |
| 10 | [Enterprise and CI/CD](#10-enterprise-and-cicd) |
| 11 | [Scenario questions](#11-scenario-questions) |

---

## ⭐ Very Important

**1. If the same variable name is defined in many places, which value does Ansible use?**

The one with the highest precedence. Extra vars (`-e`) always win, then task vars, `vars_files`, `vars_prompt`, play vars, and inventory vars are lowest of these. More specific beats more general (task > play, host > group).

---

## Other Questions

## 1. Basics and architecture

**1. What is Ansible?**

An open-source automation tool for configuration management, application deployment, provisioning and orchestration. It's agentless (SSH/WinRM), and you describe the work in YAML playbooks.

**2. What are the main components of Ansible's architecture?**

| Component | Job |
|---|---|
| Control node | where Ansible is installed and playbooks run |
| Managed nodes | the target servers (no agent) |
| Inventory | which servers |
| Playbooks | what to do |
| Modules | the tools that do the work |
| Plugins | connection (SSH/WinRM), output, inventory, etc. |

The control node **pushes** tasks over SSH (Linux) or WinRM (Windows).

**3. What does "agentless" mean, and how is Ansible different from Puppet and Chef?**

Agentless = nothing installed on managed nodes; Ansible uses SSH/WinRM.

| | Ansible | Puppet / Chef |
|---|---|---|
| Agent on nodes | ❌ no | ✅ yes |
| Model | **push**: control node sends tasks | **pull**: agents poll a master every ~30 min |
| Language | YAML | Puppet DSL / Ruby |
| Setup | easy (just SSH) | master server + agents + certificates |
| Good for | ad-hoc tasks, deployments, orchestration | constant enforcement of state |

**4. What are the key features of Ansible?**

Agentless · declarative YAML · idempotent · playbooks · thousands of modules (built-in + custom) · static and dynamic inventory · roles for reuse.

**5. What is Infrastructure as Code (IaC), and how does Ansible fit in?**

IaC = managing infrastructure with code files instead of manual clicks, so it's repeatable, reviewed and kept in Git. Ansible playbooks are IaC: readable YAML, version-controlled, idempotent, reusable through roles. (Terraform is mostly used to **create** infrastructure; Ansible to **configure** it.)

**6. Can Windows be the control node?**

No. The control node must be Linux or macOS. Windows can only be a managed node (over WinRM).

**7. What do managed nodes need?**

SSH (port 22 open from the control node), Python, and a user Ansible can log in with. Windows nodes need WinRM and PowerShell.

**8. Explain Ansible's workflow.**

Read inventory → read playbook → SSH to each host → gather facts → copy and run each module → collect the JSON result → delete the module → print PLAY RECAP.

## 2. Inventory

**9. What is an inventory, and where is the default one?**

The list of hosts and groups Ansible manages. Default `/etc/ansible/hosts`; override with `-i` or `inventory =` in `ansible.cfg`.

**10. What are the types of inventory?**

- **Static:** a file you write by hand (INI or YAML). For labs and stable setups.
- **Dynamic:** fetched at run time from a source like AWS, Azure or a CMDB (`amazon.aws.aws_ec2` plugin). For cloud and auto scaling, where servers come and go.

**11. Host vs group?**

A host is one machine; a group is a name for many hosts. `[name:children]` makes a group of groups. `all` and `ungrouped` always exist.

**12. What is `ansible_host` used for?**

The real IP/hostname to connect to, when the inventory name is a short alias (`web1 ansible_host=10.0.0.5`).

**13. How do you check an inventory?**

`ansible-inventory -i inv --graph` (tree), `--list` (JSON), `--host web1` (one host). Only `ansible <group> -m ping` proves Ansible can actually log in.

**14. How do you run a playbook with a specific inventory file?**

```bash
ansible-playbook -i /path/to/inventory myplaybook.yaml
```

## 3. Modules, ad-hoc commands and idempotency

**15. What is a module?**

A small standalone program that does one job on a managed node (`user`, `package`, `copy`, `service`). Every module returns JSON with `changed` / `failed`. Built-in modules come with Ansible; custom modules are written by users (usually in Python).

**16. What is idempotency, and how do modules like `apt` and `copy` achieve it?**

Idempotent = running many times gives the same result. Modules check the current state first: `apt` checks if the package is already installed, `copy` compares checksums. If already correct, they do nothing (`ok`). This makes playbooks safe to re-run.

**17. `command` and `shell` are not idempotent. How do you make them safe?**

1. Prefer a real module if one exists.
2. Use `creates:` / `removes:`: skip the command if a file already exists / doesn't exist.
3. `register` the output and control the result with `changed_when` / `when`.

```yaml
- name: Extract the app only once
  ansible.builtin.command: tar -xzf /tmp/app.tar.gz -C /opt
  args:
    creates: /opt/app          # skipped if /opt/app already exists

- name: Check app version
  ansible.builtin.command: /opt/app/bin/app --version
  register: ver
  changed_when: false          # only reads, never "changed"
```

**18. `command` vs `shell` module?**

`command` runs directly (no pipes, redirects or `$HOME`). `shell` runs through `/bin/sh`, so `|`, `>`, `&&` work. Both always report `changed`.

**19. `package` vs `dnf` / `apt`?**

`package` is generic: it detects the OS and uses the right package manager. `dnf` / `apt` are OS-specific with more options.

**20. What is an ad-hoc command, and when do you use it?**

A one-line command that runs one module without a playbook, for quick checks and fixes.

```bash
ansible group1 -m shell -a "df -h"
ansible web -b -m service -a "name=nginx state=restarted"
```

**21. What is FQCN?**

Fully Qualified Collection Name, e.g. `ansible.builtin.copy` instead of `copy`. It names the collection and avoids clashes.

**22. How do you see a module's options?**

`ansible-doc <module>` (full) or `ansible-doc -s <module>` (short).

## 4. Playbooks

**23. What is the structure of a playbook?**

A list of plays. Each play has `name`, `hosts`, optional `become` / `vars`, and `tasks`. Each task has a `name` and runs one module.

**24. How do you write a basic playbook to install a package on a group of servers?**

```yaml
- name: Install a package
  hosts: webservers
  become: true
  tasks:
  - name: Install git
    ansible.builtin.package:
      name: git
      state: present          # 'latest' = always upgrade to the newest
```

**25. Install nginx with a playbook (Ubuntu).**

```yaml
- name: Install nginx
  hosts: webservers
  become: true
  tasks:
  - name: Update apt cache
    ansible.builtin.apt:
      update_cache: true

  - name: Install nginx
    ansible.builtin.apt:
      name: nginx
      state: present

  - name: Start and enable nginx
    ansible.builtin.service:
      name: nginx
      state: started
      enabled: true
```

Use `dnf` (or `package`) on Amazon Linux / RHEL.

**26. What does `become: yes` do?**

Runs tasks as root (sudo). Needed for packages, services and users. `-K` asks for the sudo password if one is required.

**27. What is a handler, and what problem does it solve?**

A task that runs only when another task **notifies** it and **changed** something, once, at the end of the play. It avoids needless restarts: nginx is restarted only when its config actually changed.

**28. How do you check the status of a service on many servers?**

```yaml
- name: Check service status
  hosts: webservers
  tasks:
  - name: Collect service facts
    ansible.builtin.service_facts:

  - name: Show nginx state
    ansible.builtin.debug:
      msg: "{{ ansible_facts.services['nginx.service'].state | default('not installed') }}"
```

`service_facts` takes no options; it collects **all** services into `ansible_facts.services`. Quick version: `ansible web -a "systemctl is-active nginx"`.

**29. How do you test a playbook without changing servers?**

`--syntax-check` validates the YAML; `--check` is a dry run that shows what would change; add `--diff` to see file changes.

**30. Why can a playbook have many plays?**

Each play can target different hosts, so one run can set up the database, then the backend, then the frontend in order (orchestration).

## 5. Variables and facts

**31. `http_port` is in `group_vars/all`, `group_vars/webservers` and `--extra-vars`. Which wins? Name 5 places to define variables.**

`--extra-vars` wins: it has the highest precedence. (Between the other two, `group_vars/webservers` beats `group_vars/all`, because the more specific group wins.)

5 places: role `defaults`, inventory / `group_vars` / `host_vars`, play `vars`, `vars_files`, registered vars / `set_fact`, and `-e` on the command line.

**32. What are `group_vars` and `host_vars`?**

Folders next to the inventory/playbook. `group_vars/<group>.yaml` applies to every host in that group, `host_vars/<host>.yaml` to one host. Loaded automatically by name.

**33. What are facts?**

System info Ansible gathers at the start of each play (OS, IP, memory), stored in `ansible_facts`. See them with `ansible <host> -m setup`; turn off with `gather_facts: false`.

**34. What does `register` do? (sometimes called the "registry")**

Saves a task's JSON result in a variable (`result.stdout`, `result.rc`) for later tasks. Ansible has no feature called "registry"; the term usually means `register`, `set_fact` or fact caching.

**35. Dot notation vs bracket notation?**

| | Example | Use when |
|---|---|---|
| Dot | `{{ user.name }}` | simple key names; easy to read |
| Bracket | `{{ user['first name'] }}`, `{{ user[key] }}` | keys with spaces, dashes or special characters, or a key stored in another variable |

**36. Why does `-e "name=My Shop"` give only `My`?**

`-e` splits on spaces. Use inner quotes: `-e "name='My Shop'"`.

**37. What is `inventory_hostname`?**

A magic variable: the current host's name as written in the inventory.

## 6. Roles

**38. What is a role, and why use it?**

A standard folder layout that packs everything for one job (e.g. "webserver") into a reusable unit.

| Benefit | Why |
|---|---|
| Modularity | one role = one job |
| Reusability | the same role in many playbooks/projects |
| Scalability | big setups stay organised |
| Collaboration | share in a team or on Ansible Galaxy |

**39. What is in a role directory?**

| File | Purpose |
|---|---|
| `tasks/main.yaml` | the main tasks |
| `handlers/main.yaml` | handlers (e.g. restart service) |
| `defaults/main.yaml` | default variables, **lowest** priority, meant to be overridden |
| `vars/main.yaml` | internal variables, **high** priority |
| `templates/` | Jinja2 `.j2` files |
| `files/` | static files to copy |
| `meta/main.yaml` | role info and dependencies on other roles |

Create one with `ansible-galaxy role init webserver`.

**40. Designing a webserver role: what goes in `defaults` vs `vars`?**

`defaults/`: things users should change (port, document root, server name). `vars/`: internal values users shouldn't touch (package names, config paths). `tasks/` installs and configures; `handlers/` restarts on config change; `templates/` holds the config file.

## 7. Vault and secrets

**41. How do you keep passwords and API keys out of plain text?**

Use **Ansible Vault**:

1. `ansible-vault create secrets.yaml` (or `encrypt` an existing file); set a vault password.
2. `ansible-vault edit secrets.yaml` to change it; it stays encrypted on disk and is safe in Git.
3. Load it in the playbook with `vars_files: [secrets.yaml]`.
4. Run with `--ask-vault-pass` or `--vault-password-file ~/.vault_pass`.
5. For many environments, use **vault IDs** (`--vault-id prod@prompt`) with different passwords for staging and prod.

**42. How do you encrypt one value instead of a whole file?**

```bash
ansible-vault encrypt_string 'mypassword' --name 'db_password'
```

It prints an encrypted block you paste into any vars file.

**43. How do you set a user's password with the `user` module?**

The `user` module needs a **hashed** password, not plain text. Make one with the `password_hash` filter:

```yaml
- name: Create user with a password
  ansible.builtin.user:
    name: devuser
    password: "{{ user_password | password_hash('sha512') }}"
```

Keep `user_password` itself in a vault file.

## 8. Advanced features

**44. How do you use a jump host (bastion) to reach private servers?**

Tell SSH to hop through the bastion. In the inventory:

```ini
[private_servers]
10.0.2.15
10.0.2.16

[private_servers:vars]
ansible_ssh_common_args='-o ProxyJump=ec2-user@bastion.example.com'
```

Or put `ProxyJump` in `~/.ssh/config` for those hosts. Ansible then connects control node → bastion → private server.

**45. How do you delegate a task to another host?**

`delegate_to:` runs one task on a different host than the play targets; `run_once: true` runs it only once.

```yaml
- name: Update DNS on the central server
  ansible.builtin.command: /usr/bin/update-dns
  delegate_to: dns-master
  run_once: true
```

Typical uses: load balancer changes, DNS updates, notifications.

**46. How do you loop over all hosts in a group?**

```yaml
- name: Ping each host in webservers
  hosts: localhost
  gather_facts: false
  tasks:
  - name: Ping
    ansible.builtin.ping:
    delegate_to: "{{ item }}"
    loop: "{{ groups['webservers'] }}"
```

`groups['webservers']` is the list of hosts in that group.

**47. Install packages on Debian and disable a service on RedHat, using loops and conditions.**

```yaml
- name: OS-aware setup
  hosts: all
  become: true
  tasks:
  - name: Install tools on Debian/Ubuntu
    ansible.builtin.apt:
      name: "{{ item }}"
      state: present
    loop: [vim, git, htop]
    when: ansible_facts['os_family'] == "Debian"

  - name: Disable cockpit on RedHat family
    ansible.builtin.service:
      name: cockpit.socket
      state: stopped
      enabled: false
    when: ansible_facts['os_family'] == "RedHat"
```

`when:` uses facts, so there are no hardcoded hostnames. (`apt` also accepts a list directly: `name: [vim, git, htop]`, which is faster than a loop.)

**48. What are callback plugins?**

Plugins that hook into a run to change the **output** or **send results somewhere**: JSON or YAML output, timing per task, logging, Slack/email notifications. Configure in `ansible.cfg`:

```ini
[defaults]
stdout_callback = yaml
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer
```

(`callbacks_enabled` replaced the old `callback_whitelist`.)

## 9. Debugging and performance

**49. A task is failing. How do you debug it?**

1. `-vvv` for detail (`-vvvv` shows SSH).
2. `--limit <host>` and `--start-at-task "<name>"` to repeat just that part.
3. `register` + `debug` to print variables and output.
4. `--check --diff` to preview, `--step` to go one task at a time.
5. `ansible <host> -m ping` / `-m setup` to check login and facts.

**50. A playbook is slow on 500 hosts. What do you tune first?**

| Tune | How |
|---|---|
| **Forks** (parallel hosts, default 5) | `forks = 50` in `ansible.cfg` or `-f 50` |
| **SSH pipelining** (fewer SSH round trips) | `pipelining = True` under `[ssh_connection]` |
| **Facts** | `gather_facts: false` when unused, or **fact caching** |

Also: `async` for long tasks, `strategy: free` so fast hosts don't wait for slow ones, and modules that take lists instead of loops.

## 10. Enterprise and CI/CD

**51. What are Ansible Tower, AWX and Ansible Automation Platform?**

A **web UI + API** on top of Ansible for teams. Tower was renamed into **Red Hat Ansible Automation Platform (AAP)**; **AWX** is its free open-source upstream.

| Feature | Why it matters |
|---|---|
| Web dashboard | run and watch jobs without a terminal |
| RBAC | who can run which playbook on which servers |
| Credentials store | SSH keys and passwords hidden from users |
| Scheduling | run playbooks on a timetable |
| Workflows | chain playbooks with success/failure paths |
| REST API | trigger from CI/CD or ITSM tools |
| Logging and audit | who ran what, when, with what result |

| | AWX | AAP |
|---|---|---|
| Cost | free, open source | paid subscription |
| Stability | fast-changing | hardened, tested |
| Support | community | Red Hat support + SLAs |
| Use | labs, testing | production |

**52. How is Ansible used in a CI/CD pipeline?**

The pipeline (Jenkins, GitHub Actions, GitLab CI) runs `ansible-playbook` after build and tests: provision or configure the servers, deploy the new version, restart services. The same playbooks run in every environment, so dev, test and prod stay consistent.

## 11. Scenario questions

**53. Install different packages depending on the OS.**

Use facts with `when:`, or one task with the generic `package` module and per-OS variables:

```yaml
- name: Install packages based on OS
  hosts: all
  become: true
  tasks:
  - name: Debian/Ubuntu packages
    ansible.builtin.apt:
      name: [apache2, libapache2-mod-php]
      state: present
    when: ansible_facts['os_family'] == "Debian"

  - name: RedHat-family packages
    ansible.builtin.dnf:
      name: [httpd, php]
      state: present
    when: ansible_facts['os_family'] == "RedHat"

  - name: Common packages
    ansible.builtin.package:
      name: [git, curl]
      state: present
```

**54. How would you build a LAMP stack and deploy a web page?**

One playbook (or one role per layer): install **Apache** and start it, install **MySQL/MariaDB** and create the database and user, install **PHP** and its modules, `copy` or `template` the web page to `/var/www/html`, open port 80, and enable all services on boot. Use a handler to restart Apache when its config changes.

**55. Zero-downtime rolling update behind a load balancer: how?**

1. One orchestration playbook: database migrations → API tier → web tier.
2. `serial: 1` (or `serial: 25%`) to update a few servers at a time.
3. Before each server: take it out of the load balancer (`uri` module calling the LB API, with `delegate_to`).
4. Deploy the new version (a role).
5. Health check: `uri` on `/health` or `wait_for` on the port.
6. Put the server back in the load balancer, then move to the next.
7. Wrap it in `block` / `rescue`: on failure, roll back, alert, and restore the server to the pool. `max_fail_percentage` stops the rollout if too many fail.
