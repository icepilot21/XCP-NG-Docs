# XCP-NG

## Install XCP-NG

## Install "Free" Xen Orchestra Appliance
The Xen Orchestra Appliance provided through the base XCP-NG install will allow the user to create VMs and perform basic adminstrative capabilities, but it places a number a key capabilities behind a paywall.

## Compile Xen Orchestra Appliance from Source
This installation has been validated against a fresh Debian 11 (Bullseye) x64 install. It should be nearly the same on other dpkg systems. For RPM based OS's, it should be close, as most of our dependencies come from NPM and not the OS itself.

### Create a Virtual Machine through the "Free" XOA
#### Minimum Specs
- 2 vCPU
- 4GB Ram
- 20GB Disk
- Supported Operating Systems
    - CentOS 8 Stream
    - AlmaLinux 9
    - AlmaLinux 8
    - Rocky Linux 8
    - Red Hat Enterprise Linux 8
    - Debian 11
    - Debian 10
    - Debian 9
    - Debian 8
    - Ubuntu 22.04
    - Ubuntu 20.04
    - Ubuntu 18.04
    - Ubuntu 16.04

#### Install Git & Vim
- Red Hat based Distros
    - `sudo dnf -y install git vim`
- Debian based Distros
    - `sudo apt update && sudo apt -y upgrade && apt -y install git vim`

#### Open Firewall Rules
- Red Hat based Distros
    - `sudo firewall-cmd --permanent --add-service={http,https}`

#### Clone XenOrchestraInstallerUpdater Repo
- `cd ~`
- `git clone https://github.com/ronivay/XenOrchestraInstallerUpdater.git`
- `cd XenOrchestraInstallerUpdater/`

#### Install/Update Xen Orchestra Appliance
- `cp sample.xo-install.cfg xo-install.cfg`
- `vim xo-install.cfg` and change the following lines
    - PORT="443"
    - CONFIGUPDATE=true
    - PATH_TO_HTTPS_CERT=$INSTALLDIR/xo.crt
    - PATH_TO_HTTPS_KEY=$INSTALLDIR/xo.key
    - AUTOCERT="true"
- `sudo ./xo-install.sh`
- `Answer 1` if you want to install
- `Answer 2` if you want to update

## Post Installation

### Install Guest Management Tools
https://xcp-ng.org/docs/guests.html#linux
- Debian (Install from the guest tools ISO)
    -  Attach the guest tools ISO to the VM from XOA
    -  SSH into the VM, and elevate to root
```
mount /dev/cdrom /mnt
bash /mnt/Linux/install.sh
umount /dev/cdrom
systemctl enable xe-linux-distribution.service
systemctl start xe-linux-distribution.service
```
- RedHat
    - This is available through the EPEL repo
    - I am using RHEL 8
```
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
yum -y update
sudo yum -y install xe-guest-utilities-latest
systemctl enable xe-linux-distribution.service
systemctl start xe-linux-distribution.service
```

## Virtual Machine Migration
Virtual Machines can be migrated from a variety of sources. The vendor docs can be found at [Migrate to XCP-NG](https://xcp-ng.org/docs/migratetoxcpng.html) but I found that they were slightly lacking

### KVM/QEMU/PROXMOX to XCP-NG
#### **On each Source VM**
Update the VM to include needed dracut tools to support migration to XCP-NG
- Log in to the VM
- Install Dracut Tools
    - Red Hat based Distros
        - `dnf -y install dracut-config-generic dracut-network`
    - Debian based Distros
        - `apt -y install dracut-config-generic dracut-network`
- Execute Dracut Tools
    - `dracut --add-drivers xen-blkfront -f /boot/initramfs-$(uname -r).img $(uname -r)`
    - Red Hat BIOS based VMs
        - `dracut --regenerate-all -f && grub2-mkconfig -o /boot/grub2/grub.cfg`
    - Red HatUEFI based VMs
        - `dracut --regenerate-all -f && grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg`
    - Debian based VMs
        - `dracut --regenerate-all -f && update-grub`
- Shutdown the VM

#### **On the current QEMU Hypervisor**
- SSH into the Hypervisor
- Install QEMU Tools
    - Red Hat based Distros
        - `dnf -y install qemu-img`
    - Debian based Distros (Proxmox)
        - `apt -y install qemu-utils`
- Convert QEMU compatible VM to XCP-NG compatible VM
    - `cd /location/where/virtual/discs/are/stored`
    - `qemu-img convert -O vpc myvm.qcow2 myvm.vhd`

>**Note:** This will make a copy of the source Virtual Disk so ensure you have enough disk space

- Copy the new VHD file to the new XCP-NG hypervisor
    - `rsync myvm.vhd root@10.10.10.10:/root/ --progress --sparse`

#### **On the New XCP-NG Hypervisor**
- SSH into the Hypervisor
- Complete the QEMU to XCP-NG VM conversion
    - `vhd-util repair -n myvm.vhd`
    - `vhd-util check -n myvm.vhd` this should return `myvm.vhd is valid`
>**Note:** If it does not start he conversion process over from the beginning
- Identify the UUID of the Storage Resource you want to run the VM from
    - `xe sr-list`
```
uuid ( RO)                : 3255b4c3-ff78-fce7-1ee0-4d351642bd27
          name-label ( RW): Local storage
    name-description ( RW): 
                host ( RO): xcp-ng01.localhost.localdomain
                type ( RO): lvm
        content-type ( RO): user

```
- Create an empty Virtual Disk Image (VDI)
    - `xe vdi-create sr-uuid=3255b4c3-ff78-fce7-1ee0-4d351642bd27 virtual-size=20000000000 name-label=myvm`
>**Note:** Make the VDI with the virtual size of your VHD + 1GB (i.e the virtual size of myvm is 19GB, so create a VDI with a size of 20GB).

>**Note:** The `virtual-size` filed is in Kilobytes

>**Note:** Use the UUID of the SR you found in the provious step in the `sr-uuid` filed

- Obtain the UUID of the VDI you just created
    - `xe vdi-list name-label=myvm`
```
uuid ( RO)                : b54f7d29-67d1-4f15-af27-2673a2dc4bce
          name-label ( RW): myvm
    name-description ( RW): Created by XO
             sr-uuid ( RO): 3255b4c3-ff78-fce7-1ee0-4d351642bd27
        virtual-size ( RO): 21474836480
            sharable ( RO): false
           read-only ( RO): false
```
- Import the VHD into the VDI
    - `xe vdi-import filename=myvm.vhd format=vhd --progress uuid=b54f7d29-67d1-4f15-af27-2673a2dc4bce`
- Once the import is done, create a virtual machine using the Xen Orchestra Web GUI
- Disable `Boot VM after creation` on the Advanced Tab
- Delete the VM disk that has been created and attach your newly created VDI to the VM
    - This can be done via `Home > VMs > myvm > Disks`
>**Note:** Don't forget to set the VM boot mode to UEFI if your VMs was in UEFI mode. (Also Under Advanced)
- Power on the VM
