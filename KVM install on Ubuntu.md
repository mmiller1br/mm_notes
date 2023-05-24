#kvm #ubuntu

## check hardware capability

1. using the command "lscpu"
```bash
nuc@nucserver:~$ lscpu | grep Virtualization
Virtualization:                  VT-x
nuc@nucserver:~$
```

BLANK screen = no virtualization capability

2. using cpu info:
```bash
nuc@nucserver:~$ egrep -c '(vmx|svm)' /proc/cpuinfo
8
nuc@nucserver:~$
```

3. using cpu-checker
```bash
sudo apt update

sudo apt upgrade -y

sudo apt install cpu-checker

nuc@nucserver:~$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
nuc@nucserver:~$
```

If your CPU does not support, the MSG will be: "Your CPU does not support KVM extensions"


## Install KVM on Ubuntu

```bash
apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils -y
apt install virt-manager -y
```

Pcakages:
-   `qemu-kvm` – This package provides accelerated KVM support. This package is a userspace  component for QEMU, which is responsible for accelerating KVM guests.
-   `libvirt-daemon-system` – This package provides the `libvirt` daemon and management tools. The `libvirt` daemon is responsible for creating, managing, and destroying virtual machines on your server.
-   `libvirt-clients` – This package provides the `libvirt` command-line interface for managing virtual machines.
-   `bridge-utils` – This package provides the userland utilities for configuring virtual network bridges. On a hosted virtualization solution like KVM, a network bridge connects the guest VM network to the host network.
-   `virt-manager` – This package provides a graphical user interface (GUI) for managing virtual machines.

Checking:
```bash
systemctl status libvirtd
```

## Adding your local user to libvirt group
```bash
sudo usermod -a -G libvirt $(whoami)
```
-a = append user
-G = group name
libvirt = group name
(whoami) = your local user (if you are logged in with this user)

```bash
newgrp libvirt
```

And now restart the service:
```bash
sudo systemctl restart libvirtd.service
```


## SOURCE: 
https://adamtheautomator.com/ubuntu-install-kvm/
