# SFTP server setup


## Add a group `mateors`
> groupadd mateors

### How to generate passing using linux shell
> openssl passwd -6 'YourPassword'

## Add group
> mkdir -p /opt/mateors/admin

```shell
useradd -d "/opt/mateors/admin" -G "mateors" -s "/bin/false" -p $(openssl passwd -6 YourPassword) -c "mat user" "matadmin"
```

## Allow nginx to access files owned by the mateors group 
```
usermod -aG mateors nginx
```

### Make directory for setup virtual hosting for `workflow.lxroot.com`
> `mkdir -p /var/wwww/vhosts/workflow.lxroot.com`

## Create a user for domain `workflow.lxroot.com`
```shell
useradd -g mateors -p $(openssl passwd -6 YourPassword) -d "/var/www/vhosts/workflow.lxroot.com" -s "/bin/false" workflowlxrootcom
```

### Verify everything needed to setup sftp account for `workflow.lxroot.com`
```shell
id cc-licensingcom
getent group mateors
getent passwd workflowlxrootcom
namei -l "$(getent passwd workflowlxrootcom | cut -d: -f6)"
```


## SFTP group permission config for `mateors`
> `nano /etc/ssh/sshd_config.d/10-mateors.conf`
```
#Additional part of /etc/ssh/sshd_config Subsystem sftp internal-sftp
Match Group mateors
        PasswordAuthentication yes
        ChrootDirectory %h
        ForceCommand internal-sftp
        X11Forwarding no
        AllowTcpForwarding no
```

## Apply the following to take effect
```shell
sshd -t -f /etc/ssh/sshd_config.d/10-mateors.conf
systemctl -q restart ssh
sshd -t
```

### Verify
```shell
sshd -T -C user=cc-licensingcom,host=localhost,addr=127.0.0.1 \
  | grep -E '^(passwordauthentication|chrootdirectory|forcecommand|allowtcpforwarding|x11forwarding) '
```

> `systemctl reload ssh`

## Add persmission
> `chown -R cc-workflowlxrootcom:mateors /var/www/vhosts/workflow.lxroot.com/public_html`

### Verify all permission are correct.
> ls -ld /var/www/vhosts/workflow.lxroot.com/public_html

## Now you can login from any windows or linux terminal using
> `sftp` workflowlxrootcom@`SERVERIP`

YourPassword
