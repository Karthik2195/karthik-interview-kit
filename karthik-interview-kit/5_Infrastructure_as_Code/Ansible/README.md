# Ansible - Configuration Management & Automation

## What is it used for?
Ansible is used for:
- **Configuration management**: Manage server configurations at scale
- **Application deployment**: Deploy applications to multiple servers
- **Infrastructure provisioning**: Automate infrastructure setup
- **Orchestration**: Coordinate complex multi-tier deployments
- **Compliance management**: Ensure systems meet compliance requirements
- **Disaster recovery**: Automate recovery procedures
- **Agentless management**: No agents to install on managed nodes

## Installation

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install ansible

# RHEL/CentOS
sudo yum install ansible

# macOS
brew install ansible

# Python pip
pip install ansible

# Verify installation
ansible --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Ansible version
ansible --version

# List inventory hosts
ansible-inventory --list

# Ping all hosts
ansible all -m ping

# Run ad-hoc command on all hosts
ansible all -m command -a "uptime"

# Run on specific host
ansible hostname -m command -a "date"

# Run as specific user
ansible all -u username -m command -a "whoami"

# Run with become (sudo)
ansible all -m command -a "id" --become

# Copy file
ansible all -m copy -a "src=/local/file dest=/remote/path"

# Install package
ansible all -m package -a "name=nginx state=present"

# Start service
ansible all -m service -a "name=nginx state=started enabled=yes"

# Run playbook
ansible-playbook playbook.yml

# Run with specific inventory
ansible-playbook -i inventory.ini playbook.yml

# Run with verbose output
ansible-playbook -v playbook.yml
ansible-playbook -vvv playbook.yml  # Very verbose

# Check syntax
ansible-playbook --syntax-check playbook.yml

# Dry-run (check mode)
ansible-playbook --check playbook.yml

# List tasks that would run
ansible-playbook --list-tasks playbook.yml

# List hosts
ansible-playbook --list-hosts playbook.yml

# Debug variables
ansible all -m debug -a "var=ansible_os_family"
```

### Inventory Configuration
```ini
# inventory.ini
[webservers]
web1.example.com
web2.example.com

[databases]
db1.example.com
db2.example.com

[webservers:vars]
ansible_user=ubuntu
ansible_private_key_file=~/.ssh/id_rsa

[databases:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

### Basic Playbook Example
```yaml
# playbook.yml
---
- name: Install and start Nginx
  hosts: webservers
  become: true
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Create web page
      copy:
        content: "Hello from {{ inventory_hostname }}"
        dest: /var/www/html/index.html

    - name: Verify Nginx is running
      uri:
        url: http://{{ inventory_hostname }}
        status_code: 200
      register: result

    - name: Print result
      debug:
        var: result
```

### Common Issues & Resolution

**Issue: SSH key permission denied**
```bash
# Solution: Set correct permissions
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh

# Check SSH connection
ssh -v user@hostname

# Add SSH key to agent
ssh-add ~/.ssh/id_rsa

# Run ansible with verbose SSH output
ansible all -m ping -vvv
```

**Issue: Host key verification failed**
```bash
# Solution: Disable host key checking (not recommended)
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook playbook.yml

# Or add to ansible.cfg
[defaults]
host_key_checking = False

# Better: Pre-add keys to known_hosts
ssh-keyscan -H hostname >> ~/.ssh/known_hosts
```

**Issue: Module not found**
```bash
# Solution: Install required module
ansible-galaxy collection install community.general

# Or use built-in modules
ansible all -m setup  # Gather facts

# List available modules
ansible-doc -l
```

**Issue: Connection timeout**
```bash
# Solution: Check host is reachable
ping hostname

# Increase connection timeout in ansible.cfg
[defaults]
timeout = 30

# Or pass as option
ansible all -m ping -c paramiko --timeout=30
```

**Issue: Privilege escalation failed**
```bash
# Solution: Ensure user can sudo without password
ansible all -m command -a "sudo -l" -u username

# Configure sudoers for passwordless sudo
# On target: sudo visudo
# Add: username ALL=(ALL) NOPASSWD: ALL

# Use correct become method
ansible-playbook playbook.yml --become-method=sudo
```

### Debugging
```bash
# Run with increased verbosity
ansible-playbook -vvv playbook.yml

# Enable debug mode
export ANSIBLE_DEBUG=1

# Check facts gathered
ansible hostname -m setup

# Show variable values
ansible all -m debug -a "var=ansible_distribution"

# Check playbook syntax
ansible-playbook --syntax-check playbook.yml

# List tasks without running
ansible-playbook --list-tasks playbook.yml

# Run specific task by name
ansible-playbook playbook.yml --start-at-task="Task Name"

# Run specific task by tag
ansible-playbook playbook.yml -t tag-name
```

### Working with Variables
```yaml
# Variables in playbook
---
- hosts: all
  vars:
    app_name: myapp
    app_port: 8080
  
  tasks:
    - name: Display variables
      debug:
        msg: "App {{ app_name }} runs on port {{ app_port }}"

# Variables from file
---
- hosts: all
  vars_files:
    - vars.yml
  
  tasks:
    - debug:
        var: variable_from_file

# Prompt for variables
---
- hosts: all
  vars_prompt:
    - name: username
      prompt: "Enter username"
```

### Roles
```bash
# Create role structure
ansible-galaxy init my-role

# Use role in playbook
---
- hosts: all
  roles:
    - my-role

# Install role from galaxy
ansible-galaxy install geerlingguy.nginx

# List installed roles
ansible-galaxy list
```
