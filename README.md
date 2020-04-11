Bare_Metal_Kubeadm
=========

## Summary

This project (BMK) is the successor to [TKS](https://github.com/zimmertr/Bootstrap-Kubernetes-with-QEMU). TKS was used for years but became difficult to maintain with continuing infrastructure changes.

This Ansible project is used to stand up Kubernetes on a physical server. The procedure is quite opiniated. If it matches with your infrastructure you may find it quite useful. If not, hopefully you still find it a little useful. That being said, this project is much less opiniated than my last one. In fact, you might consider it _cloud-aware_ since you are no longer required to tie it to any underlying infrastructure like `QEMU`.

<hr>

## Background

I've been using Kubernetes at home for a few years and my infrastructure has gone through a few different architectural models. In this most recent one, I have decided to move away from virtualization all together in order to minimize computational overhead on aging physical resources. Who knew running Jira at home was so expensive anyway?

Speaking of those aging physical resources, my home server is a hyperconverged 2008 Mac Pro. It's been my trusty workhorse for the past 5 or so years. It's ran everything from vROps on ESXi to Windows Server on Proxmox. Recently I have rearchitected much of my home infrastructure to run on Kubernetes. As a result, virtualization was becoming less and less of a need and more and more of a resource sink.

BGP was previously used with MetalLB in order to put pods on specific VLANs as a means for improved security. In hindsight, this was a poor decision because it forced packets to move through my 1Gb router instead of switching virtually within the hypervisor. To reduce this bottleneck, and improve security by several magnitudes beyond that, I have adopted Istio for network control instead.

The underlying storage is managed by two ZFS Storage Pools. `SaturnPool` was previously used as both file and computational storage under Proxmox. In an effort to improve IOPS, `ComputePool` has since been created. Which is a mirrored pair of [Samsung 980 Evo m.2 NVMe SSDs](https://www.samsung.com/semiconductor/minisite/ssd/product/consumer/970evo/). These were originally intended to be used as underlying virtual machine disks but I thought putting container storage directly on them would be more performant in the end. Especially when factoring in things like database seeks.

| Storage Pool | RAID Level | Raw Size | Available Size | Disk Type                                    |
| ------------ | ---------- | -------- | -------------- | -------------------------------------------- |
| SaturnPool   | RAID 10    |          |                | 3x HGST 6TB 7200RPM<br />3x HGST 4TB 7200RPM |
| ComputePool  | RAID1      | 1024GB   | 512GB          | 2x Samsung 970 Evo M.2 NVMe                  |

Time to address the elephant in the room -- Isn't it a huge mistake to be running a single node cluster? Basically yes, it is. But I don't have any other physical servers and my processors are over 10 years old. Virtualization is expensive and I'm pushing my hardware to it's actual limits to afford  some additional time to say up for new hardware. If you're looking for something with more worker nodes, look at my old project: [TKS](https://github.com/zimmertr/Bootstrap-Kubernetes-with-QEMU).

<hr>

Requirements
------------

1. [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
2. A Bootable USB Device.

<hr>

## Role Variables

Modify the variables present in each of these files. Sometimes you will find variables that are commented out. This means that a default variable has been set in `ansible/roles/ROLE/defaults/main.yml`. If you uncomment this variable and populate it with a value it will override the default value. See `sensible/roles/ROLE/README.md` for clarifications on the use of each variable if  necessary.

* `ansible/roles/create_install_medium/vars/main.yml`
* `ansible/roltes/configure_kubernetes/vars/main.yml`

<hr>

Instructions
----------------

### Creating the bootable USB drive

1. Insert your bootable USB drive into your workstation.

2. Modify the variables files  documented above.

3. Create the installation media:

   ```bash
   sudo ansible-playbook ansible/create_install_media.yml
   ```

### Installing CentOS on your server

1. Unplug the bootable USB drive from your workstation and move it to the server. Power it on and instruct the firmware to boot from the drive. Most computers assign one of the `function` keys  to this  process. My Mac Pro assigns `option` to it.

2. Speaking of my Mac Pro, it fails to boot off this media unless I provide an extra option to GRUB. So **only perform this step if the install disk hangs for you.**

   * Select `Install CentOS Linux`  and press the `e` key.

   * Move your cursor to the end of line that begins with `linux` or `linuxefi` and append the following option:

   * ```bash
     acpi=false
     ```

   * Press `f10` to continue booting from the modified GRUB configuration.

3. Configure your install as you wish. I did not want to provide a response file out of fear of accidentally messing up partioning. I typically make the following notable configurations:

   | Configuration       | Reason                                                       |
   | ------------------- | ------------------------------------------------------------ |
   | UTC Time            | Servers play together better when using the same time zone.  |
   | Minimal Install     | Servers don't need a lot of fluff when all of the lift is done by an application. |
   | Dedicated SSD       | I like to dedicate an SSD in my server for my Host OS install. |
   | Manual Partitioning | Standard Partitioning: `/` is an `ext4` of almost max size and `/boot/efi/` is a 1GiB EFI. |

4. Let the system install, reboot your server, remove the flash drive, and boot into the OS. Troubleshoot as necessary.

### Preparing the server

1. If you have a DHCP server and map static IPs to known Mac Addresses in your environment you can likely now ping your server from your original workstation. If not, log into the server and run `ip addr` to determine the IP address it has been given. If it does not have one, troubleshoot as necessary.

2. Modify the `inventory.yml` file in `./ansible/inventory.yml` to use your IP Address/Hostname instead of mine.

3. `ssh` into the server and generate an SSH key.

   ```bash
   ssh root@server
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/key
   cat ~/.ssh/key.pub > ~/.ssh/authorized_keys
   ```

4. Now that you have created an SSH Key and installed it on the server, copy the private key to  `./ansible/ssh.key`. Make sure you do this before running the next few commands. If you don't, you will lose access to your server.

   ```bash
   sed -i 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd_config
   sed -i 's/\#PasswordAuthentication no/PasswordAuthentication no/g' /etc/ssh/sshd_config
   sed -i 's/PermitRootLogin no/PermitRootLogin without-password/g' /etc/ssh/sshd_config
   sed -i 's/\#PermitRootLogin no/PermitRootLogin without-password/g' /etc/ssh/sshd_config
   systemctl restart sshd
   exit
   ```

5. Confirm you can still access the server with your copied private key.

   ```bash
   ssh -i ./ansible/ssh.key root@server
   exit
   ```

### Configuring ZFS

1. These steps are **only required if you are following my [infrastructure model](https://github.com/zimmertr/Network-Diagram) and have existing ZFS Storage Pools** to import for Kubernetes. If this does not apply to you, skip to `Bootstrap Kubernetes`.

2.  Prepare ZFS:

   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/prepare_zfs.yml
   ```

### Bootstrap Kubernetes

1. Install all of the dependencies for and bootstrap a Kubernetes Cluster:

   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/bootstrap_kubernetes.yml
   ```

2. The previous playbook will reboot the server upon completion. Once it comes back up, copy the Kubernetes configuration to your workstation and confirm that the API Server is responding

   ```bash
   scp -ri ./ansible/ssh.key root@server:/root/.kube/ ~/
   kubectl get svc
   ```

### Configure Kubernetes

1. Now that Kubernetes is up and running, deploy some necessary workloads to it to make it fully usable.

   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/configure_kubernetes.yml
   ```

2. Confirm that all of the deploy pods are up:

   ```bash
   kubectl get po -A
   ```

3. If you want to copy this configuration file back to your workstation:
   ```bash
   scp -ri ~/.ansible/ssh.key root@server:/root/.kube/ ~/
   ```

### Continuing On

You now have a fully operational single node Kubernetes cluster with an untainted API Server and the ability for pods to network with one another over a virtual network known as a CNI. These are the bare essentials needed to get started with deploying workloads to Kubernetes.

It is likely that you will need the ability to set up controllers and operators to managed resources like Load Balancers, Ingress Controllers, and Persistent Storage volumes. I maintain another project [here](https://github.com/zimmertr/Kubernetes-Manifests) called Kubernetes-Manifests that automates the procedure of deploying these common applications as well as lots of homelab and enterprise-centric ones like Jira, Plex Media Server, and Unifi Controller.

<hr>

## FAQ

1. **Q:** Why don't you use a Kickstart file when installing the operating system?
   **A:** My server has 9 hard drives connected that make up multiple RAID arrays and are connected via 4 different systems. It's hard using Apple hardware as your hypervisor sometimes. I would hate to accidentally install the OS on the incorrect disk and destroy an array.

<hr>

## TODO

| Item               | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| Kubernetes Version | Support specifying specific Kubernetes Version               |
| ZFS Snapshots      | Enable regular snapshotting of ZFS storage pools             |
| Automate Sanoid    | Sanoid was installed manually                                |
| Configurable CNI   | Support deploying Weave Net, Flannel, Cilium                 |
| Automate SSH Keys  | Automate the creation and distribution of keys               |
| Harden System      | Harden with SSH configuration, user accounts, fail2ban, etc  |
| Configure SMTP     | Set up notifications for S.M.A.R.T. status, package updates, etc |
| SELinux Support    | Figure out how to use Kubernetes with SELinux properly       |
| Automate PKI       | Automate the creation and distribution of PKI                |
| CentOS 8           | Add support for CentOS 8                                     |
| RHEL               | Add support for RHEL 7 & 8                                   |
| Fedora             | Add support for Fedora                                       |

<hr>

## Interested in Contributing?

If you're interested in helping with this project here are a few ideas:

1. Adding support for differnet storage backends using Ansible/OpenEBS.
2. Adding support for different Cloud/Infrastructure providers using Terraform.
3. Automating the deployment of an SMTP Relay, Sanoid, ZFS ZED, etc.





