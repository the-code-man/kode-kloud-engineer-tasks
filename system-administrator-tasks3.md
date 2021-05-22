# System Administrator Tasks - Set 3

- [Task 34 - PAM Authentication For Apache](#pam-authentication-for-apache)
- [Task 35 - Install and Configure NFS Server](#install-and-configure-nfs-server)
- [Task 36 - Linux Process Troubleshooting](#linux-process-troubleshooting)
- [Task 37 - IPtables Installation And Configuration](#iptables-installation-and-configuration)

## PAM Authentication For Apache

The document root **/var/www/html** of all web apps is on NFS share **/data** on storage server in **Stratos Datacenter**. We have a requirement where we want to password protect a directory in the Apache web server document root. We want to password protect **http://<website-url>:<apache_port>/protected** URL as per the following requirements (you can use any **website-url** for it like localhost since there are no such specific requirements as of now):

a. We want to use basic authentication.

b. We do not want to use **htpasswd** file base authentication. Instead, we want to use **PAM** authentication, i.e **Basic Auth + PAM** so that we can authenticate with a Linux user.

c. We already have a user **john** with password **ksH85UJjhb** which you need to provide access to.

d. You can access the website on LBR link. To do so click on the + button on top of your terminal, select **Select port to view on Host 1**, and after adding port **80** click on **Display Port**.

### Solution

```bash
# SSH into App Server 1
ssh tony@stapp01

# Install the pwauth authenticator and mod_authnz_external tool
# https://code.google.com/archive/p/pwauth/
# https://code.google.com/archive/p/mod-auth-external/
yum --enablerepo=epel -y install mod_authnz_external pwauth

# Edit the configuration
vi /etc/httpd/conf.d/authnz_external.conf

# Add following to the end of the config
<Directory /var/www/html/protected> 
    AuthType Basic
    AuthName "PAM Authentication"
    AuthBasicProvider external
    AuthExternal pwauth
    require valid-user
</Directory>

# Create the protected directory
mkdir -p /var/www/html/protected

# Verify by executing the following command
cat /var/www/html/protected/index.html

# This should return a protected directory message

# Start apache service
systemctl enable httpd && systemctl start httpd && systemctl status httpd

###### DO THE SAME IN App Server 2 and App Server 3 as well

# verify from jump server
curl -u john:ksH85UJjhb http://stlb01:80/protected
curl -u john:ksH85UJjhb http://stapp01:8080/protected
curl -u john:ksH85UJjhb http://stapp02:8080/protected
curl -u john:ksH85UJjhb http://stapp03:8080/protected

# All should work while without user name password you should see an error message
```

## Install and Configure NFS Server

For our infrastructure in Stratos Datacenter, we need to serve our website code from a common/shared location that can be shared among all app nodes. To solve this, we came up with a solution to use the NFS (Network File System) server where we can store the data and mount the share among our app nodes. The dedicated NFS server will be our storage server. To accomplish this task, perform the following steps:

1. Install required NFS packages on storage server.

2. Configure storage server to act as an NFS server.

3. Make a NFS share **/nfsdata** on storage server.

4. Install and configure NFS client packages on all app nodes and configure them to act as NFS client.

5. Mount **/nfsdata** directory on all app nodes at **/var/www/nfsdata** directory (Create the directories if do not exist).

6. Start and enable required services.

7. There is an **/tmp/index.html** file on **jump host**. Copy this file on NFS server (storage server) under **/nfsdata** and make sure it gets replicated to all app servers in mounted location.	

### Solution

```bash
# The task requires us to install nfs server on both Storage and all app servers

# SSH into Storage Server
ssh natasha@ststor01

# Switch to root user
sudo su -

# Install NFS packages
yum install -y nfs-utils nfs-utils-lib

# Make entries for each app server in the /etc/exports
vi /etc/exports

# Mounting/shared directory has to match with the one given in the problem statement 
/nfsdata 172.16.238.10(rw,sync,no_subtree_check,no_root_squash,fsid=0)		
/nfsdata 172.16.238.11(rw,sync,no_subtree_check,no_root_squash,fsid=0)		
/nfsdata 172.16.238.12(rw,sync,no_subtree_check,no_root_squash,fsid=0)

#############################################################################################
# ro: With the help of this option we can provide read only access to the shared files      #
# i.e client will only be able to read.                                                     # 
#                                                                                           #
# rw: This option allows the client server to both read and write access within the shared  #
# directory.                                                                                #
#                                                                                           #    
# sync: Sync confirms requests to the shared directory only once the changes have been      #    
# committed.                                                                                #    
#                                                                                           #
# no_subtree_check: This option prevents the subtree checking. When a shared directory is   #
# the subdirectory of a larger file system, nfs performs scans of every directory above it, #
# in order to verify its permissions and details. Disabling the subtree check may increase  #
# the reliability of NFS, but reduce security.                                              #
#                                                                                           #
# no_root_squash: This phrase allows root to connect to the designated directory.           #
#############################################################################################

# Start nfs service
systemctl enable nfs-server && systemctl start nfs-server && systemctl status nfs-server

# To ensure that file system shared directory will reflect on the Appservers, make sure to export file system (exportfs)
exportfs -ra

#############################################################################
# exportfs -v : Displays a list of shares files and options on a server     #
# exportfs -a : Exports all shares listed in /etc/exports, or given name    #
# exportfs -u : Unexports all shares listed in /etc/exports, or given name  #
# exportfs -r : Refresh the server’s list after modifying /etc/exports      #
#############################################################################

# Print and check the list of exported filesystems.
showmount -e ststor01

# Verify openssh-clients is installed in both the Jump server and the Storage Server. Install if it's not already installed.
yum install -y openssh-clients

# Check contents of shared folder so that later we can verify if sharing worked or not.
ll /nfsdata

# From jump server, copy the /tmp/index.html to a temporary folder (/tmp) in Storage Server and move the file to target directory
sudo scp -r /tmp/index.html natasha@ststor01:/tmp
mv /tmp/index.html /nfsdata

### Enable all app servers to act as NFS clients

# SSH to all app servers
ssh tony@stapp01

# Switch to root user 
sudo su -

# Check and create the shared directory on web server
mkdir -p /var/www/web

# Install NFS packages
yum install -y nfs-utils nfs-utils-lib

# Start nfs-server
systemctl enable nfs-server && systemctl start nfs-server && systemctl status nfs-server

# Mount the shared directory into the App server
mount -t nfs ststor01:/nfsdata /var/www/nfsdata

# verify if the mount was successful
df -h

# Once the steps are completed on all app servers verify if index.html is available to all
ll /var/www/nfsdata
```

## Linux Process Troubleshooting

The production support team of xFusionCorp Industries has deployed some of the latest monitoring tools to keep an eye on every service, application, etc. running on the systems. One of the monitoring systems reported about Apache service unavailability on one of the app servers in **Stratos DC**.

Identify the faulty app host and fix the issue. Make sure Apache service is up and running on all app hosts. Also, do not try to change the Apache port on any host.

### Solution

```bash
# SSH into all app servers.
ssh tony@stapp01
ssh steve@stapp01
ssh banner@stapp01

# Switch to root user
sudo su -

# Check status of httpd service
systemctl status httpd

# If service is stopped then start it
systemctl start httpd && systemctl status httpd

## On one of the app server, the apache service will not start because the listening port is already bound to another service.
May 19 15:21:30 stapp01.stratos.xfusioncorp.com httpd[377]: (98)Address already in use: AH00072: make_sock: could not bind to address 0.0.0.0:8089

# Since we cannot change the port on which apache is listening. We now need to check which other service is listening on that port.
# For this we can make use of netstat command. Remember the PID will only be visible if you have root privilages
netstat -tp         # ref - https://www.tecmint.com/20-netstat-commands-for-linux-network-management/

# This will display a list of services along with their PID
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name    
...
tcp        0      0 127.0.0.1:8089          0.0.0.0:*               LISTEN      0          44280      335/sendmail: accep 
...

# Shutdown the service
kill -9 335

## The kill -9 command sends a SIGKILL signal indicating to a service to shut down immediately. An unresponsive program will
## ignore a kill command, but it will shut down whenever a kill -9 command is issued. Use this command with caution. It
## bypasses the standard shutdown routine so any unsaved data will be lost.

# Ref - https://phoenixnap.com/kb/how-to-kill-a-process-in-linux

# Start the apache service. It should start now
systemctl start httpd && systemctl status httpd
```

## IPtables Installation And Configuration

We have one of our websites up and running on our **Nautilus** infrastructure in **Stratos DC**. Our security team has raised a concern that right now Apache’s port i.e **8084** is open for all since there is no firewall installed on these hosts. So we have decided to add some security layer for these hosts and after discussions and recommendations we have come up with the following requirements:

Install **iptables** and all its dependencies on each app host.

Block incoming port **8084** on all apps for everyone except for LBR host.

Make sure the rules remain, even after system reboot.

### Solution

```bash
# First check from jmp and LBR server, if you are able to connect to the app servers on the said node
curl -I stapp01:8084
curl -I stapp02:8084
curl -I stapp03:8084

# You will get response on both servers (forbidden is also a response).
# We need to prevent access from everywhere except LBR server.

# SSH into all app servers and switch to root user
ssh tony@stapp01
sudo su -

ssh steve@stapp01
sudo su -

ssh banner@stapp01
sudo su -

## Perform the steps on all app servers

# Install iptables service
yum install -y iptables-services

# Add the following rules to ip tables
## We are allowing all traffic to port 
iptables -A INPUT -p tcp --destination-port 8084 -s 172.16.238.14 -j ACCEPT
iptables -A INPUT -p tcp --destination-port 8084 -j DROP

# Make the rules permanent
service iptables save

# Restart iptable service and check status
systemctl restart iptables
systemctl status iptables

# Verify again from jmp and lbr server. You should not get any response 
# on JMP server.
curl -I stapp01:8084
curl -I stapp02:8084
curl -I stapp03:8084
```