# Chef - Configuration Management & Infrastructure Automation

## What is it used for?
Chef is used for:
- **Infrastructure automation**: Automate infrastructure setup
- **Configuration management**: Manage system configurations
- **Compliance enforcement**: Ensure compliance
- **Continuous deployment**: Automate deployments
- **Multi-cloud support**: Manage across clouds
- **Version control integration**: Track changes
- **Testing framework**: Test configurations
- **Scaling**: Manage large infrastructure

## Installation

```bash
# Install ChefDK (Chef Development Kit)
# macOS
brew install chef-workstation

# Linux
wget https://packages.chef.io/files/stable/chef-workstation/23.12.1019/ubuntu/20.04/chef-workstation_23.12.1019-1_amd64.deb
sudo dpkg -i chef-workstation_23.12.1019-1_amd64.deb

# Windows
# Download MSI from https://downloads.chef.io/

# Verify installation
chef --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Chef version
chef --version

# Create cookbook
chef generate cookbook my-cookbook

# Generate recipe
chef generate recipe my-cookbook::default

# Test cookbook
chef exec rspec

# Run kitchen tests
kitchen test

# Apply recipe locally
sudo chef-client -z -r "recipe[my-cookbook::default]"

# Converge nodes
knife ssh "name:*" "sudo chef-client"

# Search for nodes
knife search "name:*"

# Node list
knife node list

# Get node details
knife node show node-name

# Set node attributes
knife node run_list add node-name "recipe[my-cookbook::default]"

# Validate cookbook
cookstyle my-cookbook

# Upload cookbook
knife cookbook upload my-cookbook

# Create data bag
knife data bag create users

# Add item to data bag
knife data bag item create users alice

# List data bags
knife data bag list
```

### Basic Cookbook Structure
```ruby
# recipes/default.rb
package 'nginx' do
  action :install
end

service 'nginx' do
  action [:enable, :start]
end

template '/etc/nginx/nginx.conf' do
  source 'nginx.conf.erb'
  variables(
    worker_processes: node['nginx']['worker_processes']
  )
  notifies :restart, 'service[nginx]'
end

# attributes/default.rb
default['nginx']['worker_processes'] = 4
default['nginx']['port'] = 80

# templates/nginx.conf.erb
worker_processes <%= @worker_processes %>;
```

### Test Kitchen
```bash
# Initialize kitchen
kitchen init

# List instances
kitchen list

# Create test instance
kitchen create

# Converge instance
kitchen converge

# Run tests
kitchen test

# Destroy instances
kitchen destroy

# Login to instance
kitchen login
```

### Common Issues & Resolution

**Issue: Recipe fails to apply**
```bash
# Solution: Run with debug
sudo chef-client -z -r "recipe[cookbook::recipe]" -l debug

# Check cookbook syntax
cookstyle cookbook-name

# Verify all attributes are set
chef-shell -z

# Check dependencies
berks install
```

**Issue: Node cannot connect to Chef Server**
```bash
# Solution: Verify credentials
ls -la ~/.chef

# Check node key
knife client list | grep node-name

# Re-register node
sudo rm -rf /etc/chef
sudo chef-client

# Check server connection
knife status
```

**Issue: Template rendering fails**
```bash
# Solution: Validate template syntax
erb -x -T 2 templates/my-template.erb

# Check variable availability
node.debug_value 'attribute-name'

# Use inline template for testing
template '/etc/config' do
  source 'template.erb'
  variables(user: 'admin')
end
```

### Debugging
```bash
# Enable debug logging
sudo chef-client -l debug

# Dry-run
sudo chef-client -z --why-run

# List resources
sudo chef-client -z -o recipe[cookbook::recipe] -l debug

# Show attributes
knife node run_list node-name

# Get computed attributes
knife node show -r node-name

# Check remote execution
ssh user@node "sudo chef-client"
```

### Chef Server
```bash
# Install Chef Server
sudo rpm -Uvh https://packages.chef.io/files/stable/chef-server/15.3.0/el/7/chef-server-core-15.3.0-1.el7.x86_64.rpm

# Configure Chef Server
sudo chef-server-ctl reconfigure

# Get admin key
sudo scp /root/.chef/admin.pem user@workstation:~/.chef/

# Connect workstation to server
knife configure

# Verify connection
knife status
```

### Data Bags
```bash
# Create encrypted data bag
knife data bag create --secret-file ~/secret.key users

# Add encrypted item
knife data bag item create users admin --secret-file ~/secret.key

# Read encrypted data in recipe
users = data_bag_item('users', 'admin')
password = users['password']
```

### Node Management
```bash
# List all nodes
knife node list

# Search nodes
knife search "platform:ubuntu"

# Update run list
knife node run_list add node-name "recipe[cookbook::recipe]"

# Remove from run list
knife node run_list remove node-name "recipe[cookbook::recipe]"

# Trigger chef-client run
knife ssh "name:node-name" "sudo chef-client"

# Watch chef-client run
watch -n 5 'tail /var/log/chef/client.log'
```

### Compliance
```bash
# Create compliance profile
chef generate profile my-profile

# Run compliance tests
inspec exec my-profile

# Generate report
inspec exec my-profile --reporter cli html:report.html
```

### Advanced Features
```bash
# Search across cluster
knife search "run_list:recipe*"

# Bulk edit
knife node bulk delete "name:old*"

# Policy locks
# Use Policyfiles for version control

# Cookbook versioning
# Maintain versions in metadata.rb
```
