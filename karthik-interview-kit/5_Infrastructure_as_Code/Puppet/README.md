# Puppet - Configuration Management & Infrastructure Automation

## What is it used for?
Puppet is used for:
- **Configuration management**: Manage system configurations
- **Infrastructure automation**: Automate infrastructure provisioning
- **Compliance**: Enforce compliance policies
- **Multi-platform support**: Manage Windows, Linux, macOS
- **Scalability**: Manage thousands of nodes
- **Version control**: Track configuration changes
- **Idempotent operations**: Reliable repeated operations
- **Reporting**: Detailed change reporting

## Installation

```bash
# Install Puppet Agent (Ubuntu/Debian)
wget https://apt.puppetlabs.com/puppet-release-focal.deb
sudo dpkg -i puppet-release-focal.deb
sudo apt update
sudo apt install puppet-agent

# Install Puppet Server
sudo apt install puppetserver

# Start Puppet services
sudo systemctl start puppet
sudo systemctl start puppetserver

# Verify installation
puppet --version
puppet agent --version
```

## Basic Commands & Troubleshooting

### Essential Commands
```bash
# Check Puppet version
puppet --version

# Validate Puppet code
puppet parser validate manifest.pp

# Run Puppet agent
sudo puppet agent --test

# Run with debug
sudo puppet agent --test --debug

# Disable Puppet agent
sudo puppet agent --disable "Reason"

# Enable Puppet agent
sudo puppet agent --enable

# Puppet server status
sudo systemctl status puppetserver

# Test agent connectivity
puppet agent --test --noop

# Check certificates
puppet cert list

# Sign certificate
sudo puppet cert sign agent-name

# View agent log
tail -f /var/log/puppet/agent.log
```

### Basic Manifest
```puppet
# site.pp
class { 'apache':
  default_mods => true,
  mods         => ['ssl', 'rewrite'],
}

apache::vhost { 'example.com':
  port    => 80,
  docroot => '/var/www/example.com',
}

package { 'nginx':
  ensure => present,
}

service { 'nginx':
  ensure  => running,
  enable  => true,
  require => Package['nginx'],
}

file { '/etc/config.txt':
  ensure  => present,
  content => template('module/config.erb'),
  mode    => '0644',
}
```

### Module Management
```bash
# Create module
puppet module generate my-namespace/my-module

# Generate class
puppet generate module my-namespace/my-module

# Install module from Puppet Forge
puppet module install puppetlabs-apache

# List installed modules
puppet module list

# Uninstall module
puppet module uninstall puppetlabs-apache

# Search Puppet Forge
puppet module search apache
```

### Common Issues & Resolution

**Issue: Agent fails to connect to server**
```bash
# Solution: Check server status
sudo systemctl status puppetserver

# Check server logs
sudo tail -f /var/log/puppetlabs/puppetserver/puppetserver.log

# Test connectivity
telnet puppet-server 8140

# Clean certificates
sudo puppet cert clean agent-name
sudo rm -rf /etc/puppetlabs/puppet/ssl
sudo puppet agent --test
```

**Issue: Certificate issues**
```bash
# Solution: Check certificate status
sudo puppet cert list

# Sign pending certificates
sudo puppet cert sign agent-name

# Sign all certificates
sudo puppet cert sign --all

# Revoke certificate
sudo puppet cert revoke agent-name
```

**Issue: Manifest syntax errors**
```bash
# Solution: Validate manifest
puppet parser validate manifest.pp

# Check specific syntax
puppet apply manifest.pp --noop --debug

# Use linter
puppet-lint manifest.pp
```

### Debugging
```bash
# Run Puppet with debug output
sudo puppet agent --test --debug

# Dry-run (no changes)
sudo puppet agent --test --noop

# Verbose output
sudo puppet agent --test --verbose

# Show differences
sudo puppet agent --test --show_diff

# View logs
tail -f /var/log/puppetlabs/puppet/agent.log

# Debug module issues
puppet apply manifest.pp --debug
```

### Puppet Server Configuration
```bash
# Start Puppet server
sudo systemctl start puppetserver

# Enable at boot
sudo systemctl enable puppetserver

# Check server logs
sudo journalctl -u puppetserver -f

# View server configuration
cat /etc/puppetlabs/puppet/puppet.conf

# Reconfigure server
sudo puppetserver ca setup
```

### Classes and Resources
```puppet
# Define a class
class my_class (
  String $package_name = 'nginx',
  Integer $port = 80,
) {
  package { $package_name:
    ensure => present,
  }

  service { $package_name:
    ensure  => running,
    enable  => true,
    require => Package[$package_name],
  }
}

# Use the class
include my_class

# Use with parameters
class { 'my_class':
  package_name => 'apache2',
  port         => 8080,
}
```

### Data Sources and Hiera
```bash
# View Hiera data
puppet lookup package_name

# Hiera configuration
# /etc/puppetlabs/puppet/hiera.yaml
---
version: 5
hierarchy:
  - name: "Nodes"
    path: "nodes/%{trusted.certname}.yaml"
  - name: "Common"
    path: "common.yaml"
defaults:
  data_hash: yaml_data
  datadir: /etc/puppetlabs/code/environments/production/hieradata
```

### Reporting
```bash
# View Puppet reports
puppet report list

# View specific report
puppet report show agent-hostname

# Puppet metrics
sudo curl http://localhost:8140/metrics/v1/jvm

# Generate report
sudo puppet agent --test --report
```

### Advanced Features
```bash
# Define custom resource types
define apache::vhost (
  Integer $port = 80,
  String $docroot = '/var/www/default',
) {
  ...
}

# Exported resources
@@service { 'myservice':
  ensure => running,
}

# Collecting exported resources
Service <<| tag == 'webserver' |>>

# Custom functions
Puppet::Functions.create_function(:'my_function') do
  dispatch :invoke do
    param 'String', :message
  end
  def invoke(message)
    puts message
  end
end
```

### Compliance and Auditing
```bash
# Enforce compliance
class profile::compliance {
  include profile::security
  include profile::firewall
  include profile::updates
}

# Audit compliance
puppet resource file /etc/passwd

# Compliance reporting
puppet module install forward/compliance_report
```
