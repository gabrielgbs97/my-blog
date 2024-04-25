---
title: "Rootless Podman: restart rootless containers on boot"
date: 2024-25-04T18:33:00+01:00
kind: page
draft: true
---

Podman is an open-source container engine that allows developers to build, manage, and run containers directly on Linux systems. Unlike Docker, which relies on a central daemon, Podman operates in a daemonless architecture.

This approach enhances security and efficiency by running each Podman command in its own process space. Additionally, Podman supports running containers without root privileges by default, further improving security.

When using Podman, managing certain aspects of the container lifecycle externally becomes necessary due to the daemonless architecture.

A common issue I’ve encountered is the absence of a mechanism to automatically start _all_ my rootless containers during system startup. Rootful containers, on the other hand, can typically be started at boot using the built-in `podman-restart.service`.

Quadlet project was merged into podman to overcome this issue, see [podman-systemd-unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html). Unfornately, with this feature we have to maintain separate systemd-unit files for ever container or set of them.

My proposed solution relies on an unprivileged system user dedicated to run podman containers (i.e. no shell), and a user-based systemd unit file. Tests have been performed on a clean RHEL/Rocky/Alma 9 installation, but they should work on other Linux distributions with minor differences.

## Install podman

You can install `podman` and `podman-compose` as follows:

```Bash
sudo dnf install container-tools
sudo install python3-pip
sudo pip3 install podman-compose # optional
sudo systemctl enable --now podman-restart
sudo systemctl enable --now firewalld
```

## Dedicated system user

Unprivileged system service user `containeruser` will be created with home `/var/lib/containeruser`. User needs [systemd user lingering](https://www.freedesktop.org/software/systemd/man/latest/loginctl.html#enable-linger%20USER%E2%80%A6) to be enabled.

1. Create system service user `containeruser`.

    ```Bash
    export SERVICE="containeruser"
    sudo useradd -r -m -d "/var/lib/${SERVICE}" -s /sbin/nologin "${SERVICE}"
    # info https://www.freedesktop.org/software/systemd/man/latest/loginctl.html#enable-linger%20USER%E2%80%A6
    sudo loginctl enable-linger "${SERVICE}"
    ```

2. (Optional) A data directory `/var/lib/containeruser/data` could be used as a base path for containers filesystem mountpoints

    ```Bash
    sudo -H -u "${SERVICE}" bash -c "mkdir ~/data"
    ```

3. On systems where SELinux is enforced, context needs to be applied.

    ```Bash
    # Context selinux
    sudo semanage fcontext -a -t user_home_dir_t \
    "/var/lib/${SERVICE}(/.+)?"

    # Optional if data directory is created
    sudo semanage fcontext -a -t container_file_t \
    "/var/lib/${SERVICE}/data(/.+)?"
    
    sudo restorecon -Frv /var/lib/"${SERVICE}"
    ```

4. Rootless Podman requires the user running it to have a range of UIDs listed in the files `/etc/subuid` and `/etc/subgid`, [more info in podman GitHub](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md).

    ```Bash
    NEW_SUBUID=$(($(tail -1 /etc/subuid |awk -F ":" '{print $2}')+65536))
    NEW_SUBGID=$(($(tail -1 /etc/subgid |awk -F ":" '{print $2}')+65536))
    
    sudo usermod \
    --add-subuids ${NEW_SUBUID}-$((${NEW_SUBUID}+65535)) \
    --add-subgids ${NEW_SUBGID}-$((${NEW_SUBGID}+65535)) \
    "${SERVICE}"
    ```

5. (Optional) Container definition files need to be accessible from `containeruser`, I created a directory to store file like `YAML`, `podman-compose`, etc.

    ```Bash
    # Log in as user
    sudo -H -u containeruser bash
    cd
    mkdir ~/container-projects
    ```

6. Finally, create a user systemd unit to (re)start containers with a `restart-always` policy on boot.

    ```Bash
    mkdir -p ~/.config/systemd/user
    vim ~/.config/systemd/user/podman-restart.service
    ```

    ```ini
    # ~/.config/systemd/user/podman-restart.service
    [Unit]
    Description=Podman Start All Containers With Restart Policy Set To Always
    Documentation=man:podman-start(1)
    StartLimitIntervalSec=0
    Wants=network-online.target
    After=network-online.target

    [Service]
    Type=oneshot
    RemainAfterExit=true
    Environment=PODMAN_SYSTEMD_UNIT=%n
    Environment=LOGGING="--log-level=info"
    ExecStart=/usr/bin/podman $LOGGING start --all --filter restart-policy=always
    ExecStop=/bin/sh -c '/usr/bin/podman $LOGGING stop $(/usr/bin/podman container ls --filter restart-policy=always -q)'

    [Install]
    WantedBy=default.target

    ```

Then you can log in as `containeruser` with `sudo -H -u containeruser bash` to manage your containers. Test the automatic startup with a reboot of your enironment.

## Conclusion

With the rootless user `containeruser` and its systemd user `podman-restart` service, there is no need to maintain separate quadlet files. This streamlined approach simplifies container management and ensures that rootless containers start efficiently during system boot.

However, it’s essential to recognize that the podman quadlet systemd generator offers an even more declarative approach. By defining dependencies and configuration directly in systemd unit files, you can achieve fine-grained control over container startup and inter-dependency. Consider using the quadlet generator when managing complex container setups or orchestrating multiple services.

In summary, both approaches have their merits, and the choice depends on your specific use case and preferences.