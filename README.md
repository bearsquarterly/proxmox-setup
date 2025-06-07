# proxmox-setup

i have been working on getting to this point for literally like 7 years. and it has not been easy on account of googling anything linux related is a nightmare of guys arguing with each other about how you do it wrong instead of helping you do it. so i will just give you this and if you want it you can use it.

so - this is a script that will do exactly two things on a new instance of Proxmox: make your SMB share automounted to the proxmox host on boot, and setup the ability to passthru hardware from your system to LXCs/VMs. 

at the end of this read.me i will also tell you how you can auto mount that share to an LXC you have running when it boots.

the body of the script is this:

```
#!/bin/bash
set -e

echo "===================================="
echo "🔧 Proxmox Node Setup Starting..."
echo "===================================="

### --- SMB Share Setup ---
echo "📁 Setting up NAS share and credentials..."

cat <<EOF > /root/.smbcredentials
username=USERNAME
password=PASSWORD
EOF

chmod 700 /root/.smbcredentials
echo "✔️ Credentials stored and secured at /root/.smbcredentials."

mkdir -p /mnt/samba
echo "✔️ Created mount point at /mnt/samba."

FSTAB_LINE="//xxx.xxx.xxx.xxx/your_share_name /mnt/samba cifs credentials=/root/.smbcredentials,uid=100000,gid=100000,x-systemd.automount,_netdev 0 0"

if grep -Fxq "$FSTAB_LINE" /etc/fstab; then
    echo "ℹ️ NAS share already in /etc/fstab, skipping."
else
    echo "$FSTAB_LINE" >> /etc/fstab
    echo "✔️ NAS share added to /etc/fstab."
fi

### --- IOMMU & VFIO Setup ---
echo "🧩 Configuring IOMMU and VFIO passthrough..."

# Update GRUB default
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="[^"]*/& intel_iommu=on iommu=pt/' /etc/default/grub
echo "✔️ GRUB_CMDLINE_LINUX_DEFAULT updated."

update-grub2
echo "✔️ GRUB updated."

# Set EFI kernel command line
echo "quiet intel_iommu=on iommu=pt" > /etc/kernel/cmdline
pve-efiboot-tool refresh
echo "✔️ Kernel command line and EFI boot refreshed."

# Add VFIO modules
cat <<EOF >> /etc/modules
vfio
vfio_iommu_type1
vfio_pcie
vfio_virqfd
EOF

echo "✔️ VFIO modules added to /etc/modules."

update-initramfs -u
echo "✔️ Initramfs updated."

echo "===================================="
echo "✅ Setup complete. Reboot is required."
echo "===================================="
read -p "Press Enter to reboot now..." dummy
reboot
```
you will need to edit this to suit your setup. 
1. add your SMB user and password where it says 'username=' or 'password=' with your credentials with no spaces (eg: username=MyUser )
2. at FSTAB line, you will need to change ' //xxx.xxx.xxx.xxx/your_share_name ' to be the ip address of your server, and the name of the shared folder your SMB user uses. ( eg. //192.168.1.50/share )
3. this assumes you have intel as your CPU on your proxmox host. if you don't, you need change any instance of intel with amd.

to walk you thru what this will do:

1. it will create a hidden file called 'smbcredentials' that stores your Credentials for your SMB user.
2. it will make that file usable by the 'root' user on your proxmox host.
3. it makes a directory in /mnt called samba. this is the folder that binds to your SMB share on your host. you can change 'samba' to anything you want, but you will need to change any references to it later in the script to match
4. it will then edit your fstab file to automount your SMB share at boot with the correct user permissions for LXCs in proxmox. if you have specific users and groups outside of the default, this will not work for you.
5. if you happen to already have this in your fstab file already, it will not edit your fstab file.
6. it will edit your GRUB file to enable hardware passthru by adding a line to the GRUB file
7. it will then update your GRUB so it will do this upon reboot. this step can take a moment if it looks like the script is hung up.
8. it will then update the EFI kernel command line with a similiar line, and then update that tool so it works when you reboot
9. lastly it will edit your /etc/modules file, with settings for VFIO. this is about virtualizing things for proxmox for hardware passhthru. then it will update that file.
10. after it has done all of this, it will prompt you to reboot by hitting Enter
11. after it reboots, if you go into the shell and do
```
ls /mnt/samba
```
you should see the files in your SMB share.

in the shell if you do

```
ls /dev/dri
```
you should see it list a few things, including renderD128, which is usually the iGPU of an intel CPU for things like plex transcoding.

to Get this script onto your proxmox host, open up the shell on the host and use:
```
cat > /root/proxmox_node_setup.sh
```

paste the script, hit enter once, and then hit Ctrl+D. this will copy the contents of the script into a new file in your root folder called 'proxmox_node_setup.sh'

then in the shell, use:
```
chmod +x proxmox_node_setup.sh
```

this will make that script executable. 

then to actually run the script, in the shell use:
```
./proxmox_node_setup.sh
```

this will then run the script.

that should set up everything for a basic proxmox node with your SMB share

to ADD that SMB folder to any LXC you have;

1. in the shell of the proxmox host, use:
```
nano /etc/pve/lxc/<ID of your LXC>.conf
```
eg:
```
nano /etc/pve/lxc/107.conf
```
this will open the config file of your LXC. then add the line:
```
lxc.mount.entry: /mnt/samba /var/lib/lxc/<ID of your LXC>/rootfs/mnt/<name of folder you want on LXC> none bind,create=dir
```
eg:
```
lxc.mount.entry: /mnt/samba /var/lib/lxc/107/rootfs/mnt/shared none bind,create=dir
```
hit Ctrl+X, then Y to save. this makes sure your SMB share that we bound to your Proxmox host, is now bound to a folder on your LXC. and then that will make sure on boot of the LXC it mounts that share to the folder.

you will need to repeat that for every LXC you want to have access to the SMB share.

i hope this helps you! 
