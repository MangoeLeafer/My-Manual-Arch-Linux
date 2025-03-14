# Initial setup

## Setting console keyboard layout/font

Load the US keymap and set a font that’s commonly used in consoles.

```
# loadkeys us
# setfont lat9w-16
```

## Verify boot mode

Running the command below should return 64.

```
# cat /sys/firmware/efi/fw_platform_size
...
```

## Connect to the internet

Run the commands below to get the network devices currently available and to make sure that is not blocking the wireless network card. This part is going to strictly cover WiFi.

```
# ip link
...
# rfkill
...
```

Then, if the wireless card is soft-blocked, run the following command.

```
# rfkill unblock wlan
```

Now, using **‘iwctl’**, run the commands below to connect to a network wirelessly.

```
# iwctl
[iwd]# device list
...
[iwd]# device name set-property Powered on
[iwd]# station name scan
[iwd]# station name get-networks
[iwd]# station name connect SSID
...(will be prompted to enter passphrase if required.)
[iwd]# exit
```

Then, with the command below, confirm that the connection is valid.

```
# ping -c 5 archlinux.org
...
```

## Updating system clock

Use the command below to make sure that the system clock is synced.

```
# timedatectl
...
```

# Disk preparation

## Partitioning the disk

First, check to see what disks and partitions are currently available with the command below.

```
# lsblk
...
```

You should find a list of devices that would be accessible as **‘/dev/sda’** or **‘/dev/nvme0n1’** or **‘/dev/mmcblk0’**. Because our setup most likely has an SSD ready for installing the OS on, we’re going to use **‘/dev/nvme0n1’** as the primary drive to partition. Other drives can be partitioned later after all installation is done.

First, we’ll enter the **‘fdisk’** environment to create new partitions for our **‘/dev/nvme0n1’** with the command below.

```
# fdisk /dev/nvme0n1
```

We will create 4 partitions: _**EFI system**_, _**Linux swap**_, _**Linux x86-64 root**_, and _**Linux home**_. The EFI system partition is going to be 1GB which is going to act as a storage for UEFI bootloaders, applications, and drivers to be launched by the UEFI firmware. The Linux swap partition will be [RAM size]GB so that in hibernation mode RAM can be stored in this partition. Next, the Linux x86-64 root is going to take up 100GB, and is going to be used to store the OS and be the root directory for our system. Then, the rest can be dedicated to the Linux home partition.

First, we’re going to type **‘g’** and press enter to set up a GUID Partition Table (GPT). This is so we actually have something to part with.

```
g
```

Next, we’re going to create our partitions with these commands. First is the EFI system.

```
n
... [press enter for default partition number]
... [press enter for default partition starting sector]
+1G
t
... [press enter for default partition]
1
```

Next is the Linux swap. Replace XX with your RAM space.

```
n
... [press enter for default partition number]
... [press enter for default partition starting sector]
+XXG
t
... [press enter for default partition]
19
```

Then the Linux x86-64 root is next.

```
n
... [press enter for default partition number]
... [press enter for default partition starting sector]
+100G
t
... [press enter for default partition]
23
```

And finally, the Linux home partition is next. The rest of the drive will be dedicated to this.

```
n
... [press enter for default partition number]
... [press enter for default partition starting sector]
... [press enter for default partition ending sector]
t
... [press enter for default partition]
42
```

After creating the needed partitions, then we can check our changes using the command below, which will show the created partitions.

```
p
...
```

Then, after confirming the correct partitions have been made, then we can write these changes to the disk and exit via this command below.

```
w
```

## Formatting the disk

Now that we have the disk partitioned, each partition can be referred to as a device.

- /dev/nvme0n1p1 : _**EFI system**_
- /dev/nvme0n1p2 : _**Linux swap**_
- /dev/nvme0n1p3 : _**Linux x86-64 root**_
- /dev/nvme0n1p4 : _**Linux home**_

So, the next step is to format each partition with an appropriate file system. For the root and home partition, they are going to each use an _Ext4 file system_. This can be done as shown below.

```
# mkfs.ext4 /dev/nvme0n1p3
# mkfs.ext4 /dev/nvme0n1p4
```

Next, you need to format the swap partition as well.

```
# mkswap /dev/nvme0n1p2
```

And then finally, the EFI system partition.

```
# mkfs.fat -F 32 /dev/nvme0n1p1
```

## Mounting the file systems

The next step is to mount the file systems. This is an important step because it makes the data within these file systems accessible so that we can make changes to it. It’ll also get our swap and EFI partitions prepared as you’ll see soon.

First, let’s mount the root, boot, and home volumes in that respective order.

```
# mount /dev/nvme0n1p3 /mnt
# mount --mkdir /dev/nvme0n1p1 /mnt/boot
# mount --mkdir /dev/nvme0n1p4 /mnt/home
```

Then, we’ll also enable the swap volume.

```
# swapon /dev/nvme0n1p2
```

The reason why everything is currently mounted to **‘/mnt’** is because this is a temporary mount. Once we run **‘genfstab’** later and finish the Arch Linux installation, then these mounted file systems will be detected and will be transferred to the true root. Essentially, all things **‘/mnt’** will be shifted over to **‘/’**.

# Installing the essential packages

Now that we have all our files systems properly mounted, we can begin installing the essential packages for Arch. However, before we do, it would be a wise idea to select the best mirrors to download from. A mirror is just another word for a place that you can download files from online. These will be defined on the file **‘/etc/pacman.d/mirrorlist’**.

So in order to do that, we’ll use **‘reflector’** to get the fastest 10 HTTPS mirrors from the United States and save it into the list as shown below.

```
# reflector --country US --latest 10 --protocol https --sort rate --connection-timeout 15 --save /etc/pacman.d/mirrorlist
```

Then, if there are no serious errors, then we can begin installing the packages we want.

THIS IS IMPORTANT! It will remove a lot of hassle later if we just install some important packages now. A list below shows some packages that should be downloaded along with the essential packages.

- base-devel : base development packages
- git : install git
- e2fsprogs : Ext2/3/4 filesystem utilities
- grub : the bootloader
- efibootmgr : needed to install grub
- amd-ucode : microcode updates for the CPU (use intel-ucode for Intel CPUS)
- nano : file editor
- networkmanager : manages internet connections wired/wireless
- pipewire
- pipewire-alsa
- pipewire-pulse
- pipewire-jack : for the new audio framework replacing pulse and jack
- wireplumber : the pipewire session manager
- reflector : to manage mirrors for pacman
- zsh : shell
- zsh-completions : zsh additional completions
- zsh-autosuggestions : zsh autocorrect
- openssh : to use ssh and manage keys
- man : for manual pages
- sudo : to run commands as other users

```
# pacstrap -K /mnt base linux linux-firmware base-devel git e2fsprogs grub efibootmgr amd-ucode nano networkmanager pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber reflector zsh zsh-completions zsh-autosuggestions openssh man sudo
```

Congrats! If you did everything right, these packages should have been downloaded with no error messages. It’s good to keep note that for right now, these downloaded packages are being temporarily stored in a cache, **‘/var/cache/pacman/pkg’** and are installed into the target system, **‘/mnt’**. After the full installation is done, then this same cache directory will be used in the newly installed system.

# Configuring the system

Now that we have all of these packages, we need to actually tell the computer how to mount each disk partition, filesystems, and devices into the directory tree. Just like we said earlier when mounting, the computer will need an **‘fstab’** file to do this, which is generated by **‘genfstab’**. As a system configuration file, these are basically the instructions that tell the computer how to access each partition/device, make sure that they function properly, and automates their mounting during boot.

To generate an ‘fstab’ file that is defined by Universally Unique Identifiers (UUIDs), run the command below.

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

Now, we’re ready to change the root of our environment into the new system that we’ve prepared up until this point. Do this using **‘arch-chroot’**.

```
# arch-chroot /mnt
```

## Configuring time

Here, we can set the time zone. I use America and New York here because I live in EST.

```
# ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

And then, we run **‘hwclock’** to generate **‘/etc/adjtime’**.

```
# hwclock --systohc
```

While we’re at it, we should also set up time synchronization using a Network Time Protocol (NTP) to prevent clock drift and ensure accurate time. We’re going to use the NTP client, **‘systemd-timesyncd’** to do this by editing its configuration file first.

```
# nano /etc/systemd/timesyncd.conf
```

Then, uncomment the **‘NTP’** line and the **‘FallbackNTP’** line, and then edit those lines to look like this.

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.fr.pool.ntp.org
...
```

Then verify your configuration.

```
# timedatectl show-timesync --all
...
```

Now, we can go ahead and enable the time syncing service.

```
# systemctl enable systemd-timesyncd.service --now
```

And you can check the status of it by running the command below.

```
# timedatectl status
...
```

## Localization

Now, we need to configure our localization. To do this, we need to first edit **‘/etc/locale.gen’** and uncomment **‘en_US.UTF-8 UTF-8’**, and other needed UTF-8 locales. You should already know how to do this by now (using nano). Next you will generate the locales by running the command below.

```
# locale-gen
```

After, create the **‘/etc/locale.conf’** using **‘nano’** and set the LANG variable to what is shown below.

```
LANG=en_US.UTF-8
```

We’ll also go ahead and set the keymap to US, editing **‘/etc/vconsole.conf’** with **‘nano’** and setting the KEYMAP variable to what is shown below.

```
KEYMAP=us
```

## Finalizing network configuration

Afterwards, we can complete the network configuration by creating the hostname file, **‘/etc/hostname’** with **‘nano’** and then fill the file with only the hostname. The only content in this file will be the hostname and that is it. We should also enable the network manager service at this time so that it can automatically connect to any available connections that have already been configured upon start-up.

```
# systemctl enable NetworkManager.service
```

## Root password and boot loader

Nearing the finish of installation here, what we need to do next is set a root password and install a Linux-capable boot loader. To set the root password, run this.

```
# passwd
...
```

Then, since we installed **‘grub’** and **‘efibootmgr’** earlier as our packages, we can install GRUB as our boot loader like this.

```
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
# grub-mkconfig -o /boot/grub/grub.cfg
```

The last line is us generating the main configuration file, as GRUB requires us to.

## Reboot and installation finalization

Alright, we’re almost done, all we have to do now is to exit from **‘chroot’**, unmount everything, and then reboot the system and unplug the installation media. Run this before unplugging.

```
# exit
# umount -R /mnt
# reboot
```

After rebooting, you should be able to see the Arch terminal in front of you. Congratulations! This means you have successfully installed Arch Linux! If you don’t, restart the computer and load up the boot menu to enter the GRUB boot loader to try and launch it there. To login, your login is, **‘root’**, and your password was what you set earlier as your root password.

# Post installation

## Package updating and internet

After you log in, you might try to update all your packages like so.

```
# pacman -Syu
```

But then you might notice that you can’t connect to the download mirrors! It’ll throw a bunch of errors at you, but don’t freak out. We simply need to reconnect to our network. This can be done with **‘nmcli’**. For WiFi, you can run this command to find networks nearby.

```
$ nmcli device wifi list
```

Then to connect to the network, do this.

```
$ nmcli device wifi connect SSID_or_BSSID password password
```

And then verify with this.

```
$ nmcli device
```

Now you should be able to update all your packages just fine! Also, because earlier we enabled the network manager service, the system should reconnect to the internet even after reboot.

## Adding users

The next we should do is to add a user, or users. With the new installation we just made, the only account you can access is the root one, otherwise known as the _**superuser**_. However, logging in as root for long periods of time is insecure. The reason being, you essentially expose all of your storage to anybody who has access to your computer. Any extraneous access to this superuser means total compromise of your system.

To add a basic user, we can run this command below.

```
# useradd -m jackie
```

The **'-m'** in the command tells our system to create a home directory for our user. This would look something like, **'/home/jackie'**. And to add a password for our user, we can run this below.

```
# passwd jackie
...
```

This will then prompt you to make a password for the user. Also note that the **'useradd'** command will automatically create a user group with the same name as the user. However, since I am not using this computer as part of a multi-user system, I won't be getting into user groups. For that, you can search that up on the Arch Linux Wiki.

Now, you can reboot and sign into the user account you just made. Once you have done that, we can move onto securing our system.

## Security

Securing the Linux system is important. Many people argue that because Linux isn't as widely used as other operating systems like Windows, that it doesn't need to be secured. HOWEVER, there does exist Linux malware and security concerns that I would personally deem important to counter.

The first thing we're going to do is to install an anti-virus and configure it. The most common one that's used is ClamAV. To install it, we'll use **pacman** to install the package.

```
$ sudo pacman -S clamav
...
```

You also might notice that we're using **'sudo'** as part of our commands now, and that is because of permission levels. Because we're not a superuser anymore, we need to request permission (act as a root user) before installing packages or data into the root directory. This will prompt you to enter the *USER*, not root, password in order to run the command.

Now let's go ahead and configure ClamAV. First, check that the clamconf configuration files have been made.

```
$ clamconf
```

If not, then go ahead and create them.

```
$ clamconf -g freshclam.conf > freshclam.conf
$ clamconf -g clamd.conf > clamd.conf
$ clamconf -g clamav-milter.conf > clamav-milter.conf
```

After that, we're going to go ahead and set up real-time protection. This is going to use on-access scanning. Any files being read, written to, or executed will be scanned prior, and then this can be configured to either notify on detection or just prevent/block. We are going to use prevention. This scanning requires the kernel to be compiled with the **fanotify** module, which our kernel already has.

To configure on-access scanning, we're going to do this by editing the **'/etc/clamav/clamd.conf'** file.

```
# Exclude the UID of the scanner itself from checking, to prevent loops
OnAccessExcludeUname clamav

# WARNING: for security reasons, clamd should NEVER run as root.
# Previous instructions in this wiki included this line, remove it:
# User root    # REMOVE THIS
# Add this instead:
User clamav
```

We will also go ahead and set this on-access scanner into prevention mode (still editing the same file).

```
# Add some directories to scan and not to scan.
OnAccessIncludePath /home
OnAccessIncludePath /tmp
OnAccessIncludePath /var/tmp
OnAccessIncludePath /mnt
OnAccessIncludePath /media

# Prevention doesn't work with OnAccessMountPath.
# It works with OnAccessIncludePath, as long as /usr and /etc are not included.
# Including /var while activating prevention is also not recommended, because
# this would slow down package installation by a factor of 1000.
# However, since we don't have any of those, we should be fine.
OnAccessPrevention yes

# Perform scans on newly created, moved, or renamed files
OnAccessExtraScanning yes

# Exclude root-owned processes
OnAccessExcludeRootUID true
```

We're not going to create any notification popups yet as that can be configured in the future. Instead, we're just going to go ahead and enable the needed ClamAV services.

```
$ sudo systemctl enable clamav-daemon.service
$ sudo systemctl enable clamav-clamonacc.service
$ sudo systemctl start clamav-daemon.service
$ sudo systemctl start clamav-clamonacc.service
```

First command enables the main ClamAV daemon, and the second one is for real-time on access protection. To check the logs for any scanned malware, run the command below (there should be nothing).

If you ever want to check if a service is running, you can run this.

```
$ sudo systemctl status *service*
...
```

Just replace the **'service'** with the service you're looking for and it'll tell you if it is active, inactive, or failed.

```
$ tail /var/log/clamav/clamonacc.log
```

Additionally, reading the **'/etc/clamav/clamd.conf'** file can let you see the other log files that are being written to, which you can read the logs for with the same command above, just with a different directory.

The next thing we will do is install rootkit protection. Rootkits are one of the most dangerous malwares that GNU/Linux users could encounter because it can enable an unauthorized user to gain control of a computer system without being detected. So for us, we're going to be installing **'rkhunter'**.

```
$ sudo pacman -S rkhunter
...
```

Then to setup, we do this.

```
$ sudo rkhunter --propupd
```

Then, we're going to edit the configuration file to white-list some files to prevent false positives due to core utilities that have been replaced by scripts. Using **'nano'**, edit **'/etc/rkhunter.conf'**.

```
SCRIPTWHITELIST=/usr/bin/egrep
SCRIPTWHITELIST=/usr/bin/fgrep
SCRIPTWHITELIST=/usr/bin/ldd
SCRIPTWHITELIST=/usr/bin/vendor_perl/GET
```

Now, whenever you need to run a check, do this.

```
$ sudo rkhunter --check
```

And to update the data files to keep up-to-date (which you should do regularly, but can automate), run this.

```
$ sudo rkhunter --update
```

In summary, we've installed a basic anti-virus software alongside a rootkit protection software to protect our system to a pretty high standard. Adding an additional firewall to protect against network threats is also another security measure we can add, but we can do that later down the line in a more cleaner non-terminal fashion. Also, because I'm using a desktop that's connected to a WiFi router, the router itself already has a firewall to protect me, so there isn't a dire need for one, unlike if you're using a laptop that will connect to unsecure or public networks.

## AUR and additional packages and drivers

Now we can begin installing some additional packages and drivers. Please note that you are not only limited to these packages and those that we installed earlier. Arch Linux and Linux systems in general work off of the installation of packages, as the use of packages are essential to customizing one's Linux build to function the way they want it to.

However, some of these packages we're going to be downloading are going to be located in something called the Arch User Repository (AUR). Up until now, we've been using **'pacman'** to download our packages. All the packages on there are supported by Arch Linux developers and trusted packagers. However, the AUR is different in that the packages there are user-made and community-maintained. To install packages from the AUR, we're going to install an AUR helper, **'yay'**.

```
$ sudo pacman -S yay
```

And to install an AUR package, we do this.

```
$ yay -S *package-name*
```

And to update them, do this.

```
$ yay -Syu
```

This is set up similar to the default **'pacman'** package manager, so most of the commands that you would normally run on that should map just fine with **'yay'**.

Since I want to run my system through a graphical environment, we need to install video drivers. Because I'm using an AMD GPU, I'm going to install the open source AMD driver, **'xf86-video-amdgpu'**, 3D support, **'mesa'**, and Vulkan support, **'vulkan-radeon'**.

```
$ sudo pacman -S xf86-video-amdgpu mesa vulkan-radeon
```

We don't need to download any sound drivers because the Linux kernel already has one built into it, ALSA. We installed PipeWire earlier when installing the OS, which works on top of this.

And for fun, we'll also install **'neofetch'** to display system information in a console.

```
$ sudo pacman -S neofetch
```

## Desktop environment

I'm going to be constructing my own desktop environment, with at least these features first.

- Window manager
- Status bar
- Application launcher
- Lockscreen
- Logout/shutdown menu

For my window manager, I'm going to be using Hyprland so we can simply install it using the command below.

```
$ sudo pacman -S hyprland
```

We're also going to go ahead and install a terminal emulator, **'kitty'**.

```
$ sudo pacman -S kitty
```

Then to install the application launcher, lockscreen, and status bar we'll run this.

```
$ sudo pacman -S wofi waybar swaylock
```

After, we can install the logout/shutdown menu.

```
$ yay -S wlogout
```

## Display manager

A display manager is essentially a login screen that appears at boot. The one we're going to be using is SDDM, which works flawlessly with Hyprland. To install it we can run this.

```
$ sudo pacman -S sddm
```

And to enable it so that it can start on boot, we'll do this.

```
$ sudo systemctl enable sddm
```

Finally, we can reboot our system and see if it works. You should be greeted with the SDDM login screen, and once you login you should see your desktop environment. If you do, congratulations! You have just set up your basic graphical environment!

# Finishing touches

## System maintenance