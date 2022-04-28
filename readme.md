# ESXi Ubuntu Cloud-Init 

This playbook is used to quickly deploy Ubuntu cloud-images and bootstrap the OS using cloud-init manifest on a standalone ESXi host (without vcenter).    

Using a cloud image with cloud-init is significantly faster than ISO mount install.  In my environment, I can have Ubuntu up and running in under 1 minute. 

## TLDR

1. Clone this repo
2. update `group_vars/all.yaml` with your network settings and ESXi IP and login credentials
3. update `vars` in `playbook.yaml`
4. run `ansible-playbook -i 'localhost' playbook.yaml` 

## How it works
vmWare `govc` is used to deploy an ova to ESXi.  This playbook uses ansible templating engine to generate a script (on your local filesystem not esxi) and then executes it.  Also, by using Ansible, configuring vmware tools `userdata` is greatly simplified.

Be advised this playbook is not idempotent. I may refactor it to use Ansible Pyvmomi module at some point but for now it meets my needs as a tool to rapidly deploy and bootstrap Ubuntu on ESXi. 

## Requirements 
Before you can run this playbook you need to make sure the following software is installed.
* python3
* python3-pip
* ansible (pip install)
* jmespath (pip install)
* netaddr (pip install)
* jq
* govc https://github.com/vmware/govmomi/tree/master/govc (or homebrew on a mac)

Before proceeding make sure you pip installed binaries are in your PATH.  A quick test is to run simply run `ansible -v`. 

## Variables
There are two files that contain playbook variables.  `group_vars/all.yaml` contains variables that are more like constants meaning you shouldn't have to change them after they've been set.  The other is in the playbook itself (`playbook.yaml`) which are variables you set on a playbook run.  Variables such as hostname, ip, storage allocation, vcpu, mem, etc.  

`group_vars/all.yaml`
  
* Ensure keys under `local_lan_info` match your network configuration.
* Do not change any values with `{{ }}`.  These are Ansible variables which resolved when the playbook runs. 
* Update env to match your ESXi configuration.  Its likely you only need to change `GOVC_URL` and `GOVC_PASSWORD`.
* `govc ls` cli command is an easy way to find the correct values.  
* WARNING: you can use a plain text password for `GOVC_PASSWORD` but never commit it to public repo.  That would be very bad. Very bad.

```yaml
local_lan_info:
  gateway: 192.168.86.1
  subnet: 255.255.255.0
  nameservers:
    - 192.168.86.3
env:
  GOVC_INSECURE: 1
  GOVC_URL: 192.168.86.101
  GOVC_USERNAME: root
  GOVC_NETWORK: VM Network
  GOVC_RESOURCE_POOL: '*/Resources' 
  GOVC_DATACENTER: ha-datacenter
  # lookup ansible vault password. plain text value works as well
  GOVC_PASSWORD: "{{ lookup('file', 'group_vars/secret.yaml') }}"
  # GOVC_PASSWORD: MyPlainTextPassword
cloud_image:
  '20.04': https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.ova
  '21.04': https://cloud-images.ubuntu.com/releases/hirsute/release/ubuntu-21.04-server-cloudimg-amd64.ova
  '21.10': https://cloud-images.ubuntu.com/impish/current/impish-server-cloudimg-amd64.ova
  '22.04': https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.ova
```
  
Playbook vars (`playbook.yaml`) are self-explanatory. Note: `ubuntu-version` maps to values in `cloud_image` key in `group_vars/all.yaml` and `user_data_file` is covered in the next section.  The data_disks key is an optional variable used to create and attach additional block devices to the vm.  `cloud-init-d/ud-docker-data-disk.yaml` is a cloud-init that demonstrates how to mount, partition and create filesystem on additional disks. 

```yaml
  vars:
    vmname: docker1
    cpu: 2
    mem: 8192
    disk: 64G
    # dhcp or ip
    ip: dhcp
    # datastore name
    ds: ds-nvme-evo5
    data_disks:
      # optional 
      # docker-data: 20G
    # bios or efi
    firmware: bios
    serial_port: no
    user_data_file: cloud-init-d/ud-docker.yaml
    ubuntu_version: 22.04
    ssh_public_key_file: ~/.ssh/id_rsa.pub
```

## Cloud-Init UserData File

  Cloud-init is very cool and this playbook only scratches the surface on what it can do.  At a high level, its a declarative way to bootstrap the OS with packages and configure various settings. The following user-data example updates all existing packages, adds docker apt repo, installs docker, installs various apt packages, configures password-less sudo, adds public ssh key, sets group memberships, sets default password of 'password' and forces a reset on the first login. You can find more examples in the `cloud-init-d` directory. 

```yaml
#cloud-config
apt:
  sources:
    docker:
      arches: amd64
      source: "deb https://download.docker.com/linux/ubuntu $RELEASE stable"
      keyserver: "hkp://keyserver.ubuntu.com:80"
      keyid: 0EBFCD88
package_upgrade: true
packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
groups:
  - docker
users:
  - default
  - name: ubuntu
    # ssh_key value is resolved at runtime 
    ssh-authorized-keys:
      - "{{ ssh_key }}"
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [sudo,docker]
    shell: /bin/bash
ssh_pwauth: True
chpasswd:
  list: |
    ubuntu:password
```

## Common package file

File `cloud-init-d/common-packages.yaml` contains standard apt packages to install.  This list is merged with userdata file specified in the playbook.  Use common-packages.yaml for packages that are always installed (jq, tree, etc).   

## Running the playbook

Now that variables are set and a `user_data_file` is specified in `playbook.yaml` you are ready to run the playbook.

```bash
ansible-playbook -i 'localhost,' playbook.yaml
```

If you want to test govc script generation without deploying the ova simply pass an extra var called `test_mode` with value of true. You can then inspect the generated script in `tmp/resolved_script.sh`

```bash
ansible-playbook -i 'localhost,' playbook.yaml -e test_mode=true
```

## Post deployment

The vm is automatically started.  After a few moments the IP address will be displayed in terminal output. Note, its possible cloud-init is still running.  To check cloud-init status tail `/var/log/cloud-init-output.log`.  

```bash
ssh username@ip_address
sudo tail -f /var/log/cloud-init-output.log
```

## Helpful govc commands
> you will need to export govc environment variables.  see govc documentation

Tree 

```bash
govc tree
```

LS

```bash
govc ls 
```

Open VMRC from CLI

```bash
  govc vm.console my-vm
  govc vm.console -capture screen.png my-vm  # screen capture
  govc vm.console -capture - my-vm | display # screen capture to stdout
  open $(govc vm.console my-vm)              # MacOSX VMRC
  open $(govc vm.console -h5 my-vm)          # MacOSX H5
  xdg-open $(govc vm.console my-vm)          # Linux VMRC
  xdg-open $(govc vm.console -h5 my-vm)      # Linux H5
```

Power off and destroy vm 

```bash
govc vm.power -off my-vm
govc vm.destroy my-vm
```