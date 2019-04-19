# Tencent centos 7 setup

## Init

1. Choose password login.
2. Choose security ip rules. (Enable 22)
3. Login as root through default login.

## Mount new disk

show free disk space - `# df -h`

## Create New user

```
# Add group
$ groupadd <group-name>

# Add user to group
$ useradd -G <group> <username(dennis)>
$ passwd <user>

# Add user to sudo group 
$ usermod -aG wheel <user> // or:
$ gpasswd -a <user> wheel

# Show user info
$ id <user>

# List all users
$ cat /etc/passwd
```

## Enable SSH login

1. Login as normal user from PuTTY.
2. Create public keys.

  Copy existing public key string from local file (aliyun3_public) or existing public key file on server. Right-click mouse to copy pasted conent to PuTTY terminal. (*Important:* right-click mouse may miss the first character.)
  
  ```
  $ mkdir ~/.ssh
  $ chmod 0700 ~/.ssh
  $ touch ~/.ssh/authorized_keys
  $ chmod 0600 ~/.ssh/authorized_keys
  $ vim ~/.ssh/authorized_keys
  ```
  
3. Setup PuTTY profile.

   Copy and modify an existing PuTTY profile or create a new one. When create a new one, we need to configure:
   
   * Connection > data > Auto-login username: <user>
   * SSH > Auth: <private key file path>
   * SAVE session
  
4. Disable password login

   ```
   $ vim /etc/ssh/sshd_config
     - PasswordAuthentication no
     - UsePAM no
   $ service sshd restart
   ```

## Firewall

In centos 7, we can use firewalld package to setup firewall. Alternatively, we can use iptables directly.

Reference - https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7

```
# Install firewalld
$ yum install firewalld

# Enable firewalld on boot
$ systemctl enable firewalld
$ reboot // important

# Start firealld manually
$ systemctl start firewalld

$ sudo firewall-cmd --state // view state
$ firewall-cmd --get-default-zone
$ firewall-cmd --get-active-zones
$ firewall-cmd --list-all // list all details
$ firewall-cmd --zone=<name> --list-services // view services

# To make any change permanent, we can add --permanent. Otherwise the changes
# made will be discarded on reboot. This make it easy to test rules.

# Set active zone
$ firewall-cmd --zone=<name> --change-interface=<eth0|eth1>

# Add new port
$ firewall-cmd --zone=<public> --add-port=<5000>/<tcp|udp>

# Add service
$ firewall-cmd --get-services
$ firewall-cmd --zone=<name> --<add|remove>-service=<servie_name(http)>
```

**Troubleshooting**

When ports are not accessable from outside, it may not be because it is blocked but the process is only listen on local interface, ie `localhost` or `127.0.0.1`.

## Create Swap file

**Do not use swap on SSD**

https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-centos-7

```
$ swapon -s
$ free -m
$ df -h
$ fallocate -l <2>G /<swapfile> // can user any file name, use 2x memory size
```

When failed to allocate, ref: https://askubuntu.com/questions/563196/fallocate-failed-operation-not-supported

```
$ dd if=/dev/zero of=</swapfile> bs=1G count=2
$ dd if=/dev/zero of=</swapfile> bs=1G seek=2 count=0  //sparse
```

Enable swap file

```
$ ls -lh /swapfile // view
$ chmod 600 /swapfile
$ ls -lh /swapfile // check again
$ mkswap /swapfile
$ swapon /swapfile
$ swapon -s // view
$ free -m
$ vim /etc/fstab // config to make it permanent
  /swapfile swap swap sw 0 0 // add this line
```

Reference: https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-centos-7

Disable swap file

```
# Identify configured swap devices and files
$ cat /proc/swaps.
# Turn off all swap devices and files
$ swapoff -a.
# Remove any matching reference found in /etc/fstab.
```

## MongoDb

Follow the official install document - https://docs.mongodb.com/v3.0/tutorial/install-mongodb-on-red-hat/

Need to config selinux if enabled. Check selinux:

```
$ cat /etc/sysconfig/selinux
```

Set THP to 'never' to improve performance - http://docs.mongodb.org/manual/reference/transparent-huge-pages/#transparent-huge-pages-thp-settings

> Defrag may appeared to be <always> even when enabled is set to <never>. This can be safely ignored - https://jira.mongodb.org/browse/SERVER-18989

**Crash**

The main cause for unexpected crash is memory constrain. Use swapfile to increase memory capacity may help. We need to configure services to restart after crash. For centos 7+, we use `systemd`. In older versions, we may use a package like `monit`.

**Troubleshooting**

After installation, check service status, whether configured to run on start.

The repo file may include yum variables like $releasever. If incorrect variable value causes problem, we can just replace it with hard value, like 7 for centos 7, etc.

To view yum variables - https://unix.stackexchange.com/questions/19701/yum-how-can-i-view-variables-like-releasever-basearch-yum0

Make sure `/tmp/mongodb-27017.sock` is set to proper owner.

```
$ chown mongodb:mongodb /tmp/mongodb-27017.sock
```

## Auto restart services

`systemd` is controlled by `.service` files in a variety of locations. However, we wanted to avoid rewriting the existing startup script. - https://singlebrook.com/2017/10/23/auto-restart-crashed-service-systemd/

Tutorial - https://www.digitalocean.com/community/tutorials/how-to-configure-a-linux-service-to-start-automatically-after-a-crash-or-reboot-part-1-practical-examples

https://singlebrook.com/2017/10/23/auto-restart-crashed-service-systemd/

```
# Create a new file if it's not present
$ sudo vim /etc/systemd/system/multi-user.target.wants/<service>.service
> [Service]
> ...
> Restart=always

# To check the auto restart is configured successfully
$ sudo systemctl status <service>.service // view status
$ sudo kill -0 <PID> // kill process by PID
$ sudo systemctl status <service>.service // should restart with a new PID
```

## Create SSH tunnel

1. Create key pair

```
# Create key pair on the client (as the user who will establish the connection)
$ ssh-keygen -t rsa

# Set permissions on private keys
$ chmod 700 ~/.ssh // most likely already done
$ chmod 600 ~/.ssh/id_rsa

# Download and then upload the public key (id_rsa.pub) to the server 
# (as the user will be connected) and add to the authorized_keys file.
$ cat id_rsa.pub >> ~/.ssh/authorized_keys
# Alternatively, we can copy/pasted the public key the the authorized_keys file.

# Set permissions on server
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys

# Make sure the correct SELinux contexts are set
$ restorecon -Rv ~/.ssh
```

2. Add remote host to known_hosts

**IMPORTANT**: we need to add remote host to root's known_hosts if we want to run autossh from rc.d because that's run as root.

```
# Add remote to known_hosts for user
$ ssh-keyscan <new_host_address> >> ~/.ssh/known_hosts

# If manually ssh to the remote server, it will be permantently added.
$ ssh -L <port>:localhost:<port(27017)> -i ~/.ssh/<key(id_rsa)> <ssh_user(dennis)>@<host_ip>
```

If we got `bind: Cannot assign requested address`, most likely it was caused by IPv6. See - https://www.electricmonk.nl/log/2014/09/24/ssh-port-forwarding-bind-cannot-assign-requested-address/

```
# Add -4 switch to force IPv4
$ ssh -4 -L ...

# Permanently disable IPv6 by edit/create ~/.ssh/config, and add the following lines:
Host *
    AddressFamily inet

# Make sure to set proper permission
$ chmod 600 ~/.ssh/config
```

3. Config ssh on server

```
# Keep site wide ssh alive. See - https://patrickmn.com/aside/how-to-keep-alive-ssh-sessions/
$ vim /etc/ssh/sshd_config
ClientAliveInterval 300 // 5 min
ClientAliveCountMax 3 // number
```

4. Enable autossh

```
# Install package
$ sudo yum install autossh
$ autossh -M(monitor) 0(disable) -f(handle following ssh options) -L <27017>:localhost:<27017> -i <full_path_to_rsa_key> <remote_user>@<remote_host> -N
```

Add the autossh command line to `/etc/rc.d/rc.local` to config it to run on start. However, it will run as root by default. We should add the remote host to root's known_hosts, or add su - <user> to run as specified user.

## MySQL

Default mysql port is 3306.

```
# Add new user to mysql
$ > create user '<user(dennis11owen24)>'@'<host(localhost)>' identified by '<password(orson5welles6)>';
$ > grant all privileges on *.* to '<user(dennis11owen24)>'@'<host(localhost)>';
```

**Troubleshooting**

* When ssh tunnel to mysql server, we may not be able to use default mysql.socks to connect to forwarded server, use -h 127.0.0.1 to specify the host.
* We may need to add the local ip to user table. See - https://stackoverflow.com/questions/19101243/error-1130-hy000-host-is-not-allowed-to-connect-to-this-mysql-server

## MariaDB

https://mariadb.com/kb/en/library/yum/

Create repo file:

```
$ vim /etc/yum.repos.d/MariaDB.repo

[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.1/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

$ yum install MariaDB-server MariaDB-clinet
$ systemctl start mariadb
$ systemctl enable mariadb

# secure
$ mysql_secure_installation 
```

## Nginx

## nodejs

## Redis

## Image processing

In order to use gm node package, we need the GraphicMagick package.

```
# Install GraphicsMagick by yum
$ yum install GraphicsMagick
```

## Useful commands

```
# View centos version
$ cat /etc/centos-release

# Find file
$ find <directory> -name "<match(*.any)>"
```

**Services**

```
$ systemctl [status|start|stop|restart] <service>
$ systemctl enable <service>
$ systemctl disable <service>
$ systemctl list-unit-files | grep enabled
$ chkconfig [--level <level>] <service> <on|off>
$ chkconfig --list // view services configured to run on start
```

**Find if a port is open**

See - https://www.thegeekdiary.com/centos-rhel-how-to-find-if-a-network-port-is-open-or-not/

```
$ netstat -tulnp
$ ss -nutlp
$ lsof -i

# Test if remote port is accessable
$ cat < /dev/tcp/<host>/<port>
```

Response:

* `Connection refused` - target machine actively rejected the connection, i.e. there's a process listening on the address but refuse to respoonse.
* `No route to host` - there is a network problem, i.e. the address can not be accessed.

**Disk use**

https://www.tecmint.com/find-top-large-directories-and-files-sizes-in-linux/

```
# biggest directories
$ du -a </diretory> | sort -n -r | head -n 5 // a: display all, 
// -n: compare according to string numerical value, // -r: reverse, 
// head: output the first part of files, -n: print the first 'n' lines.

$ du -hs * | sort -rh | head -5 // human readable

$ du -Sh | sort -rh | head -5
// -h: print in human readable, -S: don't include subdirectories.

# Find files only
$ find </directory> -type f -exec du -Sh {} + | sort -rh | head -n 5
```

Clean up /var/log/journal/ - https://bbs.archlinux.org/viewtopic.php?id=158510 and https://ma.ttias.be/clear-systemd-journal/

```
# run from the same directory as the log file
cp /dev/null <mongod.log
```
