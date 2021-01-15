# Overview

Ubuntu Desktop 20.04 supports a single ZFS boot drive out of the box. I wanted a ZFS mirror, without going through an entirely manual setup of Ubuntu as described at https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html

This adds a mirror to an existing Ubuntu ZFS boot drive after the fact.

ZFS requires native encryption to be added at pool / dataset creation. These instructions are not suitable for creating an encrypted ZFS boot disk, please use the full instructions linked above for that.

Note: If your use case is running docker instances, and not a full-fledged Ubuntu install, then take a look at [TrueNAS SCALE](https://www.truenas.com/truenas-scale/), which will manage the ZFS parts for you.

It can also be worthwhile to boot from a regular ext4 disk, whether single or mirrored, and then use a ZFS mirror pool for just `/home` and `/var`. In that case these instructions aren't needed.

# Why

ZFS has a few advantages that are good to have
- It uses checksums, which means that hardware failure and disk corruption will be detected and flagged during regular "scrub" operations
- It supports mirrors, which means that even with a failed drive, data is not lost
- It is a Copy-on-Write file system, which means that snapshots are fast to create and fast to roll back to (seconds), and only
take as much space as what was written after their creation. They can be created on a per-dataset basis.
- It has the concept of datasets, making it easy to take snapshots of specific portions of the file system, as desired.
- It can expand the size of a vdev by replacing first one, then the other drive with a larger one.

ZFS functions unlike traditional file systems such as ext4. Ars Technica has a good [introduction to ZFS](https://arstechnica.com/information-technology/2020/05/zfs-101-understanding-zfs-storage-and-performance/).

# How

## Assumptions and requirements

- All drives will be formatted. These instructions are not suitable for dual-boot
- No hardware or software RAID is to be used, these would keep ZFS from detecting disk errors and correcting them. In UEFI, set controller mode
to AHCI, not RAID.
- These instructions are specific to UEFI systems and GPT. If you have an older BIOS/MBR system, please use the full instructions linked above

## Initial Ubuntu installation

- Install from an Ubuntu Desktop 20.04 install USB. Ubuntu Server does not offer ZFS boot disk.
- For the "Erase Disk and Install Ubuntu" option, click "Advanced Features" and choose "Experimental ZFS"
- Continue install as normal and boot into Ubuntu

## Add second drive

All work will be done from CLI. Open a Terminal. In the following, use copy & paste extensively, it'll help avoid typos. Right-click in Terminal pastes.

- Update Ubuntu: `sudo apt update && sudo apt dist-upgrade`
- Find the names of your two disks: `ls -l /dev/disks/by-id`. The first disk will have four partitions, the second none.
- Let's set variables for those disk paths so we can refer to them in the following
```
DISK1=/dev/disk/by-id/scsi-disk1
DISK2=/dev/disk/by-id/scsi-disk2
```
- Install tools: `sudo apt install -y gdisk mdadm grub-efi-amd64`

### Create partitions on second drive

- List partitions: `sudo sgdisk -p $DISK1`, you expect to see four of them
- Change swap partition type: `sudo sgdisk -t2:FD00 $DISK1`
- Copy partition table from disk 1 to disk 2: `sudo sgdisk -R$DISK2 $DISK1`
- Change GUID of second disk: `sudo sgdisk -G $DISK2`

### Mirror boot pool

- Confirm that disk 1 partition 3 is the device in the **bpool** by comparing "Partition unique GUID" to the device id shown in zpool status: `sudo sgdisk -i3 $DISK1` and `zpool status`
- Get GUID of partition 3 on disk 2: `sudo sgdisk -i3 $DISK2`
- Add that partition to the pool: `sudo zpool attach bpool EXISTING-UID /dev/disk/by-partuuid/DISK2-PART3-GUID`, for example `sudo zpool attach bpool ac78ee0c-2d8d-3641-97dc-eb8b50abd492 /dev/disk/by-partuuid/8e1830b3-4e59-459c-9c02-a09c80428052`
- Verify with `zpool status`. You expect to see mirror-0 now, which has been resilvered

### Mirror root pool

- Confirm that disk 1 partition 4 is the device in the **rpool** by comparing "Partition unique GUID" to the device id shown in zpool status: `sudo sgdisk -i4 $DISK1` and `zpool status`
- Get GUID of partition 4 on disk 2: `sudo sgdisk -i4 $DISK2`
- Add that partition to the pool: `sudo zpool attach rpool EXISTING-UID /dev/disk/by-partuuid/DISK2-PART4-GUID`, for example `sudo zpool attach rpool d9844f27-a1f8-3049-9831-77b51318d9a7 /dev/disk/by-partuuid/d9844f27-a1f8-3049-9831-77b51318d9a7`
- Verify with `zpool status`. You expect to see mirror-0 now, which either is resilvering or has been resilvered

### Mirror swap

- Remove existing swap: `sudo swapoff -a`
- Remove the swap mount line in /etc/fstasb: `sudo nano /etc/fstab`, find the swap line at the end of the file and delete it, then save with Ctrl-x
- Create software mirror drive for swap: `sudo mdadm --create /dev/md0 --metadata=1.2 --level=mirror --raid-devices=2 ${DISK1}-part2 ${DISK2}-part2`
- Configure it for swap: `sudo mkswap -f /dev/md0`
- Place it into fstab: `sudo sh -c "echo UUID=$(sudo blkid -s UUID -o value /dev/md0) none swap discard 0 0 >> /etc/fstab"`
- Verify that line is in fstab: `cat /etc/fstab`
- Use the new swap: `sudo swapon -a`
- And verify: `swapon -s`

### Move GRUB boot menu to ZFS

- Verify grub can "see" the ZFS boot pool: `sudo grub-probe /boot`
- Create EFI file system on second disk: `sudo mkdosfs -F 32 -s 1 -n EFI ${DISK2}-part1`
- Remove /boot/grub from fstab: `sudo nano /etc/fstab`, find the line for /boot/grub and remove it. Leave the line for /boot/efi in place. Save with Ctrl-x.
- Unmount /boot/grub: `sudo umount /boot/grub`
- Verify with `df -h`, `/boot` should be mounted on `rpool/BOOT/ubuntu_UID`, `/boot/efi` on `/dev/sda1` or similar depending on device name of your first disk, and no `/boot/grub`
- Remove /boot/grub: `sudo rm -rf /boot/grub`
- And create a ZFS dataset for it: `sudo zfs create -o com.ubuntu.zsys:bootfs=no bpool/grub`
- Refresh initrd files: `sudo update-initramfs -c -k all`
- Disable memory zeroing to address a performance regression of ZFS on Linux: `sudo nano /etc/default/grub` and add `init_on_alloc=0` to `GRUB_CMDLINE_LINUX_DEFAULT`, it'll likely look like this: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash init_on_alloc=0"`. Save with Ctrl-x
- Update the boot config: `sudo update-grub` and ignore any errors you may see from osprober
- Install GRUB to the ESP: `sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --recheck --no-floppy`
- Disable grub-initrd-fallback.service: `sudo systemctl mask grub-initrd-fallback.service`. This is the service for /boot/grub/grubenv which does not work on mirrored or raidz topologies. Disabling this keeps it from blocking subsequent mounts of /boot/grub if that mount ever fails.

### Reboot and install GRUB to second disk

- Cross fingers and reboot! `sudo reboot`
- Once back up, open a Terminal again and install GRUB to second disk: `sudo dpkg-reconfigure grub-efi-amd64` , keep defaults and when it comes to system partitions, use space bar to select
first partition on both drives, e.g. /dev/sda1 and /dev/sdb1
- And for good measure signed efi: `sudo dpkg-reconfigure grub-efi-amd64-signed` , this should finish without prompting you
- If you like, you can remove the primary drive and reboot. You expect reboot to take a little longer, and to be successful. `zpool status` should show degraded pools without error


## Replacing a failed drive

- TBD, but in a nutshell, this will involve partitioning it, and telling zpool to replace the failed device (partition) with the new one, ditto for mdadm, and EFI needs to be taken care of.

## Increasing drive space

- Similar to replacing a failed drive, just that partition 4, the rpool partition, will be bigger. Once both drives have been replaced, rpool has the new capacity.
