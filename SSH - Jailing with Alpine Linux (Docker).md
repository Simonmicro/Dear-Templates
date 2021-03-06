---
summary: SSH user Jailing with Docker and bindfs mounts
---

> This is an improved version of the [native SSH Jailing](../ssh-jailing-with-alpine-linux-native/), because multiple users will use the same environment (the Docker container) - instead of own directories. This saves disk space and allows the typical Docker features to be used (e.g. CPU and memory limiting). Additionally the more secure SSH key authentication is available! Also updates are much faster and easier...

# Basic idea #
* SSH daemon inside Docker container
* One container for one selection of users
* Folders are mounted as volumes and permissions enforced by bindfs

# Setup the Container #
As first we prepare the `docker-compose.yml` file:
```yaml
version: '3'
services:
    sshd:
        build: ./sshd/build
        restart: always
        ports:
            - "22222:22"
        volumes:
            - ./sshd/homes/:/home/
```
...which needs the following build environment files (at `./sshd/build`):
* ssh-keys folder - created with `mkdir -p ./sshd/build/ssh-keys/etc/ssh/ && ssh-keygen -A -f ./sshd/build/ssh-keys`
* `Dockerfile`
    ```docker
    FROM alpine:latest

    # Add some packages
    RUN apk add --no-cache openssh bash

    # Import the ssh server keys
    COPY ssh-keys/ /

    # Now start the SSH server
    WORKDIR /root
    COPY entry.sh .
    RUN chmod +x entry.sh
    CMD ["/bin/bash", "entry.sh"]
    ```
* `entry.sh`: To add own users, just duplicate the two lines and generate the password hash with `mkpasswd -m sha-512`. If you remove any user, you'll need to rebuild the container (otherwise the `/etc/passwd` wont be updated correctly).
    > Why are the users added at runtime? Well, because we have no way to over-mount the /home directory while building, therefore the users would have no home folder at runtime, if we put them into a volume.
    ```bash
    # Now (re-)add the user (using bash, otherwise Alpine defaults to ash)...
    adduser --shell /bin/bash -D glaforge || true
    echo 'glaforge:$6$sa0Dor.8a.$B2yhNI5KG76G416vHIxNAR/sd8TRtZ.4bMFVIVjZ.AYpB8iSddTNw2jdHPAhO7QUeaFSPvjpVG3qGFn18INeu.' | chpasswd -e

    # And start the ssh server
    /usr/sbin/sshd -D
    ```

## Start it... ##
Just enter `docker-compose up`

# Shared folders #
Because every user should be able to modify the files inside that folder, we want to ignore any permissions set by them. The easieast way to do that is to create a `bindfs` mount for that folder and expose it into the container (we can't do that inside, because e.g. Alpine Linux doesn't even have the required packages).

## Preperation on the host ##
Just create an other `bindfs` mount inside your `/etc/fstab` (make sure to have the required packages installed):
> Be aware that every file inside that folder will be editable as you would be root - so just mounting `/` would be an incredible bad idea...
```
[PATH_TO_YOUR_FOLDER]    [PATH_TO_YOUR_COMPOSE]/sharedfolder.bind fuse.bindfs   perms=777,create-with-perms=777,chmod-filter=777,chown-ignore,chgrp-ignore,resolve-symlinks,resolved-symlink-deletion=symlink-only,hide-hard-links    0  0
```
_Note that `git` seems to have a problem with the `force-*` and `create-for-*` options. They will cause `git submodule update` or `git init` to fail with an `Operation not permitted`!_

## Preperation at Docker ##
Now you just need to add the prepared folder as new volume:
```yaml
            - ./sharedfolder.bind:/host
```

## Customize your MOTD ##
Maybe you should consider to create a new `motd`-file to greet your users with a warm welcome - mount it into the container with:
```yaml
            - ./motd:/etc/motd:ro
```

## .bashrc ##
That file will be ignored - to use it anyways rename it to `.bash_profile`, which is only loaded after logging in (for any further subshell you can still use the `.bashrc`).
