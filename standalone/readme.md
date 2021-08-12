# Standalone Cluster

We will set up and Anthos Standalone cluster with Ubuntu server as the OS. While we create 

## Prerequisites

* Create or use a GCP Project
* KVM installed 



## Set up workstation

* Download kickstart script for the workstation 

```
```

Change the username and password in this file to values of your choice from the current values in this line
```
#Initial user (user with sudo capabilities) 
user ubuntu --fullname "Ubuntu User" --password ch@ng3m3
```

* Create a virtual machine for workstation

```
virt-install \
--name=anthos-ws \
--ram=8192 \
--disk path=/var/lib/libvirt/images/anthos-ws.qcow2,size=50,bus=virtio,cache=none,format=qcow2,bus=virtio \
--vcpus=2 \
--virt-type=kvm \
--os-type linux \
--os-variant ubuntu20.04 \
--graphics none \
--network network:default \
--location='http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/' \
--initrd-inject ubuntu.cfg \
--extra-args "ks=file:/ubuntu.cfg console=tty0 console=ttyS0,115200n8" 
```

Once you see a message such as below press `^]` to get back to CLI

```
Domain creation completed.
Restarting guest.
Connected to domain anthos-ws
Escape character is ^]
```

Run `virsh list` to see the running VM

```
# virsh list
 Id   Name            State
-------------------------------
 28   anthos-ws       running
```

To find the IP address assigned to the box run `virsh domifaddr $VM_NAME`

```
virsh domifaddr anthos-ws
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet27     52:54:00:f8:7b:92    ipv4         192.168.122.211/24
```

* Login using the above IP with username and password you set in the ubuntu.cfg file. Preferably do this from another terminal as you will need the current terminal to add other VMs later.

```
ssh ubuntu@198.168.122.211
```

* Generate SSH keys running `ssh-keygen`.

* Install Docker

```
sudo apt-get install docker
sudo apt-get install docker.io
```

Verify docker version to be `19.03.13` or higher running `docker version`

* Manage Docker as a non-root user as listed [here](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)

```
USER=<<SUBSTITUTE USER NAME>>
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker 
```

Verify running `docker run hello-world` as non-root user.

* Install snap
```
sudo apt install snapd
```

* Install gcloud
```
snap install google-cloud-sdk --classic
```


* Login using gcloud and configure your project

```
gcloud init
```

* Install bmctl

Download
```
gsutil cp gs://anthos-baremetal-release/bmctl/1.8.2/linux-amd64/bmctl .
```

Provide exec permissions

```
chmod +x bmctl
```


## Set up nodes for master(s) and worker(s)

Switch back to the window 

* Create a VM for master

```
virt-install \
--name=anthos-master1 \
--ram=16384 \
--disk path=/var/lib/libvirt/images/anthos-master1.qcow2,size=150,bus=virtio,cache=none,format=qcow2,bus=virtio \
--vcpus=4 \
--virt-type=kvm \
--os-type linux \
--os-variant ubuntu20.04 \
--graphics none \
--network network:default \
--location='http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/' \
--initrd-inject ubuntu.cfg \
--extra-args "ks=file:/ubuntu.cfg console=tty0 console=ttyS0,115200n8" 
```

* Create one or more VMs for worker(s). Change CPU, RAM and disk size settings as per your needs

```
virt-install \
--name=anthos-worker1 \
--ram=32768 \
--disk path=/var/lib/libvirt/images/anthos-worker1.qcow2,size=150,bus=virtio,cache=none,format=qcow2,bus=virtio \
--vcpus=6 \
--virt-type=kvm \
--os-type linux \
--os-variant ubuntu20.04 \
--graphics none \
--network network:default \
--location='http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/' \
--initrd-inject ubuntu.cfg \
--extra-args "ks=file:/ubuntu.cfg console=tty0 console=ttyS0,115200n8" 
```

```
virt-install \
--name=anthos-worker2 \
--ram=32768 \
--disk path=/var/lib/libvirt/images/anthos-worker2.qcow2,size=150,bus=virtio,cache=none,format=qcow2,bus=virtio \
--vcpus=6 \
--virt-type=kvm \
--os-type linux \
--os-variant ubuntu20.04 \
--graphics none \
--network network:default \
--location='http://archive.ubuntu.com/ubuntu/dists/bionic/main/installer-amd64/' \
--initrd-inject ubuntu.cfg \
--extra-args "ks=file:/ubuntu.cfg console=tty0 console=ttyS0,115200n8" 
```

## Establish connectivity between the machines

### Find the Node IP Addresses
* List all the VMs running `virsh list`

```
 Id   Name             State
--------------------------------
 28   anthos-ws        running
 30   anthos-master1   running
 36   anthos-worker1   running
 38   anthos-worker2   running
```

* Find the ip address for each node running `virsh domifaddr $VM_NAME` and note them.


### Set up all the Nodes so that you can login as root from the workstation

* Switch to the window that has SSH connection to the workstation

* Copy SSH key from workstation to the each box using it's IP address. Use the user name and password provided in the kickstart file.

```
ssh-copy-id <SUBSTITUTE USERNAME>>@<<NODE_IPADDRESS>>
```

* SSH to the box without using the username and password

```
ssh <SUBSTITUTE USERNAME>>@<<NODE_IPADDRESS>>
```

* **Passwordless Root Login:** Verify if you can `sudo bash` as root without password. If you cannot, then change the permissions in the `/etc/ssh/sshd_config` and restart `sshd` service as root user.

```
sudo sed -i "s/^#PermitRootLogin.*$/PermitRootLogin yes/g" /etc/ssh/sshd_config
```

Restart SSH service
```
sudo systemctl restart sshd
```

* Enable passwordless sudo. Replace the username with the value used used to login to the box

```
USERNAME=<<SUBSTITUTE USERNAME>>
sudo sed -i '/^#includedir \/etc\/sudoers.d/a '"$USERNAME"' ALL=(ALL) NOPASSWD: ALL' /etc/sudoers
```

* Verify you can sudo without password. The following command should not ask for password now.

```
sudo bash
```

* Set root password. Choose a strong password

```
sudo passwd
```

* Exit `exit`

* Allow root access from workstation by copying the ssh key for the `root` user using root password that you just assigned in the last step.

```
ssh-copy-id root@<<NODE_IPADDRESS>>
```

* (Verify that you can) Login from the workstation without password

```
ssh root@<<NODE_IPADDRESS>>
```

* Lock root password

```
sudo passwd -l root
```

* Change root login to `without-password` and restart ssh service

```
sudo sed -i "s/^PermitRootLogin.*$/PermitRootLogin without-password/g" /etc/ssh/sshd_config
```

```
systemctl restart sshd
```

```
exit
```

* Repeat the above steps for all the Node VMs

## Install Cluster

* Export the chosen project as the CLOUD_PROJECT_ID

```
export CLOUD_PROJECT_ID=$(gcloud config get-value project)
```

* Login to gcloud

```
gcloud auth application-default login
```

Note the Output
```
...
Credentials saved to file: [/home/ubuntu/.config/gcloud/application_default_credentials.json]
...
...
```

* Create a standalone config file. Substitute the cluster name.

```
./bmctl create config -c STANDALONE_CLUSTER_NAME --enable-apis \
    --create-service-accounts --project-id=$CLOUD_PROJECT_ID
```
Note the location of the config file in the output

* Edit the config file for the following parameters

* `sshPrivateKeyPath:` - Fill in the path of ssh key from the workstation
* `type: standalone`
* `profile: edge`
* `controlPlane.nodePoolSpec.nodes` set the value of `address` to the ip address of master VM
* make sure `clusterNetwork.pods.cidrBlocks` range does not overlap the VM Ip address range. If it is, change this CIDR block. 




*** Work in progress **


