# Tencent centos 7 setup

## Init

1. Choose password login.
2. Choose security ip rules. (Enable 22)
3. Login as root through default login.

## Mount new disk

show free disk space - `# df -h`

## Create New user

1. Add group - `# groupadd <group-name>`
2. Add user to group - `# useradd -G <group> <username(dennis)>`
3. Add user to sudo group - `# usermod -aG wheel <user>` or `#gpasswd -a <user> wheel`

Show user info:
```
# id <user>
```

List all users
```
# cat /etc/passwd
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
   
## Create Swap file

**Do not use swap on SSD**

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
$ swapn -s // view
$ free -m
$ vim /etc/fstab // config to make it permanent
  /swapfile swap swap sw 0 0 // add this line
```

Reference: https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-centos-7

## MongoDb

Follow the official install document - https://docs.mongodb.com/v3.0/tutorial/install-mongodb-on-red-hat/

The repo file may include yum variables like $releasever. If incorrect variable value causes problem, we can just replace it with hard value, like 7 for centos 7, etc.

Ref: [How to view yum variables](https://unix.stackexchange.com/questions/19701/yum-how-can-i-view-variables-like-releasever-basearch-yum0)
