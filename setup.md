This is an experiment to work with a read-only-root Pop!_OS installation leveraging btrfs with nix and flatpak packages in the user's home directory.

- Install Pop!_OS normally with full disk encryption. DO NOT RESTART or SHUTDOWN at the end.
- At the end of the install, right click the install icon in the dock and choose Quit.
- Open the installer from the dock again and select language and keyboard layout. Then choose Custom (Advanced) to setup partitions
- Click on the first partition, activate Use partition, activate Format, Use as Boot /boot/efi, Filesystem: fat32.
- Click on the second partition, activate Use partition, activate Format, Use as Custom and enter /recovery, Filesystem: fat32.
- Click on the third and largest partition. A Decrypt This Partition dialog opens, enter your luks password and hit Decrypt. A new device is LVM data will be displayed (often at the bottom of the screen). Click on this partition, activate Use partition, activate Format, Use as Root (/) , Filesystem: btrfs.
- Click on the fourth partition, activate Use partition, Use as Swap.
- Choose Erase and Install and follow the instructions. DO NOT RESTART or SHUTDOWN at the end. Sometimes it was necessary to do this a couple times before the install was successful.
- Open a terminal and run the following commands:
- `sudo -i`
- `cryptsetup luksOpen /dev/nvme0n1p3 cryptdata` (Enter the decrypt passphrase)
- `mount -o subvolid=5,defaults,compress=zstd:1,discard=async /dev/mapper/data-root /mnt`
- `btrfs subvolume create /mnt/@`
- `cd /mnt`
- `ls | grep -v @ | xargs mv -t @`
- `btrfs subvolume create /mnt/@var`
- `mv /mnt/@/var/* /mnt/@var/`
- `btrfs subvolume create /mnt/@tmp`
- `btrfs subvolume create /mnt/@srv`
- `btrfs subvolume create /mnt/@nix`
- `btrfs subvolume create /mnt/@home`
- `mv /mnt/@/home/* /mnt/@home/`
- `btrfs subvolume list /mnt` will show the volumes
- Changes to fstab
- `sed -i 's/btrfs  defaults/btrfs  defaults,subvol=@,compress=zstd:1,discard=async/' /mnt/@/etc/fstab`
- `echo "UUID=$(blkid -s UUID -o value /dev/mapper/data-root)  /var  btrfs  defaults,subvol=@var,compress=zstd:1,discard=async   0 0" >> /mnt/@/etc/fstab`
- `echo "UUID=$(blkid -s UUID -o value /dev/mapper/data-root)  /tmp  btrfs  defaults,subvol=@tmp,compress=zstd:1,discard=async   0 0" >> /mnt/@/etc/fstab`
- `echo "UUID=$(blkid -s UUID -o value /dev/mapper/data-root)  /srv  btrfs  defaults,subvol=@srv,compress=zstd:1,discard=async   0 0" >> /mnt/@/etc/fstab`
- `echo "UUID=$(blkid -s UUID -o value /dev/mapper/data-root)  /nix  btrfs  defaults,subvol=@nix,compress=zstd:1,discard=async   0 0" >> /mnt/@/etc/fstab`
- `echo "UUID=$(blkid -s UUID -o value /dev/mapper/data-root)  /home  btrfs  defaults,subvol=@home,compress=zstd:1,discard=async   0 0" >> /mnt/@/etc/fstab`
- `cat /mnt/@/etc/fstab` should look like the below but with your device UUID:

```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system>  <mount point>  <type>  <options>  <dump>  <pass>
PARTUUID=41fda1ef-db7c-4158-b757-249aee246487  /boot/efi  vfat  umask=0077  0  0
PARTUUID=e1e0a703-8eea-460e-8e6d-16fe35da7abe  /recovery  vfat  umask=0077  0  0
/dev/mapper/cryptswap  none  swap  defaults  0  0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /  btrfs  defaults,subvol=@,compress=zstd:1,discard=async  0  0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /home  btrfs  defaults,subvol=@home,compress=zstd:1,discard=async   0 0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /var  btrfs  defaults,subvol=@var,compress=zstd:1,discard=async   0 0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /tmp  btrfs  defaults,subvol=@tmp,compress=zstd:1,discard=async   0 0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /srv  btrfs  defaults,subvol=@srv,compress=zstd:1,discard=async   0 0
UUID=9e42b5a3-2cf1-4c8a-bb09-4f1a8dab1dc5  /nix  btrfs  defaults,subvol=@nix,compress=zstd:1,discard=async   0 0
```

- Next changes to crypttab
- `sed -i 's/luks/luks,discard/' /mnt/@/etc/crypttab`
- And then kernelstub
- `nano /mnt/@/etc/kernelstub/configuration`
- Here you need to add rootflags=subvol=@ to the "user" kernel options. That is, your configuration file should look like this:
- `cat /mnt/@/etc/kernelstub/configuration`
- Make sure to add the comma after "splash",= like below.

```
# {
#   "default": {
#     "kernel_options": [
#       "quiet",
#       "splash"
#     ],
#     "esp_path": "/boot/efi",
#     "setup_loader": false,
#     "manage_mode": false,
#     "force_update": false,
#     "live_mode": false,
#     "config_rev":3
#   },
#   "user": {
#     "kernel_options": [
#       "quiet",
#       "loglevel=0",
#       "systemd.show_status=false",
#       "splash",
#       "rootflags=subvol=@"
#     ],
#     "esp_path": "/boot/efi",
#     "setup_loader": true,
#     "manage_mode": true,
#     "force_update": false,
#     "live_mode": false,
#     "config_rev":3
#   }
# }
```

- `mount /dev/nvme0n1p1 /mnt/@/boot/efi`
- `sed -i 's/splash/splash rootflags=subvol=@/' /mnt/@/boot/efi/loader/entries/Pop_OS-current.conf`
- Optionally add a timeout for the systemd boot menu for the recovery partition
- `echo "timeout 3" >> /mnt/@/boot/efi/loader/loader.conf`
- Update initramfs
- `cd /`
- `umount -l /mnt`
- `mount -o subvol=@,defaults,compress=zstd:1,discard=async /dev/mapper/data-root /mnt`
- `for i in /dev /dev/pts /proc /sys /run; do mount -B $i /mnt$i; done`
- `chroot /mnt`
- `mount -av`
- `update-initramfs -c -k all`
- If all went well `reboot`

Next, log into your user account, update Pop!_OS, install COSMIC DE, enable wayland in GDM, set Root's home to /var/root/, install Nix Packages and set the root partition to read-only.

- `sudo apt update && sudo apt full-upgrade`
- `sudo apt install cosmic-*`
- `sudo nano /etc/gdm3/custom.conf`
- Change `WaylandEnable=false` to `WaylandEnable=true`
- `sudo nano /etc/passwd`
- Change `root:x:0:0:root:/root:/bin/bash` to `root:x:0:0:root:/var/root:/bin/bash`
- `sudo mkdir /var/root`
- `reboot` and login again
- Install Nix Packages in multi-user mode
- `sh <(curl -L https://nixos.org/nix/install) --daemon`
- Set root to read only
- `sudo btrfs property set -ts / ro true` (Change to "false" to run updates with apt or modifiy files in root then reset to "true").
- Use `btrfs property get -ts /` to check the read only status.

Install flatpaks with `flatpak install $application`. Install nix packages with 'nix-env -i $application'.

There are some caveats. Settings > Users will not work because with read-only root. CUPS also doesn't work for printing. More caveates may still exist with samba. For more info see https://wiki.debian.org/ReadonlyRoot.

These instructions were adapted from here https://mutschler.dev/linux/pop-os-btrfs-22-04/#changes-to-fstab. Thanks for the excellent write-up.
