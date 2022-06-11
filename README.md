# XCP-NG

## Virtual Machine Migration
Virtual Machines can be migrated from a variety of sources. The vendor docs can be found here but I found that they were slightly lacking

[Migrate to XCP-NG](https://xcp-ng.org/docs/migratetoxcpng.html)

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
    - BIOS based VMs
        - `dracut --regenerate-all -f && grub2-mkconfig -o /boot/grub2/grub.cfg`
    - UEFI based VMs
        - `dracut --regenerate-all -f && grub2-mkconfig -o /boot/efi/EFI/<your distribution>/grub.cfg`
- Shutdown the VM

#### **On the current QEMU Hypervisor**
- SSH into the Hypervisor
- Install QEMU Tools
    - Red Hat based Distros
        - `dnf - y install qemu-img`
    - Debian based Distros (Proxmox)
        - `apt -y install qemu-utils`
- Convert QEMU compatible VM to XCP-NG compatible VM
    - `cd /location/where/virtual/discs/are/stored`
    - `qemu-img convert -O vpc myvm.qcow2 myvm.vhd`

>**Note:** <span style="color:red">This will make a copy of the source Virtual Disk so ensure you have enough disk space</span>

- Copy the new VHD file to the new XCP-NG hypervisor
    - `rsync myvm.vhd root@10.10.10.10:/root/ --progress --sparse`

#### **On the New XCP-NG Hypervisor**
- SSH into the Hypervisor
- Complete the QEMU to XCP-NG VM conversion
    - `vhd-util repair -n myvm.vhd`
    - `vhd-util check -n myvm.vhd` this should return `myvm.vhd is valid`
>**Note:** <span style="color:red">If it does not start he conversion process over from the beginning</span>
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
    - `xe vdi-create sr-uuid=3255b4c3-ff78-fce7-1ee0-4d351642bd27 virtual-size=20000000 name-label=myvm`
>**Note:** <span style="color:red">Make the VDI with the virtual size of your VHD + 1GB (i.e the virtual size of myvm is 19GB, so create a VDI with a size of 20GB).</span>

>**Note:** <span style="color:red">The `virtual-size` filed is in Kilobytes</span>

>**Note:** <span style="color:red">Use the UUID of the SR you found in the provious step in the `sr-uuid` filed</span>

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
>**Note:** <span style="color:red">Don't forget to set the VM boot mode to UEFI if your VMs was in UEFI mode. (Also Under Advanced)</span>
- Power on the VM
