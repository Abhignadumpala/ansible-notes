# Ansible Interview Questions

[← Back to all topics](README.md)

## ⭐ Very Important

**1. If the same variable name is defined in many places, which value does Ansible use?**

The one with the highest precedence. Extra vars (`-e`) always win, then task vars, `vars_files`, `vars_prompt`, play vars, and inventory vars are lowest of these. More specific beats more general (task > play, host > group).

## Other Questions

### Introduction

**1. What is Ansible?**

An open-source automation tool for configuration management, application deployment, provisioning and orchestration. It's agentless (SSH/WinRM), and you describe the work in YAML playbooks.

**2. What does "agentless" mean?**

Nothing is installed on the managed nodes. Ansible logs in over SSH, copies the module, runs it with Python and deletes it.

**3. What do managed nodes need?**

SSH (port 22 open from the control node), Python, and a user Ansible can log in with. Windows nodes need WinRM and PowerShell instead.

**4. Can Windows be the control node?**

No. The control node must be Linux or macOS. Windows can only be a managed node.

**5. What is idempotency?**

Running the same playbook many times gives the same result. Modules check the current state and change only what's needed: the second run reports `ok`, `changed=0`.

**6. What is the difference between declarative and imperative?**

Declarative = you describe the end state ("nginx present"); imperative = you list the commands (`dnf install nginx`). Ansible modules are declarative.

**7. Explain Ansible's workflow.**

Read inventory → read playbook → SSH to each host → gather facts → copy and run each module → collect the JSON result → delete the module → print PLAY RECAP.

### Inventory

**8. What is an inventory and where is the default one?**

The list of hosts and groups Ansible manages. Default `/etc/ansible/hosts`; override with `-i` or `inventory =` in `ansible.cfg`.

**9. Static vs dynamic inventory?**

Static = a fixed list written by hand (INI/YAML), for labs and stable setups. Dynamic = fetched at run time from a source like AWS (`amazon.aws.aws_ec2` plugin), for production where servers come and go.

**10. Host vs group?**

A host is one machine; a group is a name for many hosts. `[name:children]` makes a group of groups. `all` and `ungrouped` always exist.

**11. What is `ansible_host` used for?**

The real IP/hostname to connect to, when the inventory name is just a short alias (`web1 ansible_host=10.0.0.5`).

**12. How do you check an inventory?**

`ansible-inventory -i inv --graph` (tree), `--list` (JSON), `--host web1` (one host's vars). Only `ansible <group> -m ping` proves Ansible can actually log in.

### Modules

**13. What is a module?**

A small standalone program that does one job on a managed node (`user`, `package`, `copy`, `service`). Every module returns JSON with `changed` / `failed`.

**14. `command` vs `shell` module?**

`command` runs a command directly (no pipes, redirects or variables like `$HOME`). `shell` runs it through `/bin/sh`, so `|`, `>`, `&&` work. Both always report `changed`, so prefer real modules.

**15. `package` vs `dnf` / `apt`?**

`package` is generic: it detects the OS and uses the right package manager, so one playbook works on Amazon Linux and Ubuntu. `dnf` / `apt` are OS-specific with more options.

**16. What is an ad-hoc command?**

Running one module from the command line without a playbook: `ansible web -b -m service -a "name=nginx state=started"`. Good for quick checks.

**17. What is FQCN?**

Fully Qualified Collection Name, e.g. `ansible.builtin.copy` instead of `copy`. It names the collection, which avoids clashes.

**18. How do you see a module's options?**

`ansible-doc <module>` (full) or `ansible-doc -s <module>` (short).

### Playbooks

**19. What is the structure of a playbook?**

A list of plays. Each play has `name`, `hosts`, optional `become` / `vars`, and `tasks`. Each task has a `name` and runs one module.

**20. What does `become: yes` do?**

Runs tasks as root (sudo). Needed for installing packages, managing services and users.

**21. What is a handler and when does it run?**

A task triggered by `notify:` from another task. It runs only if that task reported `changed`, once, at the end of the play. Used to restart a service after a config change.

**22. What is a role?**

A standard folder layout (`tasks/`, `handlers/`, `vars/`, `defaults/`, `templates/`, `files/`) that groups everything for one job so it can be reused across playbooks.

**23. How do you test a playbook without changing servers?**

`--syntax-check` to validate YAML; `--check` for a dry run that shows what would change.

**24. Why can a playbook have many plays?**

Each play can target different hosts, so one run can set up the database, then the backend, then the frontend in the right order (orchestration).

### Variables

**25. Where can variables be defined?**

Play `vars`, task `vars`, `vars_files`, `vars_prompt`, inventory, `group_vars/` and `host_vars/`, `-e`, `set_fact`, `register`, and facts.

**26. What are `group_vars` and `host_vars`?**

Folders next to the inventory/playbook. `group_vars/<group>.yaml` applies to every host in that group, `host_vars/<host>.yaml` to one host. Loaded automatically by name.

**27. What are facts?**

System info Ansible gathers at the start of each play (OS, IP, memory), stored in `ansible_facts`. See them with `ansible <host> -m setup`; turn off with `gather_facts: false`.

**28. What does `register` do?**

Saves a task's JSON result in a variable (e.g. `result.stdout`, `result.rc`) for later tasks.

**29. Why does `-e "name=My Shop"` give only `My`?**

`-e` splits on spaces. Use inner quotes: `-e "name='My Shop'"`.

**30. What is `inventory_hostname`?**

A magic variable: the current host's name as written in the inventory.
