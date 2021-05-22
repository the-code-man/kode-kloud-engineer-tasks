# System Administrator Tasks - Set 2

- [Task 22 - Configure Local Yum repos](#configure-local-yum-repos)
- [Task 23 - Linux Nginx as Reverse Proxy](#linux-nginx-as-reverse-proxy)
- [Task 24 - Web Server Security](#web-server-security)
- [Task 25 - Application Security](#application-security)
- [Task 26 - Linux Find Command](#linux-find-command)
- [Task 27 - Linux Resource Limits](#linux-resource-limits)
- [Task 28 - Setup SSL for Nginx](#setup-ssl-for-nginx)
- [Task 29 - Linux Bash Scripts](#linux-bash-scripts)
- [Task 30 - Configure protected directories in Apache](#configure-protected-directories-in-apache)
- [Task 31 - Install a Package](#install-a-package)
- [Task 32 - Linux Firewalld Setup](#linux-firewalld-setup)
- [Task 33 - Install and Configure HaProxy LBR](#install-and-configure-haproxy-lbr)

## Configure Local Yum repos

The **Nautilus** production support team and security team had a meeting last month in which they decided to use local yum repositories for maintaing packages needed for their servers. For now they have decided to configure a local yum repo on **Nautilus Backup Server**. This is one of the pending items from last month, so please configure a local yum repository on **Nautilus Backup Server** as per details given below.

a. We have some packages already present at location **/packages/downloaded_rpms/** on **Nautilus Backup Server**.

b. Create a yum repo named **localyum** and make sure to set **Repository ID** to **localyum**. Configure it to use package location **/packages/downloaded_rpms/**.

c. Install package **wget** from this newly created repo.

### Solution

```bash
# Connect via SSH to the Backup server.
ssh clint@stbkp01

# Switch to super user
sudo su -

# Check the availability of packages in the local directory
ll /packages/downloaded_rpms/

# Create the yum repo  in /etc/yum.repos.d/
cd /etc/yum.repos.d/
vi localyum.repo

# Add following entries to the file
[localyum]
name=localyum
baseurl=file:///packages/downloaded_rpms/
gpgcheck=0
			
# removes cache of repos enabled in /etc/yum        # Not a mandatory step but recommended
yum clean all

# installs, updates, removes packages               # Not a mandatory step but recommended
yum update

# lists the repos created. Our new repo must be listed here
yum repolist

# Install the required package.
yum install wget        # Alternatively you can also run yum install --disablerepo="*" --enablerepo="localyum" wget. This will ensure packages are installed from local repo

# Verify package is running.
systemctl status wget

# Exit from the server
exit
```

## Linux Nginx as Reverse Proxy

**Nautilus** system admin team is planning to deploy a front end application for their backup utility on **Nautilus Backup Server**, so that they can manage the backups of different websites from a graphical user interface. They have shared requirements to set up the same; please accomplish the tasks as per detail given below:

a. Install **Apache** Server on **Nautilus Backup Server** and configure it to use **8089** port (do not bind it to **127.0.0.1** only, keep it default i.e let Apache listen on server IP, hostname, localhost, 127.0.0.1 etc).

b. Install **Nginx** webserver on **Nautilus Backup Server** and configure it to use **8097**.

c. Configure **Nginx** as a reverse proxy server for **Apache**.

d. There is a sample index file **/home/index.html** on **Jump Host**, copy that file to **Apache's** document root.

e. Make sure to start **Apache** and **Nginx** services.

f. You can test final changes using **curl** command, e.g **curl http://<backup server IP or Hostname>:8097**.

### Solution

```bash
# Connect via SSH to the backup server.
ssh clint@stbkp01

# Switch to root user
sudo su -

# Check and Install Apache.
rpm -qa | grep httpd        # Check if package is already installed

yum install httpd -y           # install the package

# configure httpd to listen to 8089 port
vi /etc/httpd/conf/httpd.conf

# Traverse the config and set the listen port
Listen 8089

# Check and Install httpd.
rpm -qa | grep nginx        # Check if package is already installed

yum install nginx -y

# This will result in no package being found because Nginx package will be found in the epel repo. Hence we have to install that first
yum install epel-release -y

yum install nginx -y

# Take backup of nginx config file (Good practice)
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# Check if backup created successfully (Good practice) 
ls -l /etc/nginx/

# Update nginx config
vi /etc/nginx/nginx.conf

# The server section mut look something like this.
server {
    listen 8097 default_server;                         # nginx port to listen
    listen [::]:8097;
    server_name 172.16.238.16;                          # ip address of the backup server
    .....

    location / {
        proxy_pass http://172.16.238.16:8089/;          # apache port
    }
    ....

# Check if the nginx configuration is valid
nginx -t

# Return to the jumphost and copy the index.html from jumphost to the Backup server.
sudo scp /home/index.html clint@stbkp01:/tmp

# SSH back to backup server and move index.html to the apache's document root 
mv /tmp/index.html /var/www/html/

# verify if the file was moved into the correct directory within the backup server.
ls -l /var/www/html/

# Start httpd and nginx services and check the status
systemctl start httpd && systemctl start nginx
systemctl status httpd && systemctl status nginx

# Verify that everything is working as they should be -- use curl. Check from both backup and jump server
curl http://172.16.238.16:8097
curl http://stbkp01:8097

# Exit from lab
exit
```

## Web Server Security

During a recent security audit, the application security team of **xFusionCorp Industries** found security issues with the Apache web server on **Nautilus App Server 3** server in **Stratos DC**. They have listed several security issues that need to be fixed on this server. Please apply the security settings below:

a. On **Nautilus App Server 3** it was identified that the Apache web server is exposing the version number. Ensure this server has the appropriate settings to hide the version number of the Apache web server.

b. There is a website hosted under **/var/www/html/blog** on **App Server 3**. It was detected that the directory **/blog** lists all of its contents while browsing the URL. Disable the directory browser listing in Apache config.

c. Also make sure to restart the Apache service after making the changes.

### Solution

```bash
# SSH into App Server 3
ssh banner@stapp03

# Switch to root user
sudo su -

# Check status of apache server
systemctl status httpd

# Start the service if it is stopped
systemctl start httpd

# Open a new terminal and try to connect to apache server
curl -I http://stapp02:8080

# It returns Server information along with other output:
......
Server: Apache/2.4.6 (CentOS) PHP/7.2.26			# 2.4.6 is the version number.
......

# We need to prevent the version number from being displayed

# In the new terminal navigate to the blog url
curl -I http://stapp02:8080/blog/

# It returns Ok that means indexes will be displayed when you navigate to the link in browser. It should display forbidden

# Edit httpd configuration to fix the issues
vi /etc/httpd/conf/httpd.conf

# Add the following towards end of the configuration
ServerSignature Off     # tells apache not to display the server version on error pages, or other pages it generates.
ServerTokens Prod       # tells apache to only return Apache in the Server header, returned on every page request

# This will fix the first issue where version number is getting displayed

# For fixing the second issue. Search for /var/www/html directory option
# The Options under it will looking something like this
Options Indexes FollowSymLinks

# Remove Indexes option. Indexes options allows apache web server to enable automatic index generation.

# Save the configuration

# Restart apache server
systemctl restart httpd

# Verify
curl -I http://stapp02:8080

curl -I http://stapp02:8080/blog/

# Exit from app server
exit
```

## Application Security

We have a backup management application UI hosted on **Nautilus's** backup server in **Stratos DC**. That backup management application code is deployed under Apache on the backup server itself, and Nginx is running as a reverse proxy on the same server. Apache and Nginx ports are **8097** and **5001**, respectively. We have iptables firewall installed on this server. Make the appropriate changes to fulfill the requirements mentioned below:

We want to open all incoming connections to Nginx port and block all incoming connections to Apache port. Also make sure rules are permanent.

### Solution

```bash
# SSH into backup server
ssh clint@stbkp01

# Switch to root user
sudo su -

# Check status of iptables service
systemctl status iptables 

# Start iptables service, if stopped
systemctl start iptables 

# Enable iptables service to start at boot
systemctl enable iptables 

# Check existing rules
iptables -L

# Add rule to accept traffic on port 8097
sudo iptables -A INPUT -p tcp --dport 8097  -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT

# Add rule to reject traffic on port 5001
sudo iptables -A INPUT -p tcp --dport 5001 -m conntrack --ctstate NEW -j REJECT

# Rules created with the iptables command are stored in memory. If the system is restarted before saving the iptables rule set, all rules are lost. For netfilter rules to persist through system reboot, they need to be saved.

# save iptables rule
service iptables save

# This executes the iptables initscript, which runs the /sbin/iptables-save program and writes the current iptables configuration to /etc/sysconfig/iptables. The existing /etc/sysconfig/iptables file is saved as /etc/sysconfig/iptables.save. 

# Commit the rules
iptables-save

# Check if new rules are applied
iptables -L

# Exit from server
exit

# Validate
curl -I 172.16.238.16:8097
curl -I 172.16.238.16:5001

# OR

telnet 172.16.238.16 8097
telnet 172.16.238.16 5001
```

## Linux Find Command

During a routine security audit, the team identified an issue on the Nautilus App Server. Some malicious content was identified within the website code. After digging into the issue they found that there might be more infected files. Before doing a cleanup they would like to find all similar files and copy them to a safe location for further investigation. Accomplish the task as per the following requirements:


a. On **App Server 3** at location **/var/www/html/beta** find out all files (not directories) having **.js** extension.

b. Copy all those files along with their **parent directory structure** to location **/beta** on same server.

c. Please make sure not to copy the entire **/var/www/html/beta** directory content.

### Solution

```bash
# SSH into App server 3
ssh banner@stapp03

# Switch to root user
sudo su -

# Verify the directories
ls -l /var/www/html/beta    # Source
ls -ltr /beta               # Target

# Find out all files having .js extension 
find /var/www/html/beta -name '*.js' | wc -l

# Copy All the files to target beta directory
find /var/www/html/beta -name '*.js' -exec cp --parents {} /beta \;         # --parents flag - will cause the full path to be copied. {} - Copy All SOURCE to DEST.

# Verify the directories
sudo ls -ltr /beta | wc -l

# Exit from the app sever
exit
```

## Linux Resource Limits

On our **Storage** server in **Stratos Datacenter** we are having some issues where **nfsuser** user is holding hundred of processes, which is degrading the performance of the server. Therefore, we have a requirement to limit its maximum processes. Please set its maximum process limits as below:

a. soft limit = **74**

b. hard limit = **90**

### Solution

```bash
# Connect via SSH to the Storage Server.
ssh natasha@ststor01

# Switch to root user
sudo su -

# Edit the limits.conf file to add nfsuser and the hard and soft limits for the processes.
vi /etc/security/limits.conf

# Add the following lines to the end
# Pattern - <domain> <type> <item> <value>
# Domain – this includes usernames, groups, guid ranges etc
# Type – soft and hard limits
# Item – the item that will be limited – core size, file size,  nproc etc
# Value – this is the value for the given limit
nfsuser	soft	nproc	74
nfsuser	hard	nproc	90	

# Verify
cat /etc/security/limits.conf | grep nfsuser

# Exit from the storage server
exit
```

## Setup SSL for Nginx

The system admins team of **xFusionCorp Industries** needs to deploy a new application on **App Server 2** in **Stratos Datacenter**. They have some pre-requites to get ready that server for application deployment. Prepare the server as per requirements shared below:

1. Install and configure **nginx** on **App Server 2**.

2. On **App Server 2** there is a self signed SSL certificate and key present at location **/tmp/nautilus.crt** and **/tmp/nautilus.key**. Move them to some appropriate location and deploy the same in Nginx.

3. Create an **index.html** file with content **Welcome!** under Nginx document root.

4. For final testing try to access the **App Server 2** link (either hostname or IP) from **jump host** using curl command. For example **curl -Ik https://<app-server-ip>/**.

### Solution

```bash
# SSH into app server 2
ssh steve@stapp02

# Switch to root user
sudo su -

# Install nginx
yum install epel-release -y
yum install nginx -y

# Add ssl.conf file in the include /etc/nginx/conf.d/ directory. You can modify /etc/nginx/nginx.conf as well but this is the preferred way as it keeps the setting segregated
vi /etc/nginx/default.d/ssl.conf

# And add the following. Take a note of the directory for ssl certificate and key
server {
    listen       443 ssl http2 default_server;
    listen       [::]:443 ssl http2 default_server;
    server_name  172.16.238.11;                             # Add the IP address of the server
    root         /usr/share/nginx/html;

    ssl_certificate "/etc/pki/certs/nautilus.crt";
    ssl_certificate_key "/etc/pki/certs/nautilus.key";
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_ciphers PROFILE=SYSTEM;
    ssl_prefer_server_ciphers on;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    location / {
    }

    error_page 404 /404.html;
        location = /40x.html {
    }

    error_page 500 502 503 504 /50x.html;
        location = /50x.html {
    }
}

# Modify /etc/nginx/nginx.conf and add Server IP address to server_name parameter

# Move certificate and key to the referred directory
cp /tmp/nautilus.crt /etc/pki/certs/
cp /tmp/nautilus.key /etc/pki/private/

# Verify that the certificates are indeed copied.
ls -l /etc/ssl/certs/
ls -l /etc/ssl/private/

# Verify nginx configuration
nginx -t

# If the nginx.conf is good and no error is detected, it will return the following:
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# Start nginx service
systemctl start nginx
systemctl enable nginx
systemctl status nginx

# Verify from Jump Server
curl -Ik https://stapp02

# Result should look something like this
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Sun, 09 May 2021 04:43:53 GMT
Content-Type: text/html
Content-Length: 10
Last-Modified: Sun, 09 May 2021 04:43:21 GMT
Connection: keep-alive
ETag: "5f264469-a"
Accept-Ranges: bytes

# Exit from server 
exit
```

## Linux Bash Scripts

The production support team of **xFusionCorp Industries** is working on developing some bash scripts to automate different day to day tasks. One is to create a bash script for taking websites backup. They have a static website running on **App Server 3** in **Stratos Datacenter**, and they need to create a bash script named **official_backup.sh** which should accomplish the following tasks. (Also remember to place the script under **/scripts** directory on **App Server 3**

a. 	Create a zip archive named **xfusioncorp_official.zip** of **/var/www/html/official** directory.

b. 	Save the archive in **/backup/** on **App Server 3**. This is a temporary storage, as backups from this location will be clean on weekly basis. Therefore, we also need to save this backup archive on **Nautilus Backup Server**.

c. 	Copy the created archive to **Nautilus Backup Server** server in **/backup/** location.

d. 	Please make sure script will not ask for password while copying the archive file. Additionally, the respective server user (for example, **steve** in case of **App Server 2**) must be able to run it.

### Solution

```bash
# SSH into App Server 3
ssh banner@stapp03

# Use ssh-keygen to generate a ssh key pair. It will be saved in the .ssh directory.
ssh-keygen -t rsa -b 4096

# Copy public key to App Server 3
ssh-copy-id clint@stbkp01

# Verify. You should be able to login without specifying a password
ssh clint@stbkp01

# Exit from the backup server
exit

# Create the bash script
vi /scripts/official_backup.sh

# Add the following to the script file and save and exit

#!/bin/bash

zip -r /backup/xfusioncorp_official.zip /var/www/html/official
scp /backup/xfusioncorp_official.zip clint@stbkup01:/backup/

# Give permission to run the script
cd /scripts
chmod +x official_backup.sh

# Execute the script
sh official_backup.sh

# Verify if it worked correctly
ls -l /backup/
ssh clint@stbkp01
ls -l /backup/
```

## Configure protected directories in Apache

**xFusionCorp Industries** has hosted several static websites on **Nautilus Application** Servers in **Stratos DC**. There are some confidential directories on document root that need to be password protected. Because they are using Apache for hosting the websites, the production support team has decided to use **.htaccess** with basic auth. There is a website that needs to be uploaded to **/var/www/html/finance** on **Nautilus App Server 3**. However, we need to set up the authentication before that.

1. Create **/var/www/html/finance** direcotry if does not exist.

2. Add a user **javed** in **htpasswd** and set its password to **8FmzjvFU6S**.

3. There is a file **/tmp/index.html** placed on **Jump Server**. Copy the same to new directory you created, please make sure default document root should remain **/var/www/html**. Also website should work on URL **http://\<app-server-hostname\>:\<port\>/finance**

### Solution

```bash
# Copy the index.html from jumphost /tmp directory to the tmp directory on app server
ll /tmp/index.html
scp /tmp/index.html banner@stapp03:/tmp

# SSH into the App Server 3
ssh banner@stapp03

# Switch to root user
sudo su -

# Create the required directory in /var/www/html then create the .htaccess file
ls -l /var/www/html/
mkdir /var/www/html/finance 

# Generate the required username and password using htpasswd
sudo htpasswd -c /etc/httpd/.htpasswd javed
# Enter the provided password >> 8FmzjvFU6S

# create the .htaccess file
vi /var/www/html/finance/.htaccess

# the .htaccess file must contain:
AuthType Basic
AuthName "Password Required"
Require valid-user 
AuthUserFile /etc/httpd/.htpasswd

# Verify that the .htaccess and .htpasswd has the correct contents.
cat /var/www/html/finance/.htaccess
cat /etc/httpd/.htpasswd
ls -l /var/www/html/finance

# Move the index.html file from tmp to target directory
mv /tmp/index.html /var/www/html/finance

# Check status of apache and start the service
systemctl status httpd
systemctl start httpd
systemctl enable httpd
systemctl status httpd

# Switch to the App Server again and verify that it is able to curl the index.html using only the user.
curl -u siva http://stapp03:8080
curl -u siva http://stapp03:8080/finance/index.html
```

## Install a Package

As per new application requirements shared by the **Nautilus** project development team, serveral new packages need to be installed on all app servers in **Stratos Datacenter**. Most of them are completed except for **vsftpd**.

Therefore, install the **vsftpd** package on all **app-servers**.

### Solution

> Do this on all app servers

```bash
# SSH into App Server 1
ssh tony@stapp01

# Install the package 
sudo yum install vsftpd

# Verify that the package is installed
vsftpd --version
sudo yum list installed | grep vsftpd
```

## Linux Firewalld Setup

To secure our **Nautilus** infrastructure in **Stratos Datacenter** we have decided to install and configure **firewalld** on all app servers. We have Apache and Nginx services running on these apps. Nginx is running as a reverse proxy server for Apache. We might have more robust firewall settings in the future, but for now we have decided to go with the given requirements listed below:

a. Allow all incoming connections on Nginx port.

b. Allow incoming connections from LB host only on Apache port and block for all others.

c. All rules must be permanent.

d. Zone should be public.

e. If Apache or Nginx services are not running already, please make sure to start them.

### Solution

```bash
# SSH into App Server 1
ssh tony@stapp01

# Switch to root user
sudo su -

# Check the status of nginx and apache 
systemctl status nginx && systemctl status httpd

# Check port of nginx and apache
grep -i Listen /etc/httpd/conf/httpd.conf /etc/nginx/nginx.conf

# Check if firewalld is installed
yum repolist installed | grep firewalld

# Install firewalld, if not present
yum install firewalld -y

# Start and enable firewalld service. Also check if it is up and running
systemctl start firewalld && systemctl enable firewalld && systemctl status firewalld

## Add the rules, along with the rich rules
firewall-cmd --zone=public --add-port=8098/tcp --permanent          # nginx port
firewall-cmd --zone=public --add-service={http,https} --permanent
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="172.16.238.14" port port=5004 protocol=tcp accept' --permanent       # LB Server IP and apache port

# enable nginx && httpd
systemctl enable nginx && systemctl enable httpd && systemctl status nginx && systemctl status httpd

# Reload firewalld rules and restart the service to check if the rules are permanently added
firewalld-cmd --reload && systemctl restart firewalld 

# Verify the rules
firewall-cmd --zone=public --list-all

# Repeat on all other app servers

# Exit from app server
exit

# SSH into LB Server
ssh loki@stlb01

# Verify
# For the HTTP connecttions through NGINX port:
curl stapp01:8098
curl stapp02:8098
curl stapp03:8098

# For the HTTP connecttions through APACHE port:
curl stapp01:5004
curl stapp02:5004
curl stapp03:5004
```

## Install and Configure HaProxy LBR

There is a static website of **Nautilus** project running in **Stratos Datacenter**. Based on the infrastructure, they have already configured app servers and code is already deployed there. To make it work properly, they need to configure LBR server. There are number of options for that, but team has decided to go with **HAproxy**.

a. So install and configure **HAproxy** on **LBR** server using **yum** only and make sure all app servers are added to **HAproxy** load balancer. **HAproxy** must serve on default **http** port (**Note**: Please do not remove **stats socket /var/lib/haproxy/stats** entry from haproxy default config.).

b. You can access the website on LBR link—to do so click on the + button on top of your terminal, select option **Select port to view on Host 1**, and after adding port **80** click on **Display Port**.

```bash
# SSH into app server 1
ssh tony@stapp01

# Switch to root user
sudo su -

# Check the port that apache is listening on
grep -i Listen /etc/httpd/conf/httpd.conf

# Make a note of the port and ensure that apache is running and you are able to curl.
systemctl status httpd
systemctl enable httpd && systemctl start httpd && systemctl status httpd

curl localhost:8083
# You should get the below response
Welcome to xFusionCorp Industries!

##### THE ABOVE STEPS HAS TO BE PERFORMED ON APP APP SERVERS #####

# SSH into load balancer server
ssh loki@stlb01

# Switch to root user
sudo su -

# Install HAProxy load balancer using yum
yum install -y haproxy

# Check if stats socket /var/lib/haproxy/stats are present
grep -i haproxy/stats /etc/haproxy/haproxy.cfg 

# Edit the the configuration
vi /etc/haproxy/haproxy.cfg

# To meet the requirement for HAProxy to server on default http port
# Modify frontend to serve on port 80
frontend  main *:80

# Add all app servers to HAProxy load balancer
# add app servers in the 'backend app' section
## Make sure correct ports are added here
backend app
    balance     roundrobin
    server  stapp01 172.16.238.10:<httpd Port> check 
    server  stapp02 172.16.238.11:<httpd Port> check               
    server  stapp03 172.16.238.12:<httpd Port> check 

# Validate haproxy configuration
haproxy -f /etc/haproxy/haproxy.cfg

# enable and start haproxy service
systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy

# verfiy by curling to load balancer server from jump server
curl stlb01:80
```