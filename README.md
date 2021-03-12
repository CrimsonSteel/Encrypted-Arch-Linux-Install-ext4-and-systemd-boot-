This is only a guide to assist you. Install at your own risk and see https://wiki.archlinux.org/ if you get lost. I am not responsible for your machine or installation.

Let's proceed!

Create bootable media ----- download newest Arch iso here: https://archlinux.org/download/

Open a terminal and use the following command inserting the paths needed on your machine

		dd bs=4M if=path/to/.iso of=/dev/sdx status=progress oflag=sync
	
 
Wipe disk Arch is to be installed on, if necessary

		cryptsetup open --type plain /dev/nvme0n1 container
		dd if=/dev/zero of=/dev/mapper/container
		cryptsetup luksClose container
 
For wireless internet otherwise skip

		iwctl
			device list
			station "device" scan
			station "device" get-networks
			station "device" connect "SSID"
			passphrase
		exit
 
Get the fastest mirrors for your area

		reflector --country "United States" --age 12 --sort rate --save /etc/pacman.d/mirrorlist
		
		pacman -Syu
 
Set time

		timedatectl set-ntp true
 
List drives (to see name of drive(s) to use for installation)

		lsblk 
 
Partition your drive

		fdisk /dev/nvme0n1
(USE YOUR DRIVE...an nvme drive will be shown throughout this guide as an example only!)

		g
		n (last sector) +256M
		t (partition type) 1
		n (defaults)
		w
 
Setup encryption

		modprobe dm-crypt 
(to verify dm-crypt is installed...no error message means it's installed.)

		cryptsetup benchmark 
(aes-xts is used by default...choose wisely if you decide to change this...RTFM!)

		cryptsetup luksFormat /dev/nvme0n1p2
(create a passphrase)

		cryptsetup luksOpen /dev/nvme0n1p2 lvm
(passphrase you just created)
 
Create Logical Volumes

		pvcreate /dev/mapper/lvm
		vgcreate main /dev/mapper/lvm
		lvcreate -L 2GB -n swap main
		lvcreate -l 100%FREE -n root main
 
Create filesystems and mount
		
		mkfs.vfat -n BOOT /dev/nvme0n1p1
		mkfs.ext4 -L root /dev/mapper/main-root
		mkswap -L swap /dev/mapper/main-swap
		mount /dev/mapper/main-root /mnt
		mkdir /mnt/boot
		mount /dev/nvme0n1p1 /mnt/boot
 
Install base system

		pacstrap /mnt base base-devel syslinux nano linux linux-firmware mkinitcpio lvm2 dhcpcd inetutils intel-ucode
 
Install syslinux and append
		
		syslinux-install_update -i -a -m -c /mnt
		
		nano /mnt/boot/syslinux/syslinux.cfg
(edit both arch and fallback)

		APPEND cryptdevice=/dev/nvme0n1p2:main root=/dev/mapper/main-root rw
 
		swapon -L swap
 
Generate fstab

		genfstab -U -p /mnt >> /mnt/etc/fstab
		
		nano /mnt/etc/fstab
(when using an ssd or nvme change "relatime" to "noatime)
 
Log into new system

		arch-chroot /mnt
 
Create language and location defaults

		echo LANG=en_US.UTF-8 > /etc/locale.conf
		
		nano /etc/locale.gen
(uncomment en_US.UTF-8, or whatever your language is, and save)

		locale-gen
 
		ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
(your local zoneinfo should be used...RTFM!)
 
New system edits
		
		nano /etc/pacman.conf
(uncomment lines "Color" and all "multilib" lines near bottom of pacman.conf if you want color in pacman and in order to install 32bit software)

		pacman -Sy
 
		nano /etc/mkinitcpio.conf
(edit as below adding "encrypt" and "lvm2")

		HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems fsck)

Generate mkinit		
		
		mkinitcpio -p linux

Set root password

		passwd
 
Create computer hostname
	
		hostnamectl set-hostname #your hostname
		echo "127.0.0.1 (your hostname)" >> /etc/hosts
		echo "(your hostname)" > /etc/hostname
		echo "127.0.0.1 localhost" > /etc/hostname
		echo "::1       localhost" >> /etc/hostname
		echo "127.0.0.1 (your hostname).localhdomain (your hostname)" >> /etc/hostname
(verify hostname is set correctly)

		ping -c 3 (your hostname)
(if it pings, your good to go)
 
Install systemd-bootloader
		
		bootctl --path=/boot install
		
		nano /boot/loader/loader.conf
(edit as below)

		default arch.conf
		timeout 3
		console-mode max
		editor no
 
Find the drive where root is installed

		blkid
		
Grab the UUID for that drive (usually the first UUID listed) and add it to /boot/loader/entries/arch.conf with the following command

		blkid |grep nvme0n1p2 |cut -d'"' -f 2 >> /boot/loader/entries/arch.conf
 
		nano /boot/loader/entries/arch.conf
(edit as below)

    		title   Arch Linux
    		linux   /vmlinuz-linux
		initrd  /intel-ucode.img
    		initrd  /initramfs-linux.img
    		options cryptdevice=UUID=(your UUID we copied from the command above):lvm root=/dev/mapper/main-root quiet rw
 
Install some more packages based on need

		pacman -S networkmanager network-manager-applet dialog wpa_supplicant gvfs gvfs-smb gvfs-nfs gvfs-mtp ntfs-3g gtk3 openssh rsync dnsutils nfs-utils mtools dosfstools polkit-gnome linux-headers bluez bluez-utils cups xdg-utils xdg-user-dirs alsa-utils pulseaudio pulseaudio-bluetooth xorg git wget reflector
 
Enable systemd autostarts

		systemctl enable NetworkManager
		systemctl enable dhcpcd
		systemctl enable bluetooth
		systemctl enable sshd
		systemctl enable reflector.timer
		systemctl enable fstrim.timer
		systemctl enable cups
 
Add user

		useradd -mG wheel (your username)
		passwd (your username)
 
		nano /etc/sudoers
(uncomment %wheel near bottom of sudoers file)
 
		exit
		umount -R /mnt
		reboot
 
-------POST INSTALL-------

Login with your username and password

on wireless connection type "nmtui" and connect to your wireless network
 
		sudo timedatectl set-ntp true
 
		sudo reflector --country "United States" --age 12 --sort rate --save /etc/pacman.d/mirrorlist
		
		sudo pacman -Syu
 
Install paru for easy AUR access

		sudo git clone https://aur.archlinux.org/paru.git
		sudo chown -R (your username):(your username) ./paru
		cd paru
		makepkg -si
		sudo paru -Syu
(now you can use paru and search the AUR!)
 
If you have an Nvidia card this is optional
 
		sudo pacman -S nvidia nvidia-utils nvidia-settings
		
reboot
 
This will get you started...install whatever DE or WM you like per the Arch Wiki and enjoy :) 
