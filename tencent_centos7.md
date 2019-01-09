# Tencent centos 7 setup

## Init

1. Choose password login.
2. Choose security ip rules. (Enable 22)
3. Login as root through default login.

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
