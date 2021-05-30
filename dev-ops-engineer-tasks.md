# DevOps Engineer Tasks - Set 1

- [Task 1 - Puppet Add Users](#puppet-add-users)

## Puppet Add Users

As a new teammate has joined Nautilus application development team, the application development team has asked the DevOps team to create a new user for the new teammate on all application servers in Stratos Datacenter. The task needs to be performed using Puppet only. You can find more details below about the task.

Create a Puppet programming file **ecommerce.pp** under **/etc/puppetlabs/code/environments/production/manifests** directory on **master** node i.e **Jump Server**, and using Puppet **user** resource add a user on all app servers as mentioned below:

  1. Create a user **anita** and set its UID to **1157** on all Puppet agent nodes i.e all **App Servers**.

**Note**: Please perform this task using **ecommerce.pp** only, do not create any separate inventory file.

### Solution

```bash
# Following tasks has to be performed on master or Jump Server

cd /etc/puppetlabs/code/environments/production/manifests

vi ecommerce.pp

# Add following to the ecommerce.pp file
class user_creator {
    user { 'anita':
      ensure => present,
      uid => 1157
    }
}

node 'stapp01.stratos.xfusioncorp.com', 'stapp02.stratos.xfusioncorp.com', 'stapp03.stratos.xfusioncorp.com' {
    include user_creator
}

# Save the file and exit from vi editor

# Check if puppet file is valid
puppet parser validate ecommerce.pp

# You will not see any messages if the file is valid.

# Following tasks has to be performed on each app servers
ssh tony@stapp01
ssh steve@stapp02           # New Terminal
ssh banner@stapp03          # New Terminal

sudo puppet agent -tv

# Verify if user created successfully
cat /etc/passwd | grep anita
```