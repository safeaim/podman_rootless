Ansible podman rootless role
=========
- [Ansible podman rootless role](#podman-rootless)
  * [Requirements](#requirements)
  * [Role Variables](#role-variables)
  * [Example playbook that builds local image and runs the container](#example-playbook-that-builds-local-image-and-runs-the-container)
  * [Example playbook with image from registry](#example-playbook-with-image-from-registry)


Ansible role that installs Podman, then sets up a service user, the correct SELinux environment, container image and systemd container unit for running the container as a rootless container.

The type of systemd unit is called a Quadlet. Read more about it here <https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html>. This is a format created to faciliate a better integration between Podman containers and systemd. 

In most cases you may need to prepare some files and directories that will be used for the container volumes. For that scenario I recommend you create a tasks file that will be run by the role. See the `container_prepare_tasks` for more information.

If you need to check on the container unit or user for that matter, you can use the following commands:
```bash
su -l service_user -s /bin/bash
[service_user@your_server ~]$ export XDG_RUNTIME_DIR=/run/user/$(id -u)
[service_user@your_server ~]$ systemctl --user status your-container
```
Note the export of XDG_RUNTIME_DIR. This is needed for systemctl commands to work, as using su to access the user does not set this environment variable.

If you need to debug the container unit, you can use podman-system-generator to see how the Quadlet unit will be rendered. Useful if you wanna see what Podman command is set for the ExecStart parameter of the unit:
```
[service_user@your_server ~]$ export XDG_RUNTIME_DIR=/run/user/$(id -u)
[service_user@your_server ~]$ /usr/lib/systemd/system-generators/podman-system-generator --user --dryrun
quadlet-generator[1744202]: Loading source unit file /var/lib/httpd/.config/containers/systemd/httpd-data.container
---httpd-data.service---
[Unit]
Description=A webserver for showing my cat images
After=network-online.target
Wants=network-online.target
SourcePath=/var/lib/httpd/.config/containers/systemd/httpd-data.container
RequiresMountsFor=%t/containers
RequiresMountsFor=/var/lib/httpd/data

[Install]
WantedBy=default.target

[X-Container]
Image=registry.hub.docker.com/library/httpd:latest
ContainerName=httpd-hello-world-webserver
Network=slirp4netns:port_handler=slirp4netns
PublishPort=8000:8000/tcp
User=696
Volume=/var/lib/httpd/data:/var/lib/httpd:U,Z
EnvironmentFile=/var/lib/httpd/httpd.env

[Service]
Restart=always
TimeoutStartSec=900
Environment=PODMAN_SYSTEMD_UNIT=%n
KillMode=mixed
ExecStop=/usr/bin/podman rm -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -f -i --cidfile=%t/%N.cid
Delegate=yes
Type=notify
NotifyAccess=all
SyslogIdentifier=%N
ExecStart=/usr/bin/podman run --name=httpd-data --cidfile=%t/%N.cid --replace --rm --cgroups=split --network=slirp4netns:port_handler=slirp4netns --sdnotify=conmon -d --user 696 -v /var/lib/httpd/data:/var/lib/httpd:U,Z --publish 8000:8000/tcp --env-file /var/lib/httpd/httpd.env registry.hub.docker.com/library/httpd:latest
```

In case this is being setup on a RHEL8 server, it also runs `setup-cgroupsv2`, which makes sure that CGroups V2 is setup. In case it has not been setup, it will change the GRUB configuration and reboot the server. CGroups V2 is needed for SystemD to have proper container process management when running Podman as rootless.  


Requirements
------------
Ansible Galaxy collections: 
- containers.podman
- community.general

Role Variables
--------------
All variables starting with `container`, are used to define the container unit, which uses the `templates/container.unit.j2` to generate the container unit file. 

- `service_user`: The username for the service.
  Example: `httpd`

- `service_user_uid`: The UID (User ID) for the service user. This is optional, as the user module will by default pick a UID under 1000 when creating the user. 
Example: `697`

- `service_user_shell`: Shell for `service_user`.
  Example: `/sbin/nologin`

- `service_user_homedir`: The home directory path for the service user.
  Example: `/var/lib/httpd`

- `container_registry_image`: The registry image that will be used by the container. This is optional, and only needs to be set if you are not using a locally built image.
  Example: `registry.hub.docker.com/library/httpd`

- `container_name`: The name of the container.
  Example: `httpd-hello-world-webserver`

- `container_service_description`: Description of the container service.
  Example: `A webserver for showing my cat images`

- `container_network_options`: Network options for the container.
  Example: `slirp4netns:port_handler=slirp4netns`
  It is recommeneded to use "slirp4netns:port_handler=slirp4netns", as the slirp4netns implementation can keep source IP addresses, while the default RootlessKit implementation treats all connections as if they were from 127.0.0.1. Note that slirp4netns is slower, but for most use cases it's fine. 

- `container_environment_file`: This is optional. This is the environment file that will be copied over to the home directory of the service user, and later be mounted on the running container. Add this to your files directory of your playbook for it to be copied over by this role.
  Example: `httpd.env`

- `container_publish_ports`: Ports to be published by the container. This takes the input as a list, and expects a mapping of container port, host port and protocol
  Example:
  ```yaml
    - "8000:8000/tcp"
  ```
- `firewalld_ports`: Ports to be opened in the firewall. Rootless Podman can't make changes to the firewall, so these are needed for the Firewalld config to be updated. By default this opens ports in the Public zone. 
  Example:
  ```yaml
  - "8000/tcp"
  ```
- `container_user_uid`: The user uid that is used inside the running container. This is optional. If not specified, it will use the UID of the `service_user`.
  Example: `0`

- `container_prepare_tasks`: In many cases you will need to for example create directories, render and/or copy files into directories that will be mounted on the container. Add a tasks file containing all those tasks in your playbooks tasks directory. This will run right before the container is started.
  Example: `prepare.yml`

- `container_volumes`: Volumes to be mounted on the container. Note that the directories that will be mounted need the SELinux context `container_file_t` set.  This could be set using your `container_prepare_tasks` task file like this:
```yaml
    - name: Set SELinux context on directories
      community.general.sefcontext:
        target: "/var/lib/httpd/{{ item.path }}"
        setype: "{{ item.setype }}"
        state: present
      loop:
        - { path: 'data(/.+)?', setype: 'container_file_t' }
        - { path: 'etc(/.+)?', setype: 'container_file_t' }

    - name: Set ownership of httpd directories
      ansible.builtin.file:
        path: "/var/lib/httpd"
        owner: httpd
        group: httpd
        mode: '0700'
        recurse: true

    - name: Restore SELinux context
      ansible.builtin.command: restorecon -FRv /var/lib/httpd
      register: restore_selinux_context
      changed_when: restore_selinux_context.rc != 0
```

This example uses the options Z and U. The `Z` option makes sure that Podman labels the content with a private unshared label, which means that only the current <<container|pod>> can use a private volume.  And `U` option makes sure that the files are owned by the `container_user` inside the container. 
Example:
```yaml
- "/var/lib/httpd/data:/var/lib/httpd:U,Z"
- "/var/lib/httpd/etc/httpd:/etc/httpd:U,Z"
```

- `container_AutoUpdate`: This is optional. This defined whether the SystemD Container unit will be auto-updating the container setup. It can be either registry or local. 
If the variable is present and set to registry, Podman reaches out to the corresponding registry to check if the image has been updated. An image is considered updated if the digest in the local storage is different than the one of the remote image. If an image must be updated, Podman pulls it down and restarts the systemd unit executing the container.

If the autoupdate label is set to local, Podman compares the image digest of the container to the one in the local container storage. If they differ, the local image is considered to be newer and the systemd unit gets restarted.
Example:
```yaml
container_AutoUpdate:
  type: registry
```

- `build_image`: Whether to build a local container image. This is false by default. This is used together with the variables below. 
Example: `false`

- `container_local_image`: The image to be used for the container. This will be used to name the locally built image, as well as being the image that the container is running. This is only needs to be set if building a local image, and not using the image directly from a registry.  
Example:
```yaml
container_local_image: my-locally-built-container-image
```

- `container_build_files`: A list of files that will be copied to `container_build_path`. 
Example:
```yaml
container_build_files:
  - Dockerfile
  - convert-cert.sh
```
- `container_build_path`: The path containing the Dockerfile and related files from which the container will be built. This is only needed if you need to build a local image. 
Example: `
```yaml
container_build_path: "/var/lib/httpd/build"
```


Example playbook that builds local image and runs the container
-----------------------------------------------------------------

In the first example this role is being used to setup the user environent, and also build a image from this Dockerfile:
```Dockerfile
FROM registry.hub.docker.com/library/httpd:latest
USER root
ADD 'http://www.tbs-x509.com/USERTrustRSACertificationAuthority.crt
      /usr/local/share/ca-certificates/

COPY 'convert-cert.sh' '/tmp/convert-cert.sh'
RUN /tmp/convert-cert.sh
RUN chmod 755 /usr/local/share/ca-certificates/*
RUN update-ca-certificates
```
prepare.yml:
```yaml
---
- name: Create directories for httpd
  ansible.builtin.file:
    path: "{{ item_folder }}"
    state: directory
    owner: httpd
    group: httpd
    mode: "0700"
  loop:
    - "/var/lib/httpd/data"
    - "/var/lib/httpd/etc/httpd"
    - "/var/lib/httpd/build"
  loop_control:
    loop_var: item_folder
```
playbook.yml
```yaml
- name: Example playbook
  hosts: myhost
  become: true
  vars:
    build_image: true
    service_user: "httpd"
    service_user_shell: "/sbin/nologin"
    service_user_homedir: "/var/lib/httpd"
    container_name: "httpd-data"
    container_build_path: "/var/lib/httpd/build"
    container_service_description: "httpd data container"
    container_network_options: "slirp4netns:port_handler=slirp4netns"
    container_environment_file: "httpd.env"
    container_local_image: "httpd-data"
    container_build_files:
      - Dockerfile
      - convert-cert.sh
    container_publish_ports:
      - "8000:8000/tcp"
    firewalld_ports:
      - "8000/tcp"
    container_user: "httpd"
    container_volumes:
      - "/var/lib/httpd/data:/var/lib/httpd:U,Z"
      - "/var/lib/httpd/etc/httpd:/etc/httpd:U,Z"
    container_prepare_tasks: prepare.yml
  tasks:

    - name: Setup Podman
      ansible.builtin.include_role:
        name: role_podman_rootless
      tags: create_container
```

Example playbook with image from registry
-----------------------------------------------------------------
In this example it's just running a image directly from a registry, without any image building.
```yaml
- name: Example playbook
  hosts: myhost
  become: true
  vars:
    service_user: "httpd"
    service_user_shell: "/sbin/nologin"
    service_user_homedir: "/var/lib/httpd"
    container_name: "httpd-data"
    container_service_description: "httpd data container"
    container_network_options: "slirp4netns:port_handler=slirp4netns"
    container_environment_file: "httpd.env"
    container_registry_image: "registry.hub.docker.com/library/httpd:latest"
    container_publish_ports:
      - "8000:8000/tcp"
    firewalld_ports:
      - "8000/tcp"
    container_user: "httpd"
    container_volumes:
      - "/var/lib/httpd/data:/var/lib/httpd:U,Z"
      - "/var/lib/httpd/etc/httpd:/etc/httpd:U,Z"
    container_prepare_tasks: prepare.yml
  tasks:
    - name: Setup Podman
      ansible.builtin.include_role:
        name: role_podman_rootless
      tags: create_container
```

Note the use of the `create_container` tag in both example. Running it with this tag lets you run the playbook to only setup the container unit, and if needed, rebuild the image. This is so that you can skip all the user environment setup, if that's already been done on your server.

License
-------
MIT
