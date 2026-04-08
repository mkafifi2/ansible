## Section 1: Ansible Basics — Inventory, Configuration, and Setup

### Q1. What is an Ansible inventory file, and how do you define host groups?

An inventory file lists the managed nodes Ansible will work with. You define groups using INI-style brackets. Example:

```ini
[web]
192.168.1.10
192.168.1.11

[db]
192.168.1.20
192.168.1.21
```

Each group (`web`, `db`) contains servers that share a role. You reference these groups in playbooks with `hosts: web` or `hosts: db`.

---

### Q2. How do you define group-specific SSH users in a static inventory file?

Use group variables in the inventory to set `ansible_user` per group:

```ini
[web]
192.168.1.10
192.168.1.11

[web:vars]
ansible_user=webadmin

[db]
192.168.1.20
192.168.1.21

[db:vars]
ansible_user=dbadmin
```

This ensures Ansible connects with the correct SSH user for each group.

---

### Q3. How do you define custom SSH key paths for each host or group in a static inventory?

Set `ansible_ssh_private_key_file` in group or host variables:

```ini
[web:vars]
ansible_ssh_private_key_file=/home/admin/.ssh/web_key

[db:vars]
ansible_ssh_private_key_file=/home/admin/.ssh/db_key
```

You can also set it per host:

```ini
[web]
192.168.1.10 ansible_ssh_private_key_file=/home/admin/.ssh/host10_key
```

---

### Q4. How do you define environment-specific variables in an inventory file?

Use group variables for each environment:

```ini
[dev]
dev-server1

[dev:vars]
env=development
app_port=8080

[prod]
prod-server1

[prod:vars]
env=production
app_port=80
```

This lets you apply different configurations per environment using the same playbook.

---

### Q5. How do you define host-specific or group-specific sudo options in a static inventory?

Set privilege escalation variables per group:

```ini
[web:vars]
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root

[db:vars]
ansible_become=true
ansible_become_method=sudo
ansible_become_user=postgres
```

---

### Q6. How do you use group variables to manage location-specific network settings?

Define network variables per location group:

```ini
[us_east]
server1
server2

[us_east:vars]
gateway=10.0.1.1
dns_server=10.0.1.53

[eu_west]
server3
server4

[eu_west:vars]
gateway=10.1.1.1
dns_server=10.1.1.53
```

---

### Q7. How do you structure a static inventory for a hybrid environment (cloud + on-premise)?

Use groups to separate cloud and on-premise hosts:

```ini
[cloud]
aws-server1 ansible_host=54.x.x.x
azure-server1 ansible_host=40.x.x.x

[onprem]
local-server1 ansible_host=192.168.1.10
local-server2 ansible_host=192.168.1.11

[cloud:vars]
ansible_user=ec2-user

[onprem:vars]
ansible_user=admin
```

---

### Q8. How do you define proxy settings for specific groups of hosts in the inventory?

Use `ansible_ssh_common_args` or environment variables:

```ini
[asia:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p proxy-asia.example.com"'

[europe:vars]
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p proxy-eu.example.com"'
```

---

### Q9. How do you split the inventory into multiple files for different environments?

Create a directory structure:

```
inventory/
  dev.ini
  staging.ini
  prod.ini
```

Reference the directory in `ansible.cfg`:

```ini
[defaults]
inventory = ./inventory/
```

Ansible automatically reads all files in the directory. You can also target a specific file: `ansible-playbook -i inventory/prod.ini playbook.yml`.

---

### Q10. How do you specify a custom ansible.cfg for a particular project?

Ansible searches for configuration in this order:
1. `ANSIBLE_CONFIG` environment variable
2. `ansible.cfg` in the current directory
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

Place an `ansible.cfg` in your project root:

```ini
[defaults]
inventory = ./inventory/hosts
remote_user = deploy
log_path = ./logs/ansible.log
```

Or set the environment variable: `export ANSIBLE_CONFIG=/path/to/project/ansible.cfg`.

---

### Q11. How do you configure ansible.cfg to store playbook logs in a dedicated directory?

Add the `log_path` directive:

```ini
[defaults]
log_path = /var/log/ansible/playbook.log
```

Ensure the directory exists and the user running Ansible has write permissions.

---

### Q12. How do you configure ansible.cfg to automatically retry failed tasks?

Use the `retries` setting in `ansible.cfg`:

```ini
[defaults]
retry_files_enabled = True
retry_files_save_path = ./retry-files
```

For task-level retries, use `retries` and `delay` in the task itself (covered later). The `ansible.cfg` primarily controls retry file behavior, not automatic task retries.

---

## Section 2: SSH, Authentication, and Privilege Escalation

### Q13. How do you set up passwordless SSH access from the control node to managed nodes?

Generate an SSH key pair and distribute the public key:

```bash
# On the control node
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key -N ""

# Distribute to managed nodes
ssh-copy-id -i ~/.ssh/ansible_key.pub user@managed-node1
ssh-copy-id -i ~/.ssh/ansible_key.pub user@managed-node2
```

Or automate with a playbook using the `authorized_key` module:

```yaml
- name: Distribute SSH keys
  hosts: all
  tasks:
    - name: Add SSH public key
      authorized_key:
        user: "{{ ansible_user }}"
        key: "{{ lookup('file', '~/.ssh/ansible_key.pub') }}"
        state: present
```

---

### Q14. How do you verify that SSH keys are correctly distributed to all managed nodes?

Use the `authorized_key` module or check the file directly:

```yaml
- name: Verify SSH key distribution
  hosts: all
  tasks:
    - name: Check authorized_keys file
      shell: grep -c "ansible_key" ~/.ssh/authorized_keys
      register: key_check
      changed_when: false

    - name: Report key status
      debug:
        msg: "SSH key is {{ 'present' if key_check.stdout | int > 0 else 'missing' }}"
```

---

### Q15. How do you configure /etc/sudoers so the ansible user can run sudo without a password?

Use the `lineinfile` module to add a sudoers entry:

```yaml
- name: Configure passwordless sudo
  hosts: all
  become: true
  tasks:
    - name: Add ansible user to sudoers
      lineinfile:
        path: /etc/sudoers.d/ansible
        line: "ansible ALL=(ALL) NOPASSWD: ALL"
        create: true
        validate: "visudo -cf %s"
        mode: '0440'
```

Always use `validate: "visudo -cf %s"` to prevent syntax errors in sudoers files.

---

### Q16. How do you use the `become` directive for privilege escalation in specific tasks?

Apply `become` at the task level rather than the play level:

```yaml
- name: Mixed privilege tasks
  hosts: all
  tasks:
    - name: Check uptime (no privilege needed)
      command: uptime
      register: uptime_result

    - name: Install package (requires root)
      package:
        name: httpd
        state: present
      become: true

    - name: Restart service (requires root)
      service:
        name: httpd
        state: restarted
      become: true
```

This limits privilege escalation to only the tasks that need it.

---

### Q17. How do you create a user with SSH access and sudo privileges using Ansible?

```yaml
- name: Set up devops user
  hosts: all
  become: true
  tasks:
    - name: Create devops user
      user:
        name: devops
        shell: /bin/bash
        create_home: true
        groups: wheel
        append: true

    - name: Add SSH public key
      authorized_key:
        user: devops
        key: "{{ lookup('file', '~/.ssh/devops_key.pub') }}"
        state: present

    - name: Grant passwordless sudo
      lineinfile:
        path: /etc/sudoers.d/devops
        line: "devops ALL=(ALL) NOPASSWD: ALL"
        create: true
        validate: "visudo -cf %s"
        mode: '0440'
```

---

## Section 3: Basic Playbook Structure and Modules

### Q18. How do you write a basic playbook to install a package and start a service?

```yaml
---
- name: Install and start nginx
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: true
```

The `package` module is distribution-agnostic. The `service` module manages service state.

---

### Q19. How do you install the httpd package ensuring the latest version on all web servers?

```yaml
- name: Install latest httpd
  hosts: web
  become: true
  tasks:
    - name: Install httpd (latest)
      package:
        name: httpd
        state: latest
```

Using `state: latest` ensures the most recent version is installed. Use `state: present` if you just need it installed regardless of version.

---

### Q20. How do you use the copy module to deploy a configuration file to managed nodes?

```yaml
- name: Deploy configuration file
  hosts: all
  become: true
  tasks:
    - name: Copy config file
      copy:
        src: files/app.conf
        dest: /etc/app/app.conf
        owner: root
        group: root
        mode: '0644'
```

---

### Q21. How do you use a loop to copy multiple files to managed nodes?

```yaml
- name: Deploy multiple config files
  hosts: all
  become: true
  tasks:
    - name: Copy configuration files
      copy:
        src: "files/{{ item }}"
        dest: "/opt/configs/{{ item }}"
        mode: '0644'
      loop:
        - file1.txt
        - file2.txt
        - file3.txt
```

---

### Q22. How do you create a directory with specific permissions using Ansible?

```yaml
- name: Create data directory
  hosts: all
  become: true
  tasks:
    - name: Create /data directory
      file:
        path: /data
        state: directory
        mode: '0755'
        owner: root
        group: root
```

---

### Q23. How do you install multiple packages in a single task using a loop?

```yaml
- name: Install web stack packages
  hosts: all
  become: true
  tasks:
    - name: Install packages
      package:
        name: "{{ item }}"
        state: present
      loop:
        - nginx
        - php
        - mysql-server
```

Or more efficiently, pass a list directly:

```yaml
    - name: Install packages
      package:
        name:
          - nginx
          - php
          - mysql-server
        state: present
```

---

### Q24. How do you use Ansible facts to retrieve and display system information?

```yaml
- name: Gather and display system info
  hosts: all
  tasks:
    - name: Display OS and memory info
      debug:
        msg: |
          Hostname: {{ ansible_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          Memory: {{ ansible_memtotal_mb }} MB
          IP: {{ ansible_default_ipv4.address }}
```

Facts are gathered automatically at the start of each play (unless `gather_facts: false` is set).

---

### Q25. How do you retrieve command output and store it in a variable?

Use the `command` or `shell` module with `register`:

```yaml
- name: Get system info
  hosts: all
  tasks:
    - name: Check disk space
      shell: df -h
      register: disk_info
      changed_when: false

    - name: Display disk space
      debug:
        var: disk_info.stdout_lines

    - name: Get kernel version
      command: uname -r
      register: kernel_version
      changed_when: false

    - name: Display kernel version
      debug:
        msg: "Kernel: {{ kernel_version.stdout }}"
```

---

### Q26. How do you retrieve the hostname of each managed node and display it?

```yaml
- name: Display hostnames
  hosts: all
  tasks:
    - name: Show hostname
      debug:
        msg: "This host is {{ ansible_hostname }} (FQDN: {{ ansible_fqdn }})"
```

`ansible_hostname` and `ansible_fqdn` are automatically gathered facts.

---

## Section 4: Conditionals, Error Handling, and Control Flow

### Q27. How do you run a task only if a specific condition is met (e.g., enough RAM)?

Use the `when` directive with Ansible facts:

```yaml
- name: Conditional task based on memory
  hosts: all
  tasks:
    - name: Run memory-intensive task
      command: /opt/scripts/heavy_process.sh
      when: ansible_memtotal_mb > 4096
```

---

### Q28. How do you run a task only if a specific file or directory exists?

Use the `stat` module to check, then use `when`:

```yaml
- name: Conditional based on file existence
  hosts: all
  tasks:
    - name: Check if /var/app directory exists
      stat:
        path: /var/app
      register: app_dir

    - name: Configure application (only if directory exists)
      template:
        src: app.conf.j2
        dest: /var/app/app.conf
      when: app_dir.stat.exists and app_dir.stat.isdir
```

---

### Q29. How do you install software only if a particular file does not exist?

```yaml
- name: Conditional software installation
  hosts: all
  become: true
  tasks:
    - name: Check if lock file exists
      stat:
        path: /opt/app/installed.lock
      register: lock_file

    - name: Install application
      package:
        name: myapp
        state: present
      when: not lock_file.stat.exists
```

---

### Q30. How do you run a task only if a specific service is already installed?

```yaml
- name: Conditional on service existence
  hosts: all
  tasks:
    - name: Check if httpd is installed
      command: rpm -q httpd
      register: httpd_check
      failed_when: false
      changed_when: false

    - name: Configure httpd (only if installed)
      template:
        src: httpd.conf.j2
        dest: /etc/httpd/conf/httpd.conf
      when: httpd_check.rc == 0
```

---

### Q31. How do you run a task only if a service is not already running?

```yaml
- name: Start service if not running
  hosts: all
  become: true
  tasks:
    - name: Check if nginx is running
      command: systemctl is-active nginx
      register: nginx_status
      failed_when: false
      changed_when: false

    - name: Start nginx
      service:
        name: nginx
        state: started
      when: nginx_status.rc != 0
```

---

### Q32. How do you execute a task only on a specific OS version?

Use the `ansible_distribution_version` fact:

```yaml
- name: OS-version-specific task
  hosts: all
  become: true
  tasks:
    - name: Apply CentOS 8 specific patch
      yum:
        name: security-patch-centos8
        state: present
      when: ansible_distribution == "CentOS" and ansible_distribution_major_version == "8"
```

---

### Q33. How do you run a task only if disk usage exceeds a threshold?

```yaml
- name: Disk usage check
  hosts: all
  tasks:
    - name: Get disk usage percentage
      shell: df / --output=pcent | tail -1 | tr -d ' %'
      register: disk_usage
      changed_when: false

    - name: Run cleanup if disk usage exceeds 80%
      command: /opt/scripts/cleanup.sh
      when: disk_usage.stdout | int > 80
```

---

### Q34. How do you use `ignore_errors` to continue a playbook when a task fails?

```yaml
- name: Resilient playbook
  hosts: all
  tasks:
    - name: Attempt to fetch remote data (may fail)
      uri:
        url: https://api.example.com/data
        return_content: true
      register: api_result
      ignore_errors: true

    - name: Use fallback if API failed
      debug:
        msg: "API call {{ 'succeeded' if api_result is success else 'failed, using fallback' }}"
```

---

### Q35. How do you retry a failed task multiple times with a delay?

Use `retries` and `delay`:

```yaml
- name: Retry unreliable task
  hosts: all
  tasks:
    - name: Download package (with retries)
      get_url:
        url: https://example.com/package.tar.gz
        dest: /tmp/package.tar.gz
      retries: 3
      delay: 10
      register: download_result
      until: download_result is success
```

The task will retry up to 3 times, waiting 10 seconds between attempts.

---

### Q36. How do you use `failed_when` to trigger failure based on custom conditions?

```yaml
- name: Custom failure conditions
  hosts: all
  tasks:
    - name: Run health check
      shell: /opt/scripts/healthcheck.sh
      register: health_result
      failed_when: "'CRITICAL' in health_result.stdout"

    - name: Check config syntax
      command: nginx -t
      register: nginx_test
      failed_when: nginx_test.rc != 0 or 'error' in nginx_test.stderr
```

---

### Q37. How do you use the `fail` module to trigger a custom error message?

```yaml
- name: Validate prerequisites
  hosts: all
  tasks:
    - name: Check available disk space
      shell: df / --output=avail | tail -1
      register: disk_avail
      changed_when: false

    - name: Fail if insufficient disk space
      fail:
        msg: "Insufficient disk space: only {{ disk_avail.stdout | trim }} KB available. Need at least 5GB."
      when: disk_avail.stdout | trim | int < 5242880
```

---

### Q38. How do you log a custom success message using the debug module?

```yaml
- name: Task with success logging
  hosts: all
  become: true
  tasks:
    - name: Install application
      package:
        name: myapp
        state: present
      register: install_result

    - name: Log success
      debug:
        msg: "Application installed successfully on {{ ansible_hostname }}"
      when: install_result is success
```

---

### Q39. How do you use `notify` and `handlers` to restart a service only when a config changes?

```yaml
- name: Configure web server
  hosts: web
  become: true
  tasks:
    - name: Deploy nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

The handler only runs if the task reports a change. Handlers execute at the end of the play.

---

### Q40. How do you use `with_first_found` to deploy environment-specific config files?

```yaml
- name: Deploy environment-specific config
  hosts: all
  become: true
  vars:
    env: production
  tasks:
    - name: Deploy config file
      copy:
        src: "{{ item }}"
        dest: /etc/app/config.yml
      with_first_found:
        - "configs/{{ env }}.yml"
        - "configs/staging.yml"
        - "configs/default.yml"
```

Ansible tries each file in order and uses the first one found.

---

## Section 5: Templates and Jinja2

### Q41. How do you use the template module to deploy a customized configuration file?

Create a Jinja2 template (`templates/app.conf.j2`):

```jinja2
# Generated by Ansible
ServerName {{ ansible_hostname }}
ServerIP {{ ansible_default_ipv4.address }}
Environment {{ env }}
```

Deploy it:

```yaml
- name: Deploy customized config
  hosts: all
  become: true
  vars:
    env: production
  tasks:
    - name: Generate config from template
      template:
        src: templates/app.conf.j2
        dest: /etc/app/app.conf
        owner: root
        mode: '0644'
```

---

### Q42. How do you create an Nginx template with environment-specific server names?

Template (`templates/nginx.conf.j2`):

```jinja2
server {
    listen 80;
    server_name {{ server_name }};
    root {{ document_root }};

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Playbook:

```yaml
- name: Deploy Nginx config
  hosts: web
  become: true
  vars:
    server_name: "{{ 'dev.example.com' if env == 'dev' else 'www.example.com' }}"
    document_root: /var/www/html
  tasks:
    - name: Deploy Nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/site.conf
      notify: Reload nginx

  handlers:
    - name: Reload nginx
      service:
        name: nginx
        state: reloaded
```

---

### Q43. How do you deploy a customized Apache configuration using a Jinja2 template?

Template (`templates/apache.conf.j2`):

```jinja2
<VirtualHost *:80>
    ServerAdmin {{ server_admin }}
    DocumentRoot {{ document_root }}
    ServerName {{ ansible_fqdn }}

    <Directory {{ document_root }}>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog /var/log/httpd/error.log
    CustomLog /var/log/httpd/access.log combined
</VirtualHost>
```

Playbook:

```yaml
- name: Configure Apache
  hosts: web
  become: true
  vars:
    server_admin: "admin@example.com"
    document_root: /var/www/html
  tasks:
    - name: Deploy Apache config
      template:
        src: templates/apache.conf.j2
        dest: /etc/httpd/conf.d/site.conf
      notify: Restart Apache

  handlers:
    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

---

### Q44. How do you deploy a customized SSH configuration using the template module?

Template (`templates/sshd_config.j2`):

```jinja2
Port {{ ssh_port | default(22) }}
PermitRootLogin {{ permit_root_login | default('no') }}
PasswordAuthentication {{ password_auth | default('no') }}
PubkeyAuthentication yes
```

Playbook with per-host variables:

```yaml
- name: Deploy SSH config
  hosts: all
  become: true
  tasks:
    - name: Deploy sshd_config
      template:
        src: templates/sshd_config.j2
        dest: /etc/ssh/sshd_config
        validate: "sshd -t -f %s"
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted
```

Define per-host variables in `host_vars/server1.yml`:

```yaml
ssh_port: 2222
permit_root_login: "no"
```

---

### Q45. How do you deploy a MySQL configuration file with environment-specific memory settings?

Template (`templates/my.cnf.j2`):

```jinja2
[mysqld]
innodb_buffer_pool_size = {{ innodb_buffer_pool_size }}
max_connections = {{ max_connections }}
query_cache_size = {{ query_cache_size }}
datadir = /var/lib/mysql
socket = /var/lib/mysql/mysql.sock
```

Playbook:

```yaml
- name: Deploy MySQL config
  hosts: db
  become: true
  vars_files:
    - "vars/{{ env }}.yml"
  tasks:
    - name: Deploy MySQL config
      template:
        src: templates/my.cnf.j2
        dest: /etc/my.cnf
      notify: Restart MySQL

  handlers:
    - name: Restart MySQL
      service:
        name: mysqld
        state: restarted
```

Environment vars file (`vars/production.yml`):

```yaml
innodb_buffer_pool_size: 4G
max_connections: 500
query_cache_size: 128M
```

---

### Q46. How do you configure persistent network settings using the template module?

Template (`templates/ifcfg.j2`):

```jinja2
TYPE=Ethernet
BOOTPROTO=static
IPADDR={{ ip_address }}
GATEWAY={{ gateway }}
DNS1={{ dns_server }}
ONBOOT=yes
DEVICE={{ network_interface | default('eth0') }}
```

Playbook:

```yaml
- name: Configure network
  hosts: all
  become: true
  tasks:
    - name: Deploy network config
      template:
        src: templates/ifcfg.j2
        dest: "/etc/sysconfig/network-scripts/ifcfg-{{ network_interface | default('eth0') }}"
      notify: Restart network

  handlers:
    - name: Restart network
      service:
        name: NetworkManager
        state: restarted
```

---

### Q47. How do you deploy a customized fstab file using the template module?

Template (`templates/fstab.j2`):

```jinja2
# /etc/fstab - Generated by Ansible
{% for mount in mounts %}
{{ mount.device }}  {{ mount.path }}  {{ mount.fstype }}  {{ mount.opts | default('defaults') }}  0  {{ mount.dump | default('0') }}
{% endfor %}
```

Playbook:

```yaml
- name: Deploy fstab
  hosts: all
  become: true
  vars:
    mounts:
      - { device: "/dev/sda1", path: "/", fstype: "xfs", opts: "defaults" }
      - { device: "/dev/sdb1", path: "/data", fstype: "ext4", opts: "defaults,noatime" }
  tasks:
    - name: Deploy fstab
      template:
        src: templates/fstab.j2
        dest: /etc/fstab
        backup: true
```

---

### Q48. How do you use the template module to configure time zones based on server location?

```yaml
- name: Configure timezone
  hosts: all
  become: true
  tasks:
    - name: Set timezone
      timezone:
        name: "{{ timezone }}"
```

Define `timezone` in group_vars:

```yaml
# group_vars/us_east.yml
timezone: America/New_York

# group_vars/eu_west.yml
timezone: Europe/London
```

---

## Section 6: Ansible Vault — Managing Sensitive Data

### Q49. How do you use Ansible Vault to encrypt sensitive variables like database passwords?

Create an encrypted variables file:

```bash
ansible-vault create vars/secrets.yml
```

Add sensitive data:

```yaml
db_password: "s3cur3P@ss"
api_key: "abc123xyz"
```

Reference in a playbook:

```yaml
- name: Deploy with secrets
  hosts: db
  become: true
  vars_files:
    - vars/secrets.yml
  tasks:
    - name: Configure database
      template:
        src: db.conf.j2
        dest: /etc/myapp/db.conf
```

Run with: `ansible-playbook playbook.yml --ask-vault-pass` or `--vault-password-file`.

---

### Q50. How do you encrypt specific files (like SSH keys) with Ansible Vault?

Encrypt an existing file:

```bash
ansible-vault encrypt files/ssh_private_key
```

Use it in a playbook:

```yaml
- name: Deploy SSH key
  hosts: all
  become: true
  tasks:
    - name: Copy encrypted SSH key
      copy:
        src: files/ssh_private_key
        dest: /root/.ssh/id_rsa
        mode: '0600'
```

Ansible automatically decrypts the file during playbook execution when the vault password is provided.

---

### Q51. How do you store encrypted user passwords and decrypt them during playbook execution?

Create encrypted vars:

```bash
ansible-vault create vars/user_passwords.yml
```

Content:

```yaml
users:
  - name: alice
    password: "$6$rounds=656000$salt$hashedpassword1"
  - name: bob
    password: "$6$rounds=656000$salt$hashedpassword2"
```

Playbook:

```yaml
- name: Create users with encrypted passwords
  hosts: all
  become: true
  vars_files:
    - vars/user_passwords.yml
  tasks:
    - name: Create users
      user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        state: present
      loop: "{{ users }}"
      no_log: true
```

Use `no_log: true` to prevent passwords from appearing in output.

---

## Section 7: Package Management and System Updates

### Q52. How do you perform a system-wide update on all managed nodes?

```yaml
- name: System update
  hosts: all
  become: true
  tasks:
    - name: Update all packages (RHEL/CentOS)
      yum:
        name: "*"
        state: latest
      when: ansible_os_family == "RedHat"

    - name: Update all packages (Debian/Ubuntu)
      apt:
        upgrade: dist
        update_cache: true
      when: ansible_os_family == "Debian"
```

---

### Q53. How do you install updates while excluding the kernel package?

```yaml
- name: Update without kernel
  hosts: all
  become: true
  tasks:
    - name: Install all updates except kernel
      yum:
        name: "*"
        state: latest
        exclude: kernel*
```

---

### Q54. How do you install security patches only?

```yaml
- name: Install security updates
  hosts: all
  become: true
  tasks:
    - name: Install security updates (RHEL/CentOS)
      yum:
        name: "*"
        state: latest
        security: true
      when: ansible_os_family == "RedHat"

    - name: Install security updates (Debian/Ubuntu)
      apt:
        upgrade: safe
        update_cache: true
      when: ansible_os_family == "Debian"
```

---

### Q55. How do you install a specific version of Python across different distributions?

```yaml
- name: Install Python 3.8
  hosts: all
  become: true
  tasks:
    - name: Install Python 3.8 on CentOS
      yum:
        name: python38
        state: present
      when: ansible_distribution == "CentOS"

    - name: Install Python 3.8 on Ubuntu
      apt:
        name: python3.8
        state: present
        update_cache: true
      when: ansible_distribution == "Ubuntu"

    - name: Verify Python version
      command: python3.8 --version
      register: python_version
      changed_when: false

    - name: Display version
      debug:
        var: python_version.stdout
```

---

### Q56. How do you install a specific version of Node.js across all development servers?

```yaml
- name: Install Node.js
  hosts: dev
  become: true
  tasks:
    - name: Add NodeSource repository
      shell: curl -fsSL https://rpm.nodesource.com/setup_18.x | bash -
      args:
        creates: /etc/yum.repos.d/nodesource-el.repo

    - name: Install Node.js
      package:
        name: nodejs
        state: present

    - name: Verify Node.js version
      command: node --version
      register: node_ver
      changed_when: false

    - name: Display version
      debug:
        var: node_ver.stdout
```

---

### Q57. How do you add and enable repositories using yum_repository or apt_repository?

```yaml
- name: Manage repositories
  hosts: all
  become: true
  tasks:
    - name: Add EPEL repository (RHEL/CentOS)
      yum_repository:
        name: epel
        description: EPEL Repository
        baseurl: https://download.fedoraproject.org/pub/epel/$releasever/$basearch/
        gpgcheck: true
        gpgkey: https://download.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-{{ ansible_distribution_major_version }}
        enabled: true
      when: ansible_os_family == "RedHat"

    - name: Add Docker repository (Ubuntu)
      apt_repository:
        repo: "deb https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
      when: ansible_os_family == "Debian"
```

---

## Section 8: Service Management

### Q58. How do you stop and disable a service across all managed nodes?

```yaml
- name: Stop and disable vsftpd
  hosts: all
  become: true
  tasks:
    - name: Stop and disable vsftpd
      service:
        name: vsftpd
        state: stopped
        enabled: false
```

---

### Q59. How do you ensure multiple services are installed, enabled, and started?

```yaml
- name: Manage web stack services
  hosts: web
  become: true
  tasks:
    - name: Install services
      package:
        name:
          - nginx
          - mysql-server
        state: present

    - name: Ensure services are running and enabled
      service:
        name: "{{ item }}"
        state: started
        enabled: true
      loop:
        - nginx
        - mysqld
```

---

### Q60. How do you configure a service to restart automatically on failure using systemd?

```yaml
- name: Configure auto-restart for httpd
  hosts: web
  become: true
  tasks:
    - name: Create systemd override directory
      file:
        path: /etc/systemd/system/httpd.service.d
        state: directory

    - name: Add restart policy
      copy:
        content: |
          [Service]
          Restart=on-failure
          RestartSec=5
        dest: /etc/systemd/system/httpd.service.d/restart.conf
      notify:
        - Reload systemd
        - Restart httpd

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: true

    - name: Restart httpd
      service:
        name: httpd
        state: restarted
```

---

### Q61. How do you configure a systemd service to wait for network availability before starting?

```yaml
- name: Configure service to wait for network
  hosts: all
  become: true
  tasks:
    - name: Create systemd override
      copy:
        content: |
          [Unit]
          After=network-online.target
          Wants=network-online.target
        dest: /etc/systemd/system/myservice.service.d/network-wait.conf
      notify: Reload systemd

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: true
```

---

## Section 9: File Management and Text Manipulation

### Q62. How do you use the `blockinfile` module to manage configuration blocks?

```yaml
- name: Manage config blocks
  hosts: all
  become: true
  tasks:
    - name: Add custom settings block
      blockinfile:
        path: /etc/app/settings.conf
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        block: |
          max_connections = 100
          timeout = 30
          log_level = info
```

The marker comments let Ansible identify and update the block on subsequent runs.

---

### Q63. How do you use the `replace` module to find and replace lines in a file?

```yaml
- name: Update configuration values
  hosts: all
  become: true
  tasks:
    - name: Change max connections
      replace:
        path: /etc/app/config.conf
        regexp: '^max_connections\s*=\s*\d+'
        replace: 'max_connections = 200'
```

---

### Q64. How do you verify a file's checksum before copying it?

```yaml
- name: Copy file with checksum verification
  hosts: all
  become: true
  tasks:
    - name: Get source file checksum
      stat:
        path: /tmp/source_file.tar.gz
        checksum_algorithm: sha256
      register: source_stat
      delegate_to: localhost

    - name: Copy file if checksum matches
      copy:
        src: /tmp/source_file.tar.gz
        dest: /opt/app/source_file.tar.gz
        checksum: "{{ source_stat.stat.checksum }}"
```

---

### Q65. How do you use Ansible facts to get the IP address and insert it into a config file?

```yaml
- name: Deploy config with IP
  hosts: all
  become: true
  tasks:
    - name: Deploy config with node IP
      template:
        src: templates/app.conf.j2
        dest: /etc/app/app.conf
```

Template (`templates/app.conf.j2`):

```jinja2
bind_address = {{ ansible_default_ipv4.address }}
hostname = {{ ansible_hostname }}
```

---

### Q66. How do you manage fine-grained file permissions with ACLs using Ansible?

```yaml
- name: Set file ACL
  hosts: all
  become: true
  tasks:
    - name: Grant developer read/write access to specific file
      acl:
        path: /srv/shared-data/config.yml
        entity: developer1
        etype: user
        permissions: rw
        state: present
```

---

## Section 10: Firewall Configuration

### Q67. How do you configure firewalld to allow traffic on a custom port (e.g., 8080)?

```yaml
- name: Allow custom port
  hosts: all
  become: true
  tasks:
    - name: Allow port 8080
      firewalld:
        port: 8080/tcp
        permanent: true
        immediate: true
        state: enabled
```

`permanent: true` ensures the rule persists after reboot. `immediate: true` applies it now.

---

### Q68. How do you configure firewalld to reject all traffic except SSH and HTTP/HTTPS?

```yaml
- name: Configure restrictive firewall
  hosts: all
  become: true
  tasks:
    - name: Set default zone to drop
      command: firewall-cmd --set-default-zone=drop

    - name: Allow SSH
      firewalld:
        service: ssh
        permanent: true
        immediate: true
        state: enabled

    - name: Allow HTTP
      firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled

    - name: Allow HTTPS
      firewalld:
        service: https
        permanent: true
        immediate: true
        state: enabled
```

---

### Q69. How do you deploy role-specific firewall rules based on server type?

```yaml
- name: Web server firewall rules
  hosts: web
  become: true
  tasks:
    - name: Allow web services
      firewalld:
        service: "{{ item }}"
        permanent: true
        immediate: true
        state: enabled
      loop:
        - http
        - https

- name: Database server firewall rules
  hosts: db
  become: true
  tasks:
    - name: Allow MySQL
      firewalld:
        port: 3306/tcp
        permanent: true
        immediate: true
        state: enabled

    - name: Deny public access
      firewalld:
        service: http
        permanent: true
        state: disabled
```

---

### Q70. How do you deploy persistent firewall configurations using templates?

Template (`templates/firewall_rules.xml.j2`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<zone target="DROP">
{% for service in allowed_services %}
  <service name="{{ service }}"/>
{% endfor %}
{% for port in allowed_ports %}
  <port protocol="{{ port.proto }}" port="{{ port.number }}"/>
{% endfor %}
</zone>
```

Playbook:

```yaml
- name: Deploy custom firewall zone
  hosts: all
  become: true
  vars:
    allowed_services: [ssh, http]
    allowed_ports:
      - { number: 8080, proto: tcp }
  tasks:
    - name: Deploy firewall zone config
      template:
        src: templates/firewall_rules.xml.j2
        dest: /etc/firewalld/zones/custom.xml
      notify: Reload firewalld

  handlers:
    - name: Reload firewalld
      service:
        name: firewalld
        state: reloaded
```

---

## Section 11: Cron Jobs and Scheduling

### Q71. How do you schedule a cleanup script to run every hour using the cron module?

```yaml
- name: Schedule hourly cleanup
  hosts: all
  become: true
  tasks:
    - name: Add cron job for temp cleanup
      cron:
        name: "Clean temp files"
        minute: "0"
        job: "/opt/scripts/cleanup_temp.sh"
        user: root
```

---

### Q72. How do you schedule a weekly archive of /var/log every Sunday at 1 AM?

```yaml
- name: Schedule weekly log archive
  hosts: all
  become: true
  tasks:
    - name: Create weekly log archive cron job
      cron:
        name: "Weekly log archive"
        minute: "0"
        hour: "1"
        weekday: "0"
        job: "tar czf /backup/logs-$(date +\\%Y\\%m\\%d).tar.gz /var/log"
        user: root
```

---

### Q73. How do you schedule application updates at a specific time to avoid downtime?

```yaml
- name: Schedule maintenance update
  hosts: all
  become: true
  tasks:
    - name: Schedule update at 2 AM Saturday
      cron:
        name: "Scheduled app update"
        minute: "0"
        hour: "2"
        weekday: "6"
        job: "/opt/scripts/update_app.sh >> /var/log/update.log 2>&1"
        user: root
```

Or use the `at` module for a one-time scheduled task:

```yaml
    - name: Schedule one-time update
      at:
        command: "/opt/scripts/update_app.sh"
        count: 2
        units: hours
```

---

## Section 12: User and Group Management

### Q74. How do you manage group memberships for users using the group and user modules?

```yaml
- name: Manage groups and users
  hosts: all
  become: true
  tasks:
    - name: Create groups
      group:
        name: "{{ item }}"
        state: present
      loop:
        - developers
        - operations

    - name: Create users with group memberships
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        append: true
      loop:
        - { name: alice, groups: "developers" }
        - { name: bob, groups: "developers,operations" }
```

---

### Q75. How do you create users with specific UID, GID, and home directory?

```yaml
- name: Create devops user
  hosts: all
  become: true
  tasks:
    - name: Create devops group
      group:
        name: devops
        gid: 2000

    - name: Create devops user
      user:
        name: devops
        uid: 2000
        group: devops
        home: /home/devops
        shell: /bin/bash
        create_home: true
```

---

### Q76. How do you automate user creation with sudo privileges?

```yaml
- name: Create users with sudo
  hosts: all
  become: true
  tasks:
    - name: Create admin group
      group:
        name: admin
        state: present

    - name: Create users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        append: true
        state: present
      loop:
        - { name: deploy, groups: "admin,wheel" }
        - { name: monitor, groups: "admin" }

    - name: Grant sudo to admin group
      lineinfile:
        path: /etc/sudoers.d/admin
        line: "%admin ALL=(ALL) NOPASSWD: ALL"
        create: true
        validate: "visudo -cf %s"
        mode: '0440'
```

---

### Q77. How do you use loops to create multiple users with unique home directories?

```yaml
- name: Create multiple users
  hosts: all
  become: true
  tasks:
    - name: Create users
      user:
        name: "{{ item }}"
        home: "/home/{{ item }}"
        create_home: true
        shell: /bin/bash
      loop:
        - developer1
        - developer2
        - tester1
```

---

### Q78. How do you enforce password complexity requirements using PAM?

```yaml
- name: Configure password complexity
  hosts: all
  become: true
  tasks:
    - name: Install PAM password quality module
      package:
        name: libpam-pwquality
        state: present

    - name: Configure password complexity
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: "^{{ item.key }}"
        line: "{{ item.key }} = {{ item.value }}"
      loop:
        - { key: "minlen", value: "12" }
        - { key: "dcredit", value: "-1" }
        - { key: "ucredit", value: "-1" }
        - { key: "lcredit", value: "-1" }
        - { key: "ocredit", value: "-1" }
```

---

## Section 13: Storage, Filesystems, and Disk Management

### Q79. How do you format a device with XFS and mount it on /data?

```yaml
- name: Format and mount storage
  hosts: all
  become: true
  tasks:
    - name: Create XFS filesystem
      filesystem:
        fstype: xfs
        dev: /dev/sdb

    - name: Mount /data
      mount:
        path: /data
        src: /dev/sdb
        fstype: xfs
        state: mounted
```

`state: mounted` both mounts the device and adds it to `/etc/fstab` for persistence.

---

### Q80. How do you resize a partition to use all available space using the parted module?

```yaml
- name: Expand partition
  hosts: all
  become: true
  tasks:
    - name: Resize partition to use all space
      parted:
        device: /dev/sdb
        number: 1
        state: present
        part_end: "100%"
        resize: true
```

---

### Q81. How do you remove a logical volume and reclaim storage?

```yaml
- name: Remove logical volume
  hosts: all
  become: true
  tasks:
    - name: Unmount the logical volume
      mount:
        path: /mnt/data
        state: absent

    - name: Remove logical volume
      lvol:
        vg: my_vg
        lv: my_lv
        state: absent
        force: true
```

---

### Q82. How do you create and enable a 2GB swap file using Ansible?

```yaml
- name: Create swap file
  hosts: all
  become: true
  tasks:
    - name: Create 2GB swap file
      command: dd if=/dev/zero of=/swapfile bs=1M count=2048
      args:
        creates: /swapfile

    - name: Set swap file permissions
      file:
        path: /swapfile
        mode: '0600'

    - name: Format swap file
      command: mkswap /swapfile
      when: ansible_swaptotal_mb < 2048

    - name: Enable swap
      command: swapon /swapfile

    - name: Add to fstab
      lineinfile:
        path: /etc/fstab
        line: "/swapfile swap swap defaults 0 0"
        state: present
```

---

### Q83. How do you configure software RAID 0 on two disks, mount it, and make it persistent?

```yaml
- name: Configure RAID 0
  hosts: all
  become: true
  tasks:
    - name: Install mdadm
      package:
        name: mdadm
        state: present

    - name: Create RAID 0 array
      command: mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
      args:
        creates: /dev/md0

    - name: Create filesystem on RAID
      filesystem:
        fstype: xfs
        dev: /dev/md0

    - name: Create mount point
      file:
        path: /mnt/raid
        state: directory

    - name: Mount RAID array
      mount:
        path: /mnt/raid
        src: /dev/md0
        fstype: xfs
        state: mounted

    - name: Save RAID config
      shell: mdadm --detail --scan >> /etc/mdadm.conf
```

---

### Q84. How do you monitor inode usage and recreate a filesystem if needed?

```yaml
- name: Monitor inode usage
  hosts: all
  become: true
  tasks:
    - name: Check inode usage
      shell: df -i / | tail -1 | awk '{print $5}' | tr -d '%'
      register: inode_usage
      changed_when: false

    - name: Alert on high inode usage
      debug:
        msg: "WARNING: Inode usage is {{ inode_usage.stdout }}% on {{ ansible_hostname }}"
      when: inode_usage.stdout | int > 80

    - name: Recreate filesystem with more inodes (if needed on separate partition)
      command: mkfs.ext4 -N 1000000 /dev/sdc1
      when: inode_usage.stdout | int > 95 and recreate_fs | default(false)
```

---

## Section 14: Ansible Roles

### Q85. How do you create an Ansible role for installing and configuring nginx?

Create the role structure:

```
roles/nginx/
  tasks/main.yml
  handlers/main.yml
  templates/nginx.conf.j2
  defaults/main.yml
```

`tasks/main.yml`:

```yaml
---
- name: Install nginx
  package:
    name: nginx
    state: present

- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: Restart nginx

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: true
```

`handlers/main.yml`:

```yaml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted
```

`defaults/main.yml`:

```yaml
---
nginx_port: 80
nginx_worker_processes: auto
```

---

### Q86. How do you install and use a role from Ansible Galaxy?

```bash
# Install a role
ansible-galaxy install geerlingguy.nginx

# List installed roles
ansible-galaxy list
```

Use in a playbook:

```yaml
- name: Deploy nginx using Galaxy role
  hosts: web
  become: true
  roles:
    - geerlingguy.nginx
```

---

### Q87. How do you check for role updates and update a role from Ansible Galaxy?

```bash
# Check installed version
ansible-galaxy list

# Force update to latest version
ansible-galaxy install geerlingguy.nginx --force

# Install a specific version
ansible-galaxy install geerlingguy.nginx,3.1.0
```

---

### Q88. How do you install a custom role from a private Git repository?

```bash
ansible-galaxy install git+https://git.internal.example.com/team/custom-role.git,main
```

Or use a `requirements.yml`:

```yaml
roles:
  - name: custom-role
    src: git+https://git.internal.example.com/team/custom-role.git
    version: main
```

Install: `ansible-galaxy install -r requirements.yml`

---

### Q89. How do you use specific role versions per project?

Create a `requirements.yml` per project:

```yaml
roles:
  - name: geerlingguy.nginx
    version: 3.1.0
  - name: geerlingguy.mysql
    version: 4.0.0
```

Install: `ansible-galaxy install -r requirements.yml -p roles/`

The `-p roles/` installs into the project's local roles directory.

---

### Q90. How do you use role defaults and role vars to enforce defaults while allowing overrides?

In the role:

- `defaults/main.yml` — lowest priority, easily overridden:
  ```yaml
  http_port: 80
  max_clients: 100
  ```

- `vars/main.yml` — higher priority, harder to override:
  ```yaml
  required_packages:
    - nginx
    - openssl
  ```

Override defaults in the playbook:

```yaml
- hosts: web
  roles:
    - role: webserver
      vars:
        http_port: 8080
        max_clients: 200
```

---

### Q91. How do you create a role that orchestrates multiple existing roles (web, db, cache)?

Create a meta-role that includes other roles:

`roles/app_stack/tasks/main.yml`:

```yaml
---
- name: Include database role
  include_role:
    name: db
  vars:
    db_env: "{{ app_env }}"

- name: Include cache role
  include_role:
    name: cache

- name: Include web role
  include_role:
    name: web
  vars:
    web_env: "{{ app_env }}"
```

Use in a playbook:

```yaml
- hosts: all
  vars:
    app_env: production
  roles:
    - app_stack
```

---

### Q92. How do you use `when` conditions within a role to run tasks conditionally?

In `roles/security/tasks/main.yml`:

```yaml
---
- name: Apply security patches
  yum:
    name: "{{ security_patches }}"
    state: latest
  when: security_patching_enabled | default(true)

- name: Configure firewall
  include_tasks: firewall.yml
  when: "'firewalld' in ansible_facts.packages"
```

---

### Q93. How do you modify a role to handle both Linux and Windows hosts?

```yaml
# roles/user_management/tasks/main.yml
---
- name: Include Linux tasks
  include_tasks: linux.yml
  when: ansible_os_family != "Windows"

- name: Include Windows tasks
  include_tasks: windows.yml
  when: ansible_os_family == "Windows"
```

`roles/user_management/tasks/linux.yml`:

```yaml
- name: Create user on Linux
  user:
    name: "{{ username }}"
    state: present
```

`roles/user_management/tasks/windows.yml`:

```yaml
- name: Create user on Windows
  win_user:
    name: "{{ username }}"
    state: present
```

---

### Q94. How do you apply OS-version-specific patches within a role?

```yaml
# roles/patching/tasks/main.yml
---
- name: Include RHEL 8 patches
  include_tasks: rhel8.yml
  when: ansible_distribution == "RedHat" and ansible_distribution_major_version == "8"

- name: Include RHEL 9 patches
  include_tasks: rhel9.yml
  when: ansible_distribution == "RedHat" and ansible_distribution_major_version == "9"

- name: Include Ubuntu patches
  include_tasks: ubuntu.yml
  when: ansible_distribution == "Ubuntu"
```

---

### Q95. How do you create a role for database backups with rotation and scheduling?

```
roles/db_backup/
  tasks/main.yml
  templates/backup_script.sh.j2
  defaults/main.yml
```

`defaults/main.yml`:

```yaml
backup_dir: /backup/db
backup_retention_days: 7
backup_schedule_hour: 2
backup_schedule_minute: 0
```

`tasks/main.yml`:

```yaml
---
- name: Create backup directory
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0750'

- name: Deploy backup script
  template:
    src: backup_script.sh.j2
    dest: /usr/local/bin/db_backup.sh
    mode: '0750'

- name: Schedule backup cron job
  cron:
    name: "Database backup"
    hour: "{{ backup_schedule_hour }}"
    minute: "{{ backup_schedule_minute }}"
    job: "/usr/local/bin/db_backup.sh"

- name: Schedule old backup cleanup
  cron:
    name: "Cleanup old backups"
    hour: "3"
    minute: "0"
    job: "find {{ backup_dir }} -name '*.sql.gz' -mtime +{{ backup_retention_days }} -delete"
```

---

### Q96. How do you ensure roles execute in a specific order in a playbook?

```yaml
- name: Deploy multi-tier application
  hosts: all
  become: true
  roles:
    - role: database      # Runs first
    - role: app_server    # Runs second
    - role: web_server    # Runs third
    - role: load_balancer # Runs last
```

Roles execute in the order listed. For more control, use separate plays:

```yaml
- hosts: db_servers
  roles:
    - database

- hosts: app_servers
  roles:
    - app_server

- hosts: web_servers
  roles:
    - web_server
```

---

### Q97. How do you extend a Content Collection role without modifying its source?

Use `include_role` and add custom tasks before/after:

```yaml
- name: Extended web server setup
  hosts: web
  become: true
  tasks:
    - name: Pre-configuration tasks
      file:
        path: /var/www/custom
        state: directory

    - name: Include collection role
      include_role:
        name: community.general.web_server

    - name: Post-configuration custom tasks
      template:
        src: custom_vhost.conf.j2
        dest: /etc/httpd/conf.d/custom.conf
```

---

## Section 15: Ansible Content Collections

### Q98. How do you install and use a Content Collection from Ansible Galaxy?

```bash
# Install a collection
ansible-galaxy collection install amazon.aws

# Install from requirements file
ansible-galaxy collection install -r requirements.yml
```

`requirements.yml`:

```yaml
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: community.general
```

Use in a playbook:

```yaml
- name: Manage AWS resources
  hosts: localhost
  tasks:
    - name: Create S3 bucket
      amazon.aws.s3_bucket:
        name: my-bucket
        state: present
```

---

### Q99. How do you ensure a specific version of a module from a Content Collection is used?

Specify the version in `requirements.yml`:

```yaml
collections:
  - name: community.mysql
    version: "3.5.0"
```

Use the fully qualified collection name (FQCN) in playbooks:

```yaml
- name: Use specific collection module
  community.mysql.mysql_db:
    name: mydb
    state: present
```

---

### Q100. How do you override default variables in roles from a Content Collection?

```yaml
- name: Customize collection role
  hosts: web
  become: true
  roles:
    - role: geerlingguy.nginx
      vars:
        nginx_worker_processes: 4
        nginx_worker_connections: 2048
        nginx_keepalive_timeout: 30
```

Variables passed at the role invocation level override the role's `defaults/main.yml`.

---

### Q101. How do you inspect a Content Collection for its roles, modules, and dependencies?

```bash
# List installed collections
ansible-galaxy collection list

# View collection details
ansible-doc -l -t module community.general

# View specific module documentation
ansible-doc community.general.nmcli

# Check collection structure
ls ~/.ansible/collections/ansible_collections/community/general/
```

---

### Q102. How do you structure and publish a custom Content Collection?

Directory structure:

```
my_namespace/my_collection/
  galaxy.yml
  roles/
    database/
    app_server/
    load_balancer/
  plugins/
    modules/
      custom_module.py
  README.md
```

`galaxy.yml`:

```yaml
namespace: my_namespace
name: my_collection
version: 1.0.0
dependencies:
  community.general: ">=5.0.0"
```

Build and publish:

```bash
ansible-galaxy collection build
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz
```

---

## Section 16: Automation Content Navigator

### Q103. How do you use Automation Content Navigator to check installed Content Collection versions?

```bash
# Launch navigator
ansible-navigator collections

# Check specific collection
ansible-navigator doc community.general.nmcli
```

Navigator provides an interactive TUI to browse collections, modules, and their documentation.

---

### Q104. How do you use Navigator to run a playbook with specific credentials and inventory?

```bash
ansible-navigator run playbook.yml \
  -i inventory/prod.ini \
  --become \
  --become-method sudo
```

For different credentials per environment, use `--extra-vars` or vault:

```bash
ansible-navigator run playbook.yml \
  -i inventory/prod.ini \
  --vault-password-file vault_pass.txt
```

---

### Q105. How do you use Navigator to run a playbook with specific inventory groups?

```bash
ansible-navigator run playbook.yml -i inventory/ --limit web
```

Or target multiple groups:

```bash
ansible-navigator run playbook.yml -i inventory/ --limit "web:db"
```

---

### Q106. How do you use Navigator to identify and resolve module dependencies between collections?

```bash
# List all collections and their versions
ansible-navigator collections

# Check a specific module's requirements
ansible-navigator doc amazon.aws.ec2_instance

# Install missing dependencies
ansible-galaxy collection install -r requirements.yml
```

---

### Q107. How do you troubleshoot missing Content Collections during a playbook run?

```bash
# Check which collections are installed
ansible-navigator collections

# Verify the specific collection
ansible-galaxy collection list | grep community.docker

# Install missing collection
ansible-galaxy collection install community.docker

# Re-run the playbook
ansible-navigator run playbook.yml
```

---

## Section 17: Heterogeneous Environments and Conditional Deployments

### Q108. How do you manage a heterogeneous environment with CentOS and Ubuntu servers?

```yaml
- name: Install packages on mixed OS
  hosts: all
  become: true
  tasks:
    - name: Install on CentOS
      yum:
        name: httpd
        state: present
      when: ansible_distribution == "CentOS"

    - name: Install on Ubuntu
      apt:
        name: apache2
        state: present
        update_cache: true
      when: ansible_distribution == "Ubuntu"
```

---

### Q109. How do you download an application from different sources based on region?

```yaml
- name: Region-based download
  hosts: all
  become: true
  tasks:
    - name: Download from US mirror
      get_url:
        url: https://us-mirror.example.com/app.tar.gz
        dest: /tmp/app.tar.gz
      when: region == "us"

    - name: Download from EU mirror
      get_url:
        url: https://eu-mirror.example.com/app.tar.gz
        dest: /tmp/app.tar.gz
      when: region == "eu"
```

---

### Q110. How do you deploy security patches by server criticality group?

```yaml
# Patch high-criticality servers first
- name: Patch high-criticality servers
  hosts: high_criticality
  become: true
  serial: 1
  tasks:
    - name: Apply security patches
      yum:
        name: "*"
        state: latest
        security: true

# Then medium
- name: Patch medium-criticality servers
  hosts: medium_criticality
  become: true
  serial: 2
  tasks:
    - name: Apply security patches
      yum:
        name: "*"
        state: latest
        security: true

# Then low
- name: Patch low-criticality servers
  hosts: low_criticality
  become: true
  tasks:
    - name: Apply security patches
      yum:
        name: "*"
        state: latest
        security: true
```

`serial` controls how many hosts are patched at a time.

---

### Q111. How do you manage location-specific variables in roles for different regions?

Use `group_vars` per region:

```
group_vars/
  us_east.yml
  eu_west.yml
  asia_pacific.yml
```

`group_vars/us_east.yml`:

```yaml
ntp_server: time.us-east.example.com
dns_server: 10.0.1.53
app_mirror: https://us-east-mirror.example.com
```

Reference in roles:

```yaml
- name: Configure regional settings
  hosts: all
  roles:
    - role: regional_config
```

The role automatically picks up the correct variables based on the host's group membership.

---

### Q112. How do you dynamically assign hostnames based on IP addresses?

```yaml
- name: Set hostname from IP
  hosts: all
  become: true
  tasks:
    - name: Set hostname
      hostname:
        name: "server-{{ ansible_default_ipv4.address | replace('.', '-') }}"

    - name: Update /etc/hosts
      lineinfile:
        path: /etc/hosts
        line: "{{ ansible_default_ipv4.address }} server-{{ ansible_default_ipv4.address | replace('.', '-') }}"
```

---

## Section 18: Backups, Monitoring, and Logging

### Q113. How do you automate daily backups of /etc with compression?

```yaml
- name: Daily backup of /etc
  hosts: all
  become: true
  tasks:
    - name: Create backup directory
      file:
        path: /backup
        state: directory
        mode: '0750'

    - name: Schedule daily backup
      cron:
        name: "Daily /etc backup"
        hour: "2"
        minute: "0"
        job: "tar czf /backup/etc-$(date +\\%Y\\%m\\%d).tar.gz /etc"
```

---

### Q114. How do you back up an application directory before deploying a new version?

```yaml
- name: Deploy with backup
  hosts: app
  become: true
  tasks:
    - name: Backup current version
      archive:
        path: /opt/myapp
        dest: "/backup/myapp-{{ ansible_date_time.iso8601_basic_short }}.tar.gz"

    - name: Deploy new version
      unarchive:
        src: files/myapp-new.tar.gz
        dest: /opt/myapp
```

---

### Q115. How do you monitor disk usage and send alerts when a threshold is exceeded?

```yaml
- name: Disk usage monitoring
  hosts: all
  tasks:
    - name: Check disk usage
      shell: df / --output=pcent | tail -1 | tr -d ' %'
      register: disk_usage
      changed_when: false

    - name: Send alert if usage exceeds 90%
      mail:
        host: smtp.example.com
        to: admin@example.com
        subject: "ALERT: Disk usage {{ disk_usage.stdout }}% on {{ ansible_hostname }}"
        body: "Disk usage on {{ ansible_hostname }} has reached {{ disk_usage.stdout }}%."
      when: disk_usage.stdout | int > 90
```

---

### Q116. How do you configure rsyslog to forward logs to a central logging server?

```yaml
- name: Configure centralized logging
  hosts: all
  become: true
  tasks:
    - name: Install rsyslog
      package:
        name: rsyslog
        state: present

    - name: Configure log forwarding
      lineinfile:
        path: /etc/rsyslog.conf
        line: "*.* @@logserver.example.com:514"
        state: present
      notify: Restart rsyslog

  handlers:
    - name: Restart rsyslog
      service:
        name: rsyslog
        state: restarted
```

---

### Q117. How do you configure log rotation for web server logs using logrotate?

```yaml
- name: Configure log rotation
  hosts: web
  become: true
  tasks:
    - name: Deploy logrotate config for web server
      copy:
        content: |
          /var/log/nginx/*.log {
              weekly
              rotate 4
              compress
              delaycompress
              missingok
              notifempty
              create 0640 nginx adm
              sharedscripts
              postrotate
                  systemctl reload nginx > /dev/null 2>&1 || true
              endscript
          }
        dest: /etc/logrotate.d/nginx
        mode: '0644'
```

---

### Q118. How do you configure Auditd to monitor changes to critical files?

```yaml
- name: Configure audit rules
  hosts: all
  become: true
  tasks:
    - name: Install auditd
      package:
        name: audit
        state: present

    - name: Add audit rules for critical files
      lineinfile:
        path: /etc/audit/rules.d/critical-files.rules
        line: "{{ item }}"
        create: true
      loop:
        - "-w /etc/passwd -p wa -k identity_changes"
        - "-w /etc/group -p wa -k identity_changes"
        - "-w /etc/shadow -p wa -k identity_changes"
      notify: Restart auditd

    - name: Ensure auditd is running
      service:
        name: auditd
        state: started
        enabled: true

  handlers:
    - name: Restart auditd
      service:
        name: auditd
        state: restarted
```

---

### Q119. How do you configure Logwatch for daily email reports?

```yaml
- name: Configure Logwatch
  hosts: all
  become: true
  tasks:
    - name: Install Logwatch
      package:
        name: logwatch
        state: present

    - name: Configure Logwatch
      copy:
        content: |
          MailTo = admin@example.com
          MailFrom = logwatch@{{ ansible_hostname }}
          Detail = Med
          Range = yesterday
          Service = All
        dest: /etc/logwatch/conf/logwatch.conf

    - name: Schedule daily Logwatch report
      cron:
        name: "Daily Logwatch report"
        hour: "6"
        minute: "0"
        job: "/usr/sbin/logwatch --output mail"
```

---

### Q120. How do you configure Monit to monitor services and send email alerts?

```yaml
- name: Configure Monit
  hosts: all
  become: true
  tasks:
    - name: Install Monit
      package:
        name: monit
        state: present

    - name: Configure Monit for httpd
      copy:
        content: |
          check process httpd with pidfile /var/run/httpd/httpd.pid
            start program = "/usr/bin/systemctl start httpd"
            stop program = "/usr/bin/systemctl stop httpd"
            if failed host 127.0.0.1 port 80 then restart
            alert admin@example.com
        dest: /etc/monit.d/httpd

    - name: Configure Monit for mysqld
      copy:
        content: |
          check process mysqld with pidfile /var/run/mysqld/mysqld.pid
            start program = "/usr/bin/systemctl start mysqld"
            stop program = "/usr/bin/systemctl stop mysqld"
            if failed host 127.0.0.1 port 3306 then restart
            alert admin@example.com
        dest: /etc/monit.d/mysqld

    - name: Start Monit
      service:
        name: monit
        state: started
        enabled: true
```

---

### Q121. How do you configure rsnapshot for automated backup with 7-day retention?

```yaml
- name: Configure rsnapshot
  hosts: all
  become: true
  tasks:
    - name: Install rsnapshot
      package:
        name: rsnapshot
        state: present

    - name: Deploy rsnapshot config
      template:
        src: rsnapshot.conf.j2
        dest: /etc/rsnapshot.conf

    - name: Schedule daily backup
      cron:
        name: "rsnapshot daily"
        hour: "3"
        minute: "0"
        job: "/usr/bin/rsnapshot daily"

    - name: Schedule weekly rotation
      cron:
        name: "rsnapshot weekly"
        hour: "2"
        minute: "0"
        weekday: "0"
        job: "/usr/bin/rsnapshot weekly"
```

---

## Section 19: SSH Hardening and Security

### Q122. How do you disable password-based SSH and enforce key-based authentication?

```yaml
- name: Harden SSH
  hosts: all
  become: true
  tasks:
    - name: Disable password authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?PasswordAuthentication"
        line: "PasswordAuthentication no"
      notify: Restart sshd

    - name: Disable root login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?PermitRootLogin"
        line: "PermitRootLogin no"
      notify: Restart sshd

    - name: Ensure PubkeyAuthentication is enabled
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?PubkeyAuthentication"
        line: "PubkeyAuthentication yes"
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted
```

---

### Q123. How do you deploy a company-wide SSH banner?

```yaml
- name: Deploy SSH banner
  hosts: all
  become: true
  tasks:
    - name: Create banner file
      copy:
        content: |
          ************************************************************
          WARNING: Unauthorized access to this system is prohibited.
          All activities are monitored and logged.
          ************************************************************
        dest: /etc/ssh/banner
        mode: '0644'

    - name: Enable banner in sshd_config
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?Banner"
        line: "Banner /etc/ssh/banner"
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted
```

---

### Q124. How do you configure AppArmor profiles for application security?

```yaml
- name: Configure AppArmor
  hosts: all
  become: true
  tasks:
    - name: Install AppArmor utilities
      package:
        name: apparmor-utils
        state: present

    - name: Deploy custom AppArmor profile
      copy:
        content: |
          #include <tunables/global>
          /usr/bin/myapp {
            #include <abstractions/base>
            /etc/myapp/** r,
            /var/log/myapp/** w,
            /tmp/myapp/** rw,
            deny /etc/shadow r,
            deny /home/** rw,
          }
        dest: /etc/apparmor.d/usr.bin.myapp

    - name: Enforce the profile
      command: aa-enforce /usr/bin/myapp
```

---

## Section 20: Networking and Advanced Infrastructure

### Q125. How do you configure network interface bonding using Ansible?

```yaml
- name: Configure network bonding
  hosts: all
  become: true
  tasks:
    - name: Create bond interface
      nmcli:
        type: bond
        conn_name: bond0
        ifname: bond0
        mode: active-backup
        ip4: "{{ bond_ip }}/24"
        gw4: "{{ gateway }}"
        state: present

    - name: Add secondary interface to bond
      nmcli:
        type: bond-slave
        conn_name: "bond0-{{ item }}"
        ifname: "{{ item }}"
        master: bond0
        state: present
      loop:
        - eth0
        - eth1
```

---

### Q126. How do you configure HAProxy as a load balancer for web servers?

```yaml
- name: Configure HAProxy
  hosts: lb
  become: true
  tasks:
    - name: Install HAProxy
      package:
        name: haproxy
        state: present

    - name: Deploy HAProxy config
      template:
        src: haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
      notify: Restart HAProxy

    - name: Start HAProxy
      service:
        name: haproxy
        state: started
        enabled: true

  handlers:
    - name: Restart HAProxy
      service:
        name: haproxy
        state: restarted
```

Template (`haproxy.cfg.j2`):

```jinja2
frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
{% for host in groups['web'] %}
    server {{ hostvars[host].ansible_hostname }} {{ hostvars[host].ansible_default_ipv4.address }}:80 check
{% endfor %}
```

---

### Q127. How do you configure NTP with Chrony for time synchronization?

```yaml
- name: Configure Chrony
  hosts: all
  become: true
  tasks:
    - name: Install Chrony
      package:
        name: chrony
        state: present

    - name: Configure NTP servers
      template:
        src: chrony.conf.j2
        dest: /etc/chrony.conf
      notify: Restart Chrony

    - name: Ensure Chrony is running
      service:
        name: chronyd
        state: started
        enabled: true

  handlers:
    - name: Restart Chrony
      service:
        name: chronyd
        state: restarted
```

---

### Q128. How do you configure an NFS server to share a directory?

```yaml
- name: Configure NFS server
  hosts: nfs_server
  become: true
  tasks:
    - name: Install NFS packages
      package:
        name: nfs-utils
        state: present

    - name: Create shared directory
      file:
        path: /srv/nfs_share
        state: directory
        mode: '0755'

    - name: Configure exports
      lineinfile:
        path: /etc/exports
        line: "/srv/nfs_share 192.168.1.0/24(rw,sync,no_root_squash)"
      notify: Restart NFS

    - name: Start NFS service
      service:
        name: nfs-server
        state: started
        enabled: true

  handlers:
    - name: Restart NFS
      service:
        name: nfs-server
        state: restarted
```

---

### Q129. How do you configure SFTP without SSH access, restricting users to home directories?

```yaml
- name: Configure SFTP
  hosts: all
  become: true
  tasks:
    - name: Create SFTP group
      group:
        name: sftponly
        state: present

    - name: Configure sshd for SFTP
      blockinfile:
        path: /etc/ssh/sshd_config
        block: |
          Match Group sftponly
              ForceCommand internal-sftp
              ChrootDirectory /home/%u
              AllowTcpForwarding no
              X11Forwarding no
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      service:
        name: sshd
        state: restarted
```

---

### Q130. How do you configure an IPsec VPN with StrongSwan between two servers?

```yaml
- name: Configure IPsec VPN
  hosts: vpn_servers
  become: true
  tasks:
    - name: Install StrongSwan
      package:
        name: strongswan
        state: present

    - name: Deploy ipsec.conf
      template:
        src: ipsec.conf.j2
        dest: /etc/strongswan/ipsec.conf
      notify: Restart StrongSwan

    - name: Deploy pre-shared key
      copy:
        content: |
          {{ server1_ip }} {{ server2_ip }} : PSK "{{ vpn_psk }}"
        dest: /etc/strongswan/ipsec.secrets
        mode: '0600'
      notify: Restart StrongSwan

    - name: Ensure StrongSwan starts on boot
      service:
        name: strongswan
        state: started
        enabled: true

  handlers:
    - name: Restart StrongSwan
      service:
        name: strongswan
        state: restarted
```

---

### Q131. How do you configure stunnel to encrypt communication on port 8443?

```yaml
- name: Configure stunnel
  hosts: all
  become: true
  tasks:
    - name: Install stunnel
      package:
        name: stunnel
        state: present

    - name: Deploy stunnel config
      copy:
        content: |
          [myservice]
          accept = 8443
          connect = 127.0.0.1:8080
          cert = /etc/stunnel/stunnel.pem
        dest: /etc/stunnel/stunnel.conf
      notify: Restart stunnel

    - name: Enable and start stunnel
      service:
        name: stunnel
        state: started
        enabled: true

  handlers:
    - name: Restart stunnel
      service:
        name: stunnel
        state: restarted
```

---

### Q132. How do you configure DRBD for real-time data replication between two servers?

```yaml
- name: Configure DRBD
  hosts: drbd_nodes
  become: true
  tasks:
    - name: Install DRBD
      package:
        name:
          - drbd-utils
          - kmod-drbd
        state: present

    - name: Deploy DRBD resource config
      template:
        src: drbd_resource.res.j2
        dest: /etc/drbd.d/data.res
      notify: Restart DRBD

    - name: Initialize DRBD metadata (first run only)
      command: drbdadm create-md data
      args:
        creates: /var/lib/drbd

    - name: Start DRBD
      service:
        name: drbd
        state: started
        enabled: true

  handlers:
    - name: Restart DRBD
      service:
        name: drbd
        state: restarted
```

---

## Section 21: Containers, Virtualization, and Cloud

### Q133. How do you deploy a containerized web application using Podman with auto-restart?

```yaml
- name: Deploy container with Podman
  hosts: all
  become: true
  tasks:
    - name: Install Podman
      package:
        name: podman
        state: present

    - name: Pull web application image
      containers.podman.podman_image:
        name: docker.io/library/nginx
        state: present

    - name: Run web application container
      containers.podman.podman_container:
        name: webapp
        image: docker.io/library/nginx
        state: started
        ports:
          - "80:80"
        restart_policy: always

    - name: Generate systemd service for container
      command: podman generate systemd --name webapp --files --new
      args:
        chdir: /etc/systemd/system

    - name: Enable container service
      systemd:
        name: container-webapp
        enabled: true
        daemon_reload: true
```

---

### Q134. How do you configure persistent storage for a PostgreSQL container with Podman?

```yaml
- name: PostgreSQL with persistent storage
  hosts: db
  become: true
  tasks:
    - name: Create data directory
      file:
        path: /var/lib/pgdata
        state: directory
        mode: '0700'

    - name: Run PostgreSQL container with volume
      containers.podman.podman_container:
        name: postgres
        image: docker.io/library/postgres:15
        state: started
        ports:
          - "5432:5432"
        env:
          POSTGRES_PASSWORD: "{{ db_password }}"
        volumes:
          - /var/lib/pgdata:/var/lib/postgresql/data
        restart_policy: always
```

---

### Q135. How do you automate KVM setup and VM creation using Ansible?

```yaml
- name: Configure KVM
  hosts: hypervisor
  become: true
  tasks:
    - name: Install KVM packages
      package:
        name:
          - qemu-kvm
          - libvirt
          - virt-install
        state: present

    - name: Start libvirtd
      service:
        name: libvirtd
        state: started
        enabled: true

    - name: Create VM
      command: >
        virt-install
        --name testvm
        --ram 8192
        --vcpus 4
        --disk size=100
        --os-variant centos-stream9
        --location /var/lib/libvirt/images/CentOS-Stream-9.iso
        --network bridge=virbr0
        --noautoconsole
      args:
        creates: /etc/libvirt/qemu/testvm.xml
```

---

### Q136. How do you set up a chroot environment for application testing?

```yaml
- name: Create chroot environment
  hosts: all
  become: true
  tasks:
    - name: Create chroot directory structure
      file:
        path: "/chroot/{{ item }}"
        state: directory
      loop:
        - bin
        - lib
        - lib64
        - etc
        - usr/bin

    - name: Copy essential binaries
      copy:
        src: "/bin/{{ item }}"
        dest: "/chroot/bin/{{ item }}"
        mode: '0755'
        remote_src: true
      loop:
        - bash
        - ls
        - cat

    - name: Copy required libraries
      shell: "ldd /bin/bash | grep -o '/lib[^ ]*' | xargs -I{} cp {} /chroot{}"
      args:
        creates: /chroot/lib64/ld-linux-x86-64.so.2
```

---

## Section 22: Database Management

### Q137. How do you write a playbook to install MySQL, set up users, and ensure the service runs?

```yaml
- name: Configure MySQL
  hosts: db
  become: true
  tasks:
    - name: Install MySQL
      package:
        name:
          - mysql-server
          - python3-PyMySQL
        state: present

    - name: Start and enable MySQL
      service:
        name: mysqld
        state: started
        enabled: true

    - name: Create application database
      mysql_db:
        name: appdb
        state: present
        login_unix_socket: /var/lib/mysql/mysql.sock

    - name: Create database user
      mysql_user:
        name: appuser
        password: "{{ db_password }}"
        priv: "appdb.*:ALL"
        state: present
        login_unix_socket: /var/lib/mysql/mysql.sock
```

---

### Q138. How do you use a loop to create multiple databases on managed nodes?

```yaml
- name: Create multiple databases
  hosts: db
  become: true
  tasks:
    - name: Create databases
      mysql_db:
        name: "{{ item.name }}"
        encoding: "{{ item.encoding | default('utf8mb4') }}"
        state: present
      loop:
        - { name: "app_production", encoding: "utf8mb4" }
        - { name: "app_analytics", encoding: "utf8" }
        - { name: "app_logs" }
```

---

### Q139. How do you automate daily MySQL backups with 7-day retention?

```yaml
- name: MySQL backup automation
  hosts: db
  become: true
  tasks:
    - name: Create backup directory
      file:
        path: /backup/mysql
        state: directory
        mode: '0750'

    - name: Deploy backup script
      copy:
        content: |
          #!/bin/bash
          BACKUP_DIR=/backup/mysql
          DATE=$(date +%Y%m%d)
          mysqldump --all-databases | gzip > $BACKUP_DIR/all-databases-$DATE.sql.gz
          # Remove backups older than 7 days
          find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
        dest: /usr/local/bin/mysql_backup.sh
        mode: '0750'

    - name: Schedule daily backup
      cron:
        name: "MySQL daily backup"
        hour: "2"
        minute: "0"
        job: "/usr/local/bin/mysql_backup.sh"
```

---

### Q140. How do you configure a MySQL high-availability cluster with Pacemaker and Corosync?

```yaml
- name: Configure HA MySQL cluster
  hosts: db_cluster
  become: true
  tasks:
    - name: Install cluster packages
      package:
        name:
          - pacemaker
          - corosync
          - pcs
          - mysql-server
        state: present

    - name: Start PCS daemon
      service:
        name: pcsd
        state: started
        enabled: true

    - name: Set cluster password
      user:
        name: hacluster
        password: "{{ cluster_password | password_hash('sha512') }}"

    - name: Authenticate cluster nodes
      command: pcs cluster auth {{ groups['db_cluster'] | join(' ') }} -u hacluster -p {{ cluster_password }}
      run_once: true

    - name: Create cluster
      command: pcs cluster setup --name mysql_cluster {{ groups['db_cluster'] | join(' ') }}
      run_once: true
      args:
        creates: /etc/corosync/corosync.conf

    - name: Start cluster
      command: pcs cluster start --all
      run_once: true
```

---

## Section 23: Web Server Deployment and SSL

### Q141. How do you configure a web server to serve a static website?

```yaml
- name: Deploy static website
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Deploy sample HTML
      copy:
        content: |
          <!DOCTYPE html>
          <html lang="en">
          <head><title>Welcome</title></head>
          <body><h1>Hello from {{ ansible_hostname }}</h1></body>
          </html>
        dest: /usr/share/nginx/html/index.html

    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: true
```

---

### Q142. How do you deploy a secure web server with a self-signed SSL certificate?

```yaml
- name: Deploy HTTPS web server
  hosts: web
  become: true
  tasks:
    - name: Install packages
      package:
        name:
          - httpd
          - mod_ssl
          - openssl
        state: present

    - name: Generate self-signed certificate
      command: >
        openssl req -x509 -nodes -days 365
        -newkey rsa:2048
        -keyout /etc/pki/tls/private/server.key
        -out /etc/pki/tls/certs/server.crt
        -subj "/CN={{ ansible_fqdn }}"
      args:
        creates: /etc/pki/tls/certs/server.crt

    - name: Start Apache
      service:
        name: httpd
        state: started
        enabled: true
```

---

### Q143. How do you deploy Apache with a provided SSL certificate?

```yaml
- name: Deploy Apache with SSL
  hosts: web
  become: true
  tasks:
    - name: Install Apache and mod_ssl
      package:
        name:
          - httpd
          - mod_ssl
        state: present

    - name: Deploy SSL certificate
      copy:
        src: "certs/{{ ansible_hostname }}.crt"
        dest: /etc/pki/tls/certs/server.crt
        mode: '0644'

    - name: Deploy SSL key
      copy:
        src: "certs/{{ ansible_hostname }}.key"
        dest: /etc/pki/tls/private/server.key
        mode: '0600'

    - name: Deploy SSL config
      template:
        src: ssl.conf.j2
        dest: /etc/httpd/conf.d/ssl.conf
      notify: Restart Apache

    - name: Start Apache
      service:
        name: httpd
        state: started
        enabled: true

  handlers:
    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

---

### Q144. How do you deploy SSL certificates per server using variables?

Define per-host variables in `host_vars/webserver1.yml`:

```yaml
ssl_cert_src: certs/webserver1.crt
ssl_key_src: certs/webserver1.key
```

Playbook:

```yaml
- name: Deploy per-server SSL certs
  hosts: web
  become: true
  tasks:
    - name: Deploy certificate
      copy:
        src: "{{ ssl_cert_src }}"
        dest: /etc/pki/tls/certs/server.crt
        mode: '0644'

    - name: Deploy key
      copy:
        src: "{{ ssl_key_src }}"
        dest: /etc/pki/tls/private/server.key
        mode: '0600'
      notify: Restart Apache

  handlers:
    - name: Restart Apache
      service:
        name: httpd
        state: restarted
```

---

## Section 24: CI/CD, Multi-Tier Deployments, and Advanced Playbooks

### Q145. How do you organize a playbook for a multi-tier application deployment?

```yaml
# Deploy database tier first
- name: Configure database servers
  hosts: db
  become: true
  roles:
    - database

# Then application tier
- name: Configure application servers
  hosts: app
  become: true
  roles:
    - app_server

# Then web/load balancer tier
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - web_server

- name: Configure load balancer
  hosts: lb
  become: true
  roles:
    - load_balancer
```

---

### Q146. How do you create a playbook that integrates roles and custom tasks for CI/CD setup?

```yaml
- name: Set up CI/CD pipeline
  hosts: cicd
  become: true
  pre_tasks:
    - name: Update package cache
      package:
        name: "*"
        state: latest

  roles:
    - role: geerlingguy.java
    - role: geerlingguy.jenkins

  post_tasks:
    - name: Install additional Jenkins plugins
      jenkins_plugin:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - pipeline
        - docker-workflow

    - name: Deploy custom Jenkins config
      template:
        src: jenkins_config.xml.j2
        dest: /var/lib/jenkins/config.xml
      notify: Restart Jenkins

  handlers:
    - name: Restart Jenkins
      service:
        name: jenkins
        state: restarted
```

---

### Q147. How do you configure Nagios monitoring for servers using Ansible?

```yaml
- name: Install and configure Nagios
  hosts: monitoring
  become: true
  tasks:
    - name: Install Nagios
      package:
        name:
          - nagios
          - nagios-plugins-all
        state: present

    - name: Deploy host configuration
      template:
        src: nagios_host.cfg.j2
        dest: "/etc/nagios/conf.d/{{ item }}.cfg"
      loop: "{{ groups['all'] }}"
      notify: Restart Nagios

    - name: Start Nagios
      service:
        name: nagios
        state: started
        enabled: true

  handlers:
    - name: Restart Nagios
      service:
        name: nagios
        state: restarted
```

---

### Q148. How do you configure Kerberos authentication using Ansible?

```yaml
- name: Configure Kerberos client
  hosts: all
  become: true
  tasks:
    - name: Install Kerberos packages
      package:
        name:
          - krb5-workstation
          - krb5-libs
          - pam_krb5
        state: present

    - name: Deploy krb5.conf
      template:
        src: krb5.conf.j2
        dest: /etc/krb5.conf

    - name: Configure PAM for Kerberos
      command: authconfig --enablekrb5 --krb5realm={{ krb_realm }} --krb5kdc={{ krb_kdc }} --update
```

---

### Q149. How do you monitor disk I/O performance to identify hardware upgrade needs?

```yaml
- name: Monitor disk I/O
  hosts: all
  become: true
  tasks:
    - name: Install sysstat
      package:
        name: sysstat
        state: present

    - name: Collect I/O stats
      shell: iostat -dx 1 5 | tail -20
      register: io_stats
      changed_when: false

    - name: Check for high I/O wait
      shell: iostat -c 1 3 | awk '/^ /{print $4}' | tail -1
      register: iowait
      changed_when: false

    - name: Alert on high I/O wait
      debug:
        msg: "HIGH I/O WAIT ({{ iowait.stdout }}%) on {{ ansible_hostname }} — consider hardware upgrade"
      when: iowait.stdout | float > 20
```

---

### Q150. How do you limit memory usage of a specific service using Ansible?

```yaml
- name: Limit service memory
  hosts: all
  become: true
  tasks:
    - name: Create systemd override directory
      file:
        path: /etc/systemd/system/myservice.service.d
        state: directory

    - name: Set memory limit
      copy:
        content: |
          [Service]
          MemoryMax=512M
          MemoryHigh=400M
        dest: /etc/systemd/system/myservice.service.d/memory-limit.conf
      notify:
        - Reload systemd
        - Restart service

  handlers:
    - name: Reload systemd
      systemd:
        daemon_reload: true

    - name: Restart service
      service:
        name: myservice
        state: restarted
```

---

### Q151. How do you manage hybrid cloud infrastructure using multiple Content Collections?

```yaml
# requirements.yml
collections:
  - name: amazon.aws
    version: ">=5.0.0"
  - name: azure.azcollection
    version: ">=1.0.0"
  - name: community.general
```

Playbook:

```yaml
- name: Manage AWS resources
  hosts: localhost
  tasks:
    - name: Create AWS EC2 instance
      amazon.aws.ec2_instance:
        name: web-server
        instance_type: t3.micro
        image_id: ami-12345678
        state: running

- name: Manage Azure resources
  hosts: localhost
  tasks:
    - name: Create Azure VM
      azure.azcollection.azure_rm_virtualmachine:
        resource_group: myResourceGroup
        name: web-server
        vm_size: Standard_B1s
```

---

### Q152. How do you ensure role version control and consistency across team projects?

Use a shared `requirements.yml` in version control:

```yaml
roles:
  - name: company.database
    src: git+https://git.internal.example.com/roles/database.git
    version: v2.1.0

  - name: company.webserver
    src: git+https://git.internal.example.com/roles/webserver.git
    version: v1.5.3
```

Enforce with CI/CD:

```bash
# In CI pipeline
ansible-galaxy install -r requirements.yml --force
ansible-lint playbook.yml
```

Pin versions explicitly and update through pull requests to maintain consistency.

---

### Q153. How do you configure a network role that applies settings based on inventory variables?

Role `roles/network_config/tasks/main.yml`:

```yaml
---
- name: Configure network interface
  template:
    src: ifcfg.j2
    dest: "/etc/sysconfig/network-scripts/ifcfg-{{ network_interface }}"
  notify: Restart NetworkManager
```

Inventory:

```ini
[web]
web1 network_interface=eth0 ip_address=10.0.1.10 gateway=10.0.1.1
web2 network_interface=eth0 ip_address=10.0.1.11 gateway=10.0.1.1

[db]
db1 network_interface=eth1 ip_address=10.0.2.10 gateway=10.0.2.1
```

---

### Q154. How do you apply a Content Collection's roles to specific inventory groups (e.g., routers vs. switches)?

```yaml
- name: Configure routers
  hosts: routers
  tasks:
    - name: Apply router configuration
      include_role:
        name: cisco.ios.ios_config
      vars:
        config_lines:
          - ip route 0.0.0.0 0.0.0.0 10.0.0.1

- name: Configure switches
  hosts: switches
  tasks:
    - name: Apply switch configuration
      include_role:
        name: cisco.ios.ios_vlans
      vars:
        vlans:
          - vlan_id: 100
            name: management
```

---

### Q155. How do you manage Kubernetes clusters using an Ansible Content Collection?

```bash
ansible-galaxy collection install kubernetes.core
```

Playbook:

```yaml
- name: Manage Kubernetes
  hosts: localhost
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: myapp

    - name: Deploy application
      kubernetes.core.k8s:
        state: present
        src: deployment.yml
        namespace: myapp
```

---

### Q156. How do you write a playbook that retrieves uptime and performs maintenance on nodes with uptime > 30 days?

```yaml
- name: Uptime-based maintenance
  hosts: all
  tasks:
    - name: Get uptime in days
      shell: awk '{print int($1/86400)}' /proc/uptime
      register: uptime_days
      changed_when: false

    - name: Display uptime
      debug:
        msg: "{{ ansible_hostname }} has been up for {{ uptime_days.stdout }} days"

    - name: Schedule reboot for long-running nodes
      command: shutdown -r +5 "Scheduled maintenance reboot"
      when: uptime_days.stdout | int > 30
      become: true
```

---

This concludes the organized Q&A covering all unique Ansible concepts from the original question set, arranged from foundational to advanced topics.
