# Ansible — Deep-Dive Interview Q&A

## Table of Contents
1. [Architecture & Core Concepts](#architecture--core-concepts)
2. [Playbooks & Roles](#playbooks--roles)
3. [Variables & Templating](#variables--templating)
4. [Advanced Patterns](#advanced-patterns)
5. [Troubleshooting & Tricky Scenarios](#troubleshooting--tricky-scenarios)

---

## Architecture & Core Concepts


**Q1: Explain Ansible architecture. How does it differ from Puppet/Chef?**

**A:**

```
Control Node (your machine)
    ↓ SSH/WinRM (agentless!)
Managed Nodes (servers)
```

| Feature | Ansible | Puppet/Chef |
|---------|---------|-------------|
| Architecture | Agentless (push) | Agent-based (pull) |
| Communication | SSH/WinRM | Custom agent + server |
| Language | YAML (declarative) | Ruby DSL |
| Execution | Sequential (top-to-bottom) | Catalog-based (order by dependencies) |
| Idempotency | Module-level | Built into resource model |
| Learning curve | Low | High |
| State | Stateless | Stores state on server |

**Tricky**: Ansible is "mostly push" but can work in pull mode with `ansible-pull` (cron job pulls playbook from Git and applies locally). Useful for auto-scaling instances that self-configure on boot.

---

**Q2: Explain Ansible inventory — static vs dynamic. How does dynamic inventory work with AWS?**

**A:**

**Static inventory (hosts file):**
```ini
[webservers]
web1.example.com ansible_host=10.0.1.10
web2.example.com ansible_host=10.0.1.11

[dbservers]
db1.example.com ansible_host=10.0.2.10

[production:children]
webservers
dbservers

[webservers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/prod.pem
```

**Dynamic inventory (AWS plugin):**
```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type
  - key: placement.availability_zone
filters:
  instance-state-name: running
  "tag:ManagedBy": ansible
compose:
  ansible_host: private_ip_address
```

```bash
# Test dynamic inventory
ansible-inventory -i aws_ec2.yml --graph
# Output: @env_production → instances tagged Environment=production
```

**Tricky**: Dynamic inventory is re-queried on every run. If an instance is terminated between gathering inventory and executing tasks, you get connection failures. Use `ignore_unreachable: true` for fault tolerance.

---

**Q3: What is the difference between `ansible` ad-hoc commands and `ansible-playbook`?**

**A:**

```bash
# Ad-hoc: One-off commands (quick tasks)
ansible webservers -m ping
ansible all -m shell -a "df -h"
ansible webservers -m apt -a "name=nginx state=present" --become
ansible dbservers -m service -a "name=mysql state=restarted" --become

# Playbook: Repeatable, versioned automation
ansible-playbook deploy.yml -i inventory/production --limit webservers
```

**Key ad-hoc flags:**
```bash
-m module_name        # Module to use
-a "arguments"        # Module arguments
--become              # Sudo/privilege escalation
-b -K                 # Become + ask sudo password
-f 10                 # Forks (parallelism)
--limit "host1"       # Run only on specific hosts
-e "var=value"        # Extra variables
--check               # Dry run
--diff                # Show file changes
```

---

## Playbooks & Roles

**Q4: Explain Ansible role structure. What is each directory for?**

**A:**

```
roles/webserver/
├── tasks/
│   └── main.yml        # Main task list (auto-included)
├── handlers/
│   └── main.yml        # Handlers (triggered by notify)
├── templates/
│   └── nginx.conf.j2   # Jinja2 templates
├── files/
│   └── index.html      # Static files (copied as-is)
├── vars/
│   └── main.yml        # High-priority variables (rarely overridden)
├── defaults/
│   └── main.yml        # Default variables (easily overridden)
├── meta/
│   └── main.yml        # Role dependencies, metadata
├── tests/
│   └── test.yml        # Test playbook
└── README.md
```

**Key differences:**
- `defaults/` → Lowest priority (user SHOULD override these)
- `vars/` → High priority (internal to role, user shouldn't change)
- `files/` → Copied with `copy` module (no template processing)
- `templates/` → Processed with `template` module (Jinja2 rendering)
- `handlers/` → Only run when notified AND when task changed something

**Tricky**: Handlers run at the END of the play, not immediately after the notifying task. Use `meta: flush_handlers` to force immediate execution if needed.

---

**Q5: Explain handler execution. What happens if multiple tasks notify the same handler?**

**A:**

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: restart nginx

  - name: Update SSL cert
    copy:
      src: cert.pem
      dest: /etc/nginx/ssl/cert.pem
    notify: restart nginx

handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted
```

**Key behaviors:**
1. Handler runs only ONCE even if notified multiple times
2. Handlers run at END of play (after all tasks)
3. Handlers only run if at least one notifying task CHANGED something
4. If a task fails BEFORE handlers run, handlers are SKIPPED (use `--force-handlers` to override)
5. Handler order follows handler definition order, NOT notification order

**Tricky**: If task A notifies handler X, then task B fails, handler X NEVER runs (unless `--force-handlers`). This can leave config updated but service not restarted = broken state!

**Fix with blocks:**
```yaml
- block:
    - name: Update config
      template: ...
      notify: restart nginx
    - name: Other task that might fail
      command: ...
  rescue:
    - name: Rollback config
      copy: ...
  always:
    - meta: flush_handlers
```

---

**Q6: What is the difference between `include_*` and `import_*` in Ansible?**

**A:**

| Feature | `import_*` (static) | `include_*` (dynamic) |
|---------|---------------------|----------------------|
| Processing time | Playbook parse time | Runtime |
| Conditionals | Applied to EACH task inside | Applied to include itself |
| Loops | Cannot loop | Can loop |
| Tags | Inherited by inner tasks | NOT inherited |
| `--list-tasks` | Shows inner tasks | Shows include only |
| Handler notify | Can notify inner handlers | Cannot notify by name |

```yaml
# STATIC - processed at parse time
- import_tasks: setup.yml
  when: ansible_os_family == "Debian"
  # "when" applied to EVERY task in setup.yml individually

# DYNAMIC - processed at runtime
- include_tasks: "{{ ansible_os_family }}.yml"
  # Variable in filename (impossible with import!)

# Dynamic with loop
- include_tasks: deploy_app.yml
  loop: "{{ applications }}"
  loop_control:
    loop_var: app
```

**Tricky**: With `import_tasks` + `when`, the condition is evaluated for each inner task separately. With `include_tasks` + `when`, the condition is evaluated ONCE for the include statement. If the condition changes during execution (rare), behavior differs!

---

## Variables & Templating

**Q7: Explain Ansible variable precedence (from lowest to highest).**

**A:** (22 levels, key ones shown)

```
Lowest priority:
 1. role defaults (defaults/main.yml)
 2. inventory file group_vars
 3. inventory group_vars/all
 4. inventory group_vars/specific_group
 5. inventory host_vars
 6. host facts (gathered)
 7. play vars
 8. play vars_prompt
 9. play vars_files
10. role vars (vars/main.yml)
11. block vars
12. task vars (set in task)
13. include_vars
14. set_facts / registered vars
15. role parameters
16. include parameters
17. extra vars (-e) ← ALWAYS WIN
Highest priority
```

**Critical rule**: `-e` (extra vars) ALWAYS override everything. Use for emergency overrides.

**Tricky scenario:**
```yaml
# group_vars/all.yml
http_port: 80

# group_vars/production.yml
http_port: 443

# host_vars/web1.yml
http_port: 8080

# Result for web1 in production group: http_port = 8080
# host_vars > group_vars
```

---

**Q8: Explain Jinja2 templating in Ansible. What are filters and tests?**

**A:**

```yaml
# Common patterns:
- name: Template nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf

# nginx.conf.j2:
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_fqdn }};

    {% for backend in app_servers %}
    upstream backend_{{ loop.index }} {
        server {{ backend }}:{{ app_port }};
    }
    {% endfor %}

    {% if ssl_enabled | bool %}
    ssl_certificate {{ ssl_cert_path }};
    {% endif %}
}
```

**Useful filters:**
```yaml
{{ list | join(',') }}              # Join list
{{ string | hash('sha256') }}       # Hash
{{ path | basename }}               # Filename from path
{{ dict | dict2items }}             # Dict to list of {key, value}
{{ password | password_hash('sha512') }}  # Password hash
{{ ip_list | ipaddr('private') }}   # Filter private IPs
{{ variable | default(omit) }}      # Omit parameter entirely if undefined
{{ items | selectattr('state', 'eq', 'active') | list }}  # Filter objects
{{ output.stdout | from_json }}     # Parse JSON string
```

**Tricky**: `{{ variable }}` in `when` conditions must NOT have curly braces:
```yaml
# WRONG
when: "{{ my_var }} == true"

# CORRECT
when: my_var == true
# Or: when: my_var | bool
```

---

## Advanced Patterns

**Q9: Explain Ansible Vault. How do you manage secrets in a team?**

**A:**

```bash
# Encrypt a file
ansible-vault encrypt secrets.yml

# Decrypt for editing
ansible-vault edit secrets.yml

# Encrypt a string (inline)
ansible-vault encrypt_string 'SuperSecret123' --name 'db_password'
# Output (put in vars file):
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   623564613264...

# Run playbook with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file=~/.vault_pass

# Multiple vault IDs (different passwords for different secrets)
ansible-playbook site.yml \
  --vault-id dev@~/.vault_pass_dev \
  --vault-id prod@~/.vault_pass_prod
```

**Team best practices:**
1. Use vault ID labels for different environments
2. Store vault password in CI/CD secrets (never in Git)
3. Encrypt only sensitive files, not entire playbooks
4. Use `no_log: true` on tasks that handle secrets
5. Consider external lookup: `lookup('aws_ssm', '/prod/db_password')`

---

**Q10: How do you make Ansible playbooks idempotent? What modules are NOT idempotent?**

**A:**

**Non-idempotent modules (use carefully):**
- `command` / `shell` — Always reports "changed"
- `raw` — No change detection
- `script` — Runs every time

**Making command/shell idempotent:**
```yaml
# BAD - runs every time, always "changed"
- command: /opt/scripts/setup.sh

# GOOD - only runs if condition met
- command: /opt/scripts/setup.sh
  args:
    creates: /opt/app/.initialized  # Skip if file exists

- shell: echo "export PATH=/opt/bin:$PATH" >> /etc/profile
  args:
    creates: /etc/profile.d/custom.sh  # Wrong! Use lineinfile instead

# BEST - use purpose-built modules
- lineinfile:
    path: /etc/profile
    line: 'export PATH=/opt/bin:$PATH'
    state: present

# Use 'changed_when' to control change reporting
- command: /opt/check_status.sh
  register: status_result
  changed_when: "'NEEDS_UPDATE' in status_result.stdout"
```

---

**Q11: Explain Ansible delegation, `delegate_to`, and `local_action`.**

**A:**

```yaml
# Run task on a DIFFERENT host than the play target
- name: Remove server from load balancer
  community.general.haproxy:
    state: disabled
    host: "{{ inventory_hostname }}"
  delegate_to: lb-server

- name: Wait for server to drain
  wait_for:
    host: "{{ inventory_hostname }}"
    port: 80
    state: stopped
  delegate_to: localhost

# Run on control node (equivalent to delegate_to: localhost)
- name: Send Slack notification
  local_action:
    module: slack
    token: "{{ slack_token }}"
    msg: "Deployment complete on {{ inventory_hostname }}"

# Delegate + serial (rolling deployment)
- hosts: webservers
  serial: 1  # One server at a time
  tasks:
    - name: Remove from LB
      haproxy: state=disabled host={{ inventory_hostname }}
      delegate_to: "{{ lb_host }}"

    - name: Deploy application
      # ... deployment tasks ...

    - name: Add back to LB
      haproxy: state=enabled host={{ inventory_hostname }}
      delegate_to: "{{ lb_host }}"
```

**Tricky**: `delegate_to` runs the task on the delegate host but variables still reference the ORIGINAL inventory host. `hostvars[inventory_hostname]` = original host vars. Use `delegate_facts: true` if you want gathered facts stored for the delegate.

---

**Q12: How do you handle errors and implement retry logic in Ansible?**

**A:**

```yaml
# Block/rescue/always (try/catch/finally)
- block:
    - name: Attempt deployment
      command: /opt/deploy.sh
    - name: Run health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
  rescue:
    - name: Rollback on failure
      command: /opt/rollback.sh
    - name: Alert team
      slack:
        msg: "Deployment FAILED on {{ inventory_hostname }}"
  always:
    - name: Clean temp files
      file:
        path: /tmp/deploy_artifacts
        state: absent

# Retry logic
- name: Wait for service to be ready
  uri:
    url: "http://localhost:8080/health"
  register: result
  until: result.status == 200
  retries: 30
  delay: 10  # Wait 10s between retries (total: 5 min max)

# Ignore errors (continue on failure)
- name: Optional cleanup
  command: /opt/cleanup.sh
  ignore_errors: true

# Fail with custom message
- name: Validate config
  command: nginx -t
  register: nginx_check
  failed_when: "'error' in nginx_check.stderr"
```

---

**Q13: Explain `serial`, `max_fail_percentage`, and rolling update strategies.**

**A:**

```yaml
# Rolling update: 2 hosts at a time
- hosts: webservers  # 10 hosts total
  serial: 2
  # Runs on hosts 1-2, then 3-4, then 5-6, etc.

# Percentage-based
- hosts: webservers
  serial: "25%"  # 25% of hosts at a time

# Escalating batch size
- hosts: webservers
  serial:
    - 1       # First: 1 host (canary)
    - 5       # Then: 5 hosts
    - "100%"  # Finally: all remaining

# Fail-fast: Stop if too many hosts fail
- hosts: webservers
  serial: 5
  max_fail_percentage: 30  # Stop if 30%+ of batch fails

# Throttle forks (parallel connections)
# ansible.cfg: forks = 20 (default 5)
# Or: ansible-playbook site.yml -f 20
```

**Tricky**: Without `serial`, Ansible runs tasks on ALL hosts simultaneously (limited by forks). A bug in your playbook could break ALL servers at once. Always use `serial` for production deployments!

---

## Troubleshooting & Tricky Scenarios

**Q14: Ansible playbook works for one host but fails with "Permission denied" on another. Both use the same key. What's wrong?**

**A:**

**Common causes:**
1. **Different SSH user**: Host uses different default user (ubuntu vs ec2-user vs root)
2. **SSH key not in authorized_keys**: Key added to some hosts but not others
3. **Become/sudo issue**: User can SSH but `become: true` fails (no sudoers entry)
4. **Host key changed**: Known_hosts has old key → connection rejected
5. **SELinux blocking**: SSH works, but Ansible module operations fail
6. **Different SSH port**: Host uses non-standard port

**Debugging:**
```bash
# Verbose mode (shows SSH commands)
ansible-playbook site.yml -vvvv

# Test connection
ansible problematic_host -m ping -vvvv

# Check SSH manually
ssh -i ~/.ssh/key.pem -o StrictHostKeyChecking=no user@host

# Check become
ansible problematic_host -m command -a "whoami" --become -vvv
```

---

**Q15: How do you test Ansible roles? Explain Molecule.**

**A:**

```bash
# Initialize molecule scenario
molecule init scenario --driver-name docker

# molecule/default/molecule.yml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-test
    image: ubuntu:22.04
    pre_build_image: true
  - name: centos-test
    image: centos:8
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible  # or testinfra

# molecule/default/verify.yml
- hosts: all
  tasks:
    - name: Verify nginx is installed
      package:
        name: nginx
        state: present
      check_mode: true
      register: nginx_check
      failed_when: nginx_check.changed

    - name: Verify nginx is running
      service:
        name: nginx
        state: started
      check_mode: true
      register: service_check
      failed_when: service_check.changed
```

```bash
# Full test lifecycle
molecule create    # Create test containers
molecule converge  # Run role
molecule verify    # Run verification tests
molecule destroy   # Clean up

# All-in-one
molecule test
```

---

**Q16: Explain Ansible callback plugins and custom module development.**

**A:**

**Callback plugins** (customize output/logging):
```ini
# ansible.cfg
[defaults]
stdout_callback = yaml    # Pretty YAML output instead of JSON
callbacks_enabled = timer, profile_tasks, profile_roles
# timer: Total execution time
# profile_tasks: Time per task
# profile_roles: Time per role
```

**Custom module (basic):**
```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(
            name=dict(required=True, type='str'),
            state=dict(default='present', choices=['present', 'absent']),
        ),
        supports_check_mode=True
    )

    name = module.params['name']
    state = module.params['state']

    # Check mode - report what would happen
    if module.check_mode:
        module.exit_json(changed=True, msg=f"Would ensure {name} is {state}")

    # Actual logic here
    changed = do_something(name, state)

    module.exit_json(changed=changed, msg=f"{name} is now {state}")

if __name__ == '__main__':
    main()
```

Place in `library/` directory of your playbook or role.

---

**Q17: How do you optimize Ansible performance for large inventories (1000+ hosts)?**

**A:**

```ini
# ansible.cfg optimizations
[defaults]
forks = 50                          # Parallel connections (default 5!)
gathering = smart                   # Cache facts, don't re-gather
fact_caching = jsonfile             # Persist facts to disk
fact_caching_connection = /tmp/facts_cache
fact_caching_timeout = 86400       # 24 hour cache

[ssh_connection]
pipelining = True                  # Reduce SSH operations (BIG speedup)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s  # SSH multiplexing
```

**Playbook-level optimizations:**
```yaml
- hosts: all
  gather_facts: false          # Skip if not needed
  strategy: free               # Don't wait for slowest host per task

# Or mitogen strategy (10-70x faster)
# strategy: mitogen_linear

# Async for long-running tasks
- name: Long running update
  apt:
    upgrade: dist
  async: 3600    # Max runtime seconds
  poll: 0        # Fire and forget

- name: Check on update
  async_status:
    jid: "{{ update_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 60
```

**Tricky**: `pipelining = True` requires `requiretty` disabled in sudoers on target hosts. Without it, each task is: open SSH → upload module → execute → download result → close SSH. With pipelining: reuse single SSH connection for everything.

---

**Q18: What is `ansible_facts` vs `set_fact` vs `register`? When do you use each?**

**A:**

```yaml
# ansible_facts - automatically gathered host information
- debug:
    msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    # "OS: Ubuntu 22.04"

# register - capture task output
- command: cat /etc/app/version
  register: app_version
- debug:
    msg: "App version: {{ app_version.stdout }}"

# set_fact - create new variable (persists for host across plays)
- set_fact:
    full_app_name: "{{ app_name }}-{{ app_version.stdout }}"
    cacheable: true  # Persist to fact cache
```

| Feature | ansible_facts | register | set_fact |
|---------|---------------|----------|----------|
| Source | Auto-gathered from host | Task output | You define |
| Scope | Host (per play by default) | Host + play | Host (all plays if cacheable) |
| Persistence | Per run (or cached) | Current play | Current run (or cached) |
| Available in | Templates, tasks | Tasks after registration | Tasks after setting |

**Tricky**: `register` variables include `changed`, `failed`, `stdout`, `stderr`, `rc` (for commands). Check `result.rc == 0` not just `result.stdout`.

---

**Q19: How do you manage Windows hosts with Ansible?**

**A:**

```ini
# inventory
[windows]
win-server1 ansible_host=10.0.1.50

[windows:vars]
ansible_user=Administrator
ansible_password={{ vault_win_password }}
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5986  # HTTPS WinRM
```

**Windows-specific modules:**
```yaml
- name: Install IIS
  win_feature:
    name: Web-Server
    state: present

- name: Copy file
  win_copy:
    src: app.zip
    dest: C:\temp\app.zip

- name: Run PowerShell
  win_shell: |
    Get-Service | Where-Object {$_.Status -eq 'Running'}
  register: services

- name: Install MSI
  win_package:
    path: C:\temp\installer.msi
    state: present

- name: Manage Windows service
  win_service:
    name: MyApp
    state: started
    start_mode: auto
```

**Tricky**: WinRM must be configured on Windows hosts BEFORE Ansible can connect. Use `ConfigureRemotingForAnsible.ps1` script. Also: Windows modules start with `win_*` — using Linux modules (like `copy` instead of `win_copy`) will fail!

---

**Q20: Explain Ansible Galaxy and collections. How do you manage dependencies?**

**A:**

```bash
# Install role from Galaxy
ansible-galaxy role install geerlingguy.docker

# Install collection
ansible-galaxy collection install amazon.aws

# Requirements file (for CI/CD)
# requirements.yml
---
roles:
  - name: geerlingguy.docker
    version: "6.1.0"
  - name: geerlingguy.nginx

collections:
  - name: amazon.aws
    version: ">=6.0.0,<7.0.0"
  - name: community.general
    version: "7.0.0"
  - name: kubernetes.core

# Install all dependencies
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

**Collections structure:**
```
namespace.collection_name
├── plugins/
│   ├── modules/       # Custom modules
│   ├── inventory/     # Inventory plugins
│   ├── callback/      # Callback plugins
│   └── lookup/        # Lookup plugins
├── roles/             # Bundled roles
├── playbooks/         # Bundled playbooks
└── galaxy.yml         # Metadata
```

**Tricky**: Since Ansible 2.10, most modules moved from "built-in" to collections. A playbook that worked on 2.9 may fail on newer versions without installing the correct collection. Always pin collection versions in `requirements.yml`!



---

## Additional Scenario-Based Tricky Questions

---

**Q21: Your Ansible playbook runs successfully on 95% of servers but fails on 5% with "unreachable" errors. All servers are in the same subnet and you can SSH to them manually. What's happening?**

**A:**

**Diagnosis:**
```bash
# Run with verbose to see SSH details
ansible all -m ping -vvvv 2>&1 | grep -A5 "UNREACHABLE"

# Common output:
# "Failed to connect to the host via ssh: Connection timed out"
# "Failed to connect to the host via ssh: Permission denied (publickey)"
# "Failed to connect to the host via ssh: No route to host"
```

**Causes and fixes:**
```
1. SSH CONNECTION LIMIT on target hosts:
   - /etc/ssh/sshd_config: MaxSessions=10, MaxStartups=10:30:60
   - Ansible opens many parallel connections (default forks=5)
   - With 100 servers and forks=20 → 20 simultaneous SSH connections
   - Some servers already have 8 sessions → MaxSessions exceeded
   - Fix: Increase MaxStartups or reduce Ansible forks

2. DNS RESOLUTION TIMEOUT:
   - Ansible resolves hostnames on each connection
   - 5% of servers have reverse DNS entries pointing to stale records
   - SSH does reverse lookup → times out
   - Fix: ansible.cfg: [ssh_connection] ssh_args = -o UseDNS=no

3. HOST KEY CHANGED (after server rebuild):
   - Server was rebuilt but kept same IP
   - known_hosts has old host key → SSH refuses to connect
   - Fix: ansible.cfg: host_key_checking = False (for dynamic infra)
   - Better: StrictHostKeyChecking=accept-new (accept first time only)

4. CONTROL PERSIST SOCKET STALE:
   - ControlMaster sockets from previous run in /tmp
   - Stale socket prevents new connection
   - Fix: ssh_args = -o ControlPersist=60s -o ControlPath=/tmp/ansible-%r@%h:%p

5. SELINUX / FIREWALL INTERMITTENT:
   - firewalld or iptables rate limiting SSH connections
   - 5% fail because they hit the rate limit threshold
   - Fix: Check conntrack limits and firewall rules on failing hosts
```

**Ansible SSH optimization:**
```ini
# ansible.cfg
[ssh_connection]
pipelining = True              # Reduces SSH operations (HUGE speedup)
ssh_args = -o ControlMaster=auto -o ControlPersist=300s -o PreferredAuthentications=publickey
retries = 3                    # Retry unreachable hosts
timeout = 30                   # Connection timeout

[defaults]
forks = 20                     # Parallel connections (balance speed vs target load)
gathering = smart              # Cache facts, don't regather
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 3600
```

**Tricky**: `pipelining = True` is the single biggest Ansible performance win (2-5x faster) but it ONLY works if `requiretty` is disabled in `/etc/sudoers` on the target hosts. If you enable pipelining and some hosts have `requiretty`, those hosts will fail with cryptic errors. Check: `grep requiretty /etc/sudoers` on failing hosts. Fix: `Defaults !requiretty` in sudoers.

---

**Q22: You're managing 500 servers with Ansible. A playbook change needs to go to all servers but you can't afford downtime if it fails. How do you safely roll out changes with Ansible's serial execution?**

**A:**

**Progressive rollout strategy:**
```yaml
# Canary → Small batch → Full rollout with automatic rollback

- hosts: webservers
  serial:
    - 1           # First: single canary server
    - 5           # Then: small batch (5 servers)
    - "10%"       # Then: 10% at a time
    - "25%"       # Then: 25% at a time
    - "100%"      # Finally: all remaining
  max_fail_percentage: 5   # Stop if >5% of batch fails

  pre_tasks:
    - name: Health check before changes
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      register: pre_health

  tasks:
    - name: Deploy new configuration
      template:
        src: app.conf.j2
        dest: /etc/app/app.conf
      notify: restart_app

    - name: Wait for service to be healthy after change
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      retries: 10
      delay: 5
      register: post_health

  post_tasks:
    - name: Validate application functionality
      uri:
        url: "http://{{ inventory_hostname }}:8080/api/v1/status"
        status_code: 200
        return_content: yes
      register: api_response
      failed_when: "'healthy' not in api_response.content"

    - name: Notify monitoring of deployment progress
      uri:
        url: "https://slack.webhook.url"
        method: POST
        body_format: json
        body:
          text: "Deployed to {{ ansible_play_batch | length }} servers. Remaining: {{ ansible_play_hosts_all | length - ansible_play_batch | length }}"

  handlers:
    - name: restart_app
      service:
        name: myapp
        state: restarted
```

**Automatic rollback on failure:**
```yaml
  rescue:
    - name: Rollback — restore previous config
      copy:
        src: /etc/app/app.conf.bak
        dest: /etc/app/app.conf
        remote_src: yes

    - name: Restart with old config
      service:
        name: myapp
        state: restarted

    - name: Alert team of rollback
      uri:
        url: "https://pagerduty.webhook.url"
        method: POST
        body_format: json
        body:
          routing_key: "abc123"
          event_action: "trigger"
          payload:
            summary: "Ansible rollback triggered on {{ inventory_hostname }}"
            severity: "critical"
```

**Tricky**: `max_fail_percentage` applies PER BATCH, not total. With `serial: 50` and `max_fail_percentage: 10` on 500 servers: if 5 out of 50 fail in the first batch (10%), Ansible STOPS. But if only 4 fail per batch (8%), it continues — potentially leaving 40+ servers in a broken state across 10 batches. For critical infrastructure, use `serial: 1` for the first few, then increase. Also, `any_errors_fatal: true` stops the ENTIRE play on the first failure — nuclear option but safest.

---

**Q23: Your Ansible vault password is stored in a file on the control node. A security audit found that 15 engineers have access to this file. How do you implement proper secrets management for Ansible in production?**

**A:**

**Current (insecure) setup:**
```bash
# Password stored in plaintext file readable by team
cat ~/.vault_password  # "SuperSecretP@ss!"
ansible-playbook site.yml --vault-password-file ~/.vault_password
# Problem: Anyone on the control node can decrypt ALL vault secrets
```

**Production-grade secrets management:**

```bash
# Option 1: Vault password from external secrets manager (recommended)
# ansible.cfg:
[defaults]
vault_password_file = /scripts/get-vault-password.sh

# /scripts/get-vault-password.sh:
#!/bin/bash
# Fetch from AWS Secrets Manager (requires IAM role, not stored locally)
aws secretsmanager get-secret-value \
  --secret-id ansible-vault-password \
  --query SecretString --output text

# Option 2: HashiCorp Vault integration (best for enterprises)
# Use ansible-vault with lookup plugin:
# In playbook (no vault-encrypted files needed):
- name: Get database password
  set_fact:
    db_password: "{{ lookup('hashi_vault', 'secret/data/production/db:password') }}"

# Option 3: Multiple vault IDs (different passwords per environment)
# Encrypt prod secrets with prod password, dev with dev password
ansible-vault encrypt_string --vault-id prod@prompt 'realpassword' --name 'db_pass'
ansible-vault encrypt_string --vault-id dev@prompt 'devpassword' --name 'db_pass'

# Run with correct vault ID:
ansible-playbook site.yml --vault-id prod@/scripts/get-prod-password.sh --vault-id dev@/scripts/get-dev-password.sh
```

**Best practice — don't use Ansible Vault for secrets at all:**
```yaml
# Instead of encrypting secrets IN playbooks/vars files,
# fetch them at runtime from external secret store:

- name: Fetch secrets from AWS Secrets Manager
  set_fact:
    app_secrets: "{{ lookup('aws_ssm', '/production/myapp/', bypath=true) }}"
  no_log: true  # CRITICAL: Don't log secret values!

- name: Template config with secrets
  template:
    src: app.conf.j2
    dest: /etc/app/app.conf
    mode: '0600'
  no_log: true
```

**Tricky**: `no_log: true` is essential for any task that handles secrets. Without it, Ansible prints secret values in stdout and logs. But `no_log: true` also hides ERROR messages, making debugging impossible. The compromise: use `no_log: "{{ not ansible_check_mode }}"` — secrets are hidden in real runs but visible in check mode (which doesn't actually execute anything). Also, Ansible facts are cached — if you `set_fact` a secret, it may persist in the fact cache on disk. Always set `fact_caching_timeout` low or exclude secret facts from caching.

---
