---
summary: SSH user Jailing with native tools and bindfs mounts
---

> This has been reworked [in here]((../ssh-jailing-with-alpine-linux-docker/))

# Basic idea #
* Use a seperate VM for this - jails are under `/jails/`
* Every user which sould be jailed into his folder `/jails/[USERNAME]` is in group `sshjailed`
* `sshd` is configured to use multiple ports and activate jailing if required ↑
* Every jail has its own binaries and libs
* External folders are mounted by fstab with e.g. `cifs`

## Setup sshd for the new user ##
`sudo nano /etc/ssh/sshd_config`
```apacheconf
Port 22
Port 1234
Subsystem   sftp    internal-sftp
```
[...]
```apacheconf
Match group sshjailed
    # Forces the user into /jails/[USERNAME]/home/[USERNAME]
    ChrootDirectory /jails/%u
    AllowTcpForwarding no
    X11Forwarding no
```

## Prepare global jail ##
```bash
sudo mkdir /jails
sudo chown root: /jails
sudo chmod 750 /jails
```

## Create the individual jail-cell with the user ##
### For simpler use... ###
...let us define a variable for the user here: `export CHROOT_USER_NAME=[USERNAME]`

### Add the user ###
```bash
sudo adduser --no-create-home $CHROOT_USER_NAME
sudo adduser $CHROOT_USER_NAME sshjailed
```

### Create his root ###
For more hints about the used chroot env [see here](https://wiki.alpinelinux.org/wiki/Alpine_Linux_in_a_chroot)!
```bash
sudo mkdir /jails/$CHROOT_USER_NAME
sudo chown root: /jails/$CHROOT_USER_NAME
sudo chmod 755 /jails/$CHROOT_USER_NAME
```
Now download the latest alpine linux installer (`apk-tools-static`) from [here](http://dl-cdn.alpinelinux.org/alpine/latest-stable/main/) and extract it with `tar -xzf apk-tools-static-*.apk`
```bash
sudo ./sbin/apk.static -X http://dl-cdn.alpinelinux.org/alpine/latest-stable/main -U --allow-untrusted --root /jails/$CHROOT_USER_NAME --initdb add alpine-base bash openssh git nano iputils doxygen graphviz
sudo mkdir /jails/$CHROOT_USER_NAME/home/$CHROOT_USER_NAME
sudo chown $CHROOT_USER_NAME: /jails/$CHROOT_USER_NAME/home/$CHROOT_USER_NAME
echo $(getent passwd $CHROOT_USER_NAME) | sudo tee -a /jails/$CHROOT_USER_NAME/etc/passwd
sudo usermod --shell /usr/sbin/nologin $CHROOT_USER_NAME
sudo ln -s ../../bin/bash /jails/$CHROOT_USER_NAME/usr/sbin/nologin
```

### Enable networking inside the chroot env ###
1. Add a barebone to allow bind mounting `sudo touch /jails/$CHROOT_USER_NAME/etc/resolv.conf`
2. Add a mount to `/etc/fstab`:
    ```
    /etc/resolv.conf    /jails/[USERNAME]/etc/resolv.conf   none    ro,bind 0   0
    ```

### Insert a moint point(s) ###
1. `sudo mkdir -p /jails/$CHROOT_USER_NAME/mnt/[MOUNT_POINT]`
2. Add the mount to `/etc/fstab`
    * Native version
        ```
        [SOURCE_PATH]    /jails/[USERNAME]/mnt/[MOUNT_POINT] none    bind    0   0
        ```
    * (requires `bindfs`)
        ```
        [SOURCE_PATH]    /jails/[USERNAME]/mnt/[MOUNT_POINT] fuse.bindfs   force-user=[USERNAME],force-group=[USERNAME or e.g. www-data],create-for-user=[USERNAME],create-for-group=[USERNAME or e.g. www-data],perms=770,create-with-perms=770,chmod-filter=770,chown-ignore,chgrp-ignore,resolve-symlinks,resolved-symlink-deletion=symlink-only,hide-hard-links    0  0
        ```
        _Note that `git` seems to have a problem with the `force-*` and `create-for-*` options. They will cause `git submodule update` or `git init` to fail with an `Operation not permitted`!_
    * or use you own cifs / sshfs / ... magic!
