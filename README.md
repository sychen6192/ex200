# EX200 
## Chapter1 

### Bash basic
#### Command execution path
```shell=
$ which hello
~/bin/hello
```
#### PATH environment variable
```shell=
echo $PATH
```

#### 強引號弱引號
```shell=
$ echo '# not a comment #' # 強引號，使得特殊符號沒意義
$ var=$(hostname -s); echo $var
MacBook-Pro
$ echo "hello $(hostname -s)"
hello MacBook-Pro
$ echo 'hello $(hostname -s)'
hello $(hostname -s)
$ echo '"Hello, world"'
"Hello, world"
```

#### Output in Shell Script
```shell=
$ cat hello
#!/bin/bash

echo "Hello, world"
echo "ERROR: Houston, we have a problem." >&2 
$ bash hello 2> hello.log
Hello, world
$ cat hello.log
ERROR: Houston, we have a problem.
```

#### append or overwrite
```shell=
$ echo "hello1" > aaa.txt && cat aaa.txt
hello1
$ echo "hello2" > aaa.txt && cat aaa.txt
hello2
$ echo "hello3" >> aaa.txt && cat aaa.txt
hello2
hello3
```

#### error code
```shell=
$ test 1 -gt 0; echo $?
0
$ test 1 -gt 2; echo $?
1
# new syntex
$ [ 1 -eq 1 ]; echo $?
0
$ [ 1 -eq 3 ]; echo $?
1
# unary
# emptystring check
$ STRING=''; [ -z "$STRING" ]; echo $?
0
# nonzero check
$ STRING='abc'; [ -n "$STRING" ]; echo $?
0
```

#### file description
```shell=
0 代表鍵盤輸入
1 代表螢幕輸出
2 代表錯誤輸出

$ systemctl is-active psacct > /dev/null 2>&1
把標準出錯重定向到標準輸出,然後扔到/DEV/NULL下面去。
```



# SELinux
#### display SELinux contexts
```shell=
ps axZ # -Z option
ps -ZC httpd #specify service
ls -Z　/var/www # specify path
```
#### changing mode
```shell=
getenforce
setenforce
setenforce [0 | 1] # enable=1 
getenforce
```
> we can also configure settings from /etc/selinux/config

#### controlling selinux file contexts
```shell=
# changing context of a file
mkdir /virtual
ls -Zd /virtual
chcon -t httpd_sys_content_t /virtual
ls -Zd /virtual
restorecon -v /virtual
# define rules
semanage fcontext [-a|-d|-l|-m]
semanage fcontext -a -t httpd_sys_content_t:s0 '/virtual(/.*)?' # add rule
restorecon -Rv /virtual # apply the rule above
# adjusting selinux policy w/ boolean
getsebool -a
getsebool httpd_enable_homedirs
setsebool httpd_enable_homedirs on # -P for persistent
semanage boolean -l | grep 'httpd_enable_homedirs'
getsebool httpd_enable_homedirs
# investigating SELinux issues
tail /var/log/audit/audit.log
tail /var/log/messages
sealert -l | grep <uuid>
ausearch -m AVC -ts recent (type=AVC, ts=time)
```

# Manage basic storage
creating File Systems => add FS => Mount FS
### managing partition w/ parted
```shell=
# display partition table
parted <disk_name> # in powers of 10
# writing partition table
parted <disk_name> mklabel [msdos|gpt]
# creating MBR partition (interation)
parted <disk_name>
mkpart
primary
xfs # parted /dev/vdb help mkpart
2048s
1000MB
# size = 1000MB-2048s
quit
udevadm settle
# delete partition
parted <disk_name>
print # check disk number
rm <disk_number>
# MBR partition in one shot
parted /dev/vdb mkpart primary xfs 2048s 1000MB
# GPT partition in one shot
parted /dev/vdb mkpart userdata xfs 2048s 1000MB
```

```shell=
# creating FS
mkfs.xfs <partition_name>
# mounting FS
mount <partition_name> /mnt
# display mounting status
mount | grep 'nvme0n1p3'
# mounting FS persistently (preferred UUID)
## get uuid
lsblk --fs
"UUID=<uuid> /mnt xfs defaults 0 0" >>　/etc/fstab
```

### Managing Swap space
```shell=
# creating a swap space
# 1. define size
# 2. set to linux-swap
parted <disk_name>
print
mkpart
swap1 # partition name
linux-swap # FS type
1000MB
1257MB
print
q
udevadm settle
# formatting the device (applies a swap signature)
mkswap <partition_name>
# Inspect
free
spapon --show
# Activating a swaping space
swapon <partition_name>
swapoff <partition_name>
# Swap partition in one shot
parted /dev/vdb mkpart myswap linux-swap 1001MB 1501MB

"UUID=<uuid> swap swap defaults 0 0" >>　/etc/fstab
# swap space priority (larger number first)
"UUID=<uuid> swap swap pri=4 0 0" >>　/etc/fstab
"UUID=<uuid> swap swap pri=10 0 0" >>　/etc/fstab
> wiping
> dd if = /dev/zeor bs=1M count=20 of=/dev/vdb
```
# LVM
```shell=
# prepare physical device
parted -s /dev/vdb mkpart primary 1MiB 769MiB
parted -s /dev/vdb mkpart primary 770MiB 1026MiB
parted -s /dev/vdb set 1 lvm on
parted -s /dev/vdb set 2 lvm on
# create pv
pvcreate /dev/vdb2 /dev/vdb1
# create vg
vgcreate vg01 /dev/vdb2 /dev/vdb1 # 2pv in 1vg
# create lv
lvcreate -n lv01 -L 700M vg01
# add FS
mkfs.xfs /dev/vg01/lv01
# mount
mkdir /mnt/data
"/dev/vg01/lv01 /mnt/data xfs defaults 1 2" >> /etc/fstab
mount /mnt/data
# removing LVM
# need to comment the mount point on /etc/fstab
umount /mnt/data
lvremove /dev/vg01/lv01
vgremove vg01
pvremove /dev/vdb02 /dev/vdb01
```
#### Inspect LVM
```shell=
pvdisplay /dev/vdb01
vgdisplay vg01
lvdisplay /dev/mapper/vg01-lv01
```
#### Extending VG
```shell=
parted -s /dev/vdb mkpart primary 1027MiB 1539MiB
parted -s /dev/vdb set 3 lvm on
pvcreate /dev/vdb3
vgextend vg01 /dev/vdb3
```
#### Extending LV & FS
```shell=
vgdisplay vg01 # check free space
lvextend -L +300M /dev/vg01/lv01
xfs_growfs /mnt/data
# resize2fs /dev/vg01/lv01 => ext4
df -h /mountpoint
```
```shell=
# extending swap space
vgdisplay vgname
swapoff -v /dev/vgname/lvname
lvextend -l +extents /dev/vgname/lvname
mkswap /dev/vgname/lvname
swapon -va /dev/vgname/lvname
```

#### Reducing VG
```shell=
# Move the PE
pvmove /dev/vdb3
vgreduce vg01 /dev/vdb3
```

# Advanced Storage features
#### Assembling Block Storage into Stratis Pools
```shell=
stratis pool create pool1 /dev/vdb
stratis pool list
stratis pool add-data pool1 /dev/vdc
stratis blockdev list pool1
```
#### Managing Stratis FS
```shell=
stratis filesystem create pool1 fs1
stratis filesystem list
stratis filesystem snapshot pool1 fs1 snapshot1 (path=/stratis/labpool/labfs-snap)
stratis filesystem destroy stratispool1 stratis-filesystem1
```
#### Persistently Mounting Stratis File Systems
```shell=
lsblk --output=UUID /statis/pool1/fs1
"UUID=<uuid> /dir xfs defaults,x-systemd.requires=stratised.service 0 0"
```

### VDO
```shell=
# enabling VDO
yum install vdo kmod-kvdo
# creating a VDO Volume
vdo create --name=vdo1 --device=/dev/vdd --vdoLogicalSize=50G
# verify
vdo list
# Analyzing VDO volume
vdo status --name=vdo1
# vdo status
vdostats --human-readable
# FS vdo1
mkfs.xfs -K /dev/mapper/vdo1
mkdir /mnt/vdo1
mount /dev/mapper/vod1 /mnt/vdo1
```

> mount -o remount,rw /

# Accessing NAS
1. Identify
```shell=
sudo mkdir mountpoint
sudo mount serverb:/ mountpoint
sudo ls mountpoint
```
2. Mountpoint
```shell=
mkdir -p mountpoint
```
3. Mount
```shell=
sudo mount -t nfs -o rw,sync serverb:/sharte mountpoint
```

```shell=
# mount persistently
"serverb:/share /mountpoint nfs rw,soft 0 0" >> /etc/fstab
```

### AutoFS
1. Install
```shell=
yum install autofs
```
2. Modify setting
```shell=
# add a master map file
vim /etc/auto.master.d/demo.autofs
>> /shares /etc/auto.demo (auto.demo for sub settings, convention: auto.xxx)
vim /etc/auto.demo
>> work -rw,sync serverb:/shares/work
sudo systemctl enable --now autofs
```
#### Direct maps
```shell=
# master map file
>> /-  /etc/auto.direct # /- as the base directory
# /etc/auto.direct
>> /mnt/docs -rw,sync serverb:/shares/docs
```
#### Indirect wildcard maps
```shell=
# /etc/auto.demo
>> *  -rw,sync  serverb:/shares/&
```

# Controlling the Boot Process
```shell=
# list all boot target
systemctl list-dependencies graphical.target | grep target graphical.target
systemctl list-units --type=target --all
# Selecting a target at runtime
systemctl isolate multi-user.target
systemctl cat graphical.target
systemctl cat cryptsetup.target
# Setting a default target
systemctl get-default
systemctl set-default graphical.target
```
### Get all systemctl dependencies list
```shell=
systemctl list-dependencies graphical.target | grep
```
##### Selecting a different target at a boot time
1. reboot
2. interrupt the boot loader menu by pressing any key
3. move the cursor to the kernel entry
4. e to edit
5. move the cursor to the line start w/ linux
6. append systemd.unit=target.target
7. CTRL+X to boot
### Emergency boot
1. append ^linux
```shell=
systemd.unit=emergency.target
mount # check all mount point
mount -o rw,remount /
mount -a
# edit /etc/fstab to fix
systemctl daemon-reload
mount -a
init 6
```
### PASSWORD RESETTING
1. reboot
2. on the select kernel page
3. press e to edit
4. ^linux append "rd.break" on tail
5. ctrl+x
```shell=
mount -o rw,remount /sysroot
chroot /sysroot
passwd root
touch /.autorelabel # selinux config
```

### Inspect Logs
> system journals => /run/log/journal
```shell=
vim /etc/systemd/journald.conf
# Storage=persistent
systemctl restart systemd-journald.service
journalctl -b -1 -p err # show error
```

# Managing Network security
```shell=
# set-default zone to dmz
firewall-cmd --set-default-zone=dmz
firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24
firewall-cmd --permanent --zone=internal --add-service=mysql
firewall-cmd --permanent --zone=public --add-port=82/tcp 
firewall-cmd --reload
```
### service check
```shell=
systemctl status nftables
systemctl mask nftables
systemctl status firewalld
```

## Controlling SELinux Port Labeling
```shell=
# Listing port labels
semanage port -l
# Managing Port Labels
semanage port -a -t gopher_port_t -p tcp 71
# check local change
semanage port -l -C
# Removing port labels
semanage port -d -t gopher_port_t -p tcp 71
# Modifying port bindings (gopher_port_t=>http_port_t)
semanage port -m -t http_port_t -p tcp 71
# sealert 
sealert -a /var/log/audit/audit.log
```

# Containers
### Basic commands
```shell=
# Install podman
yum module install container-tools
# info
podman info
# Image
podman images
podman login
podman search
podman pull
podman inspect
podman rmi
# Container
podman ps
podman run --name -p <hostPort:containerPort> -e <key:value> -v <containerHostDir:containerDir:Z>
podman port -a
# firewall-cmd --add-port=8000/tcp
podman stop | start | restart | kill
podman exec -it
podman rm {-af}
# Inspect Container images
podman images
podman inspect <repo_name>
skopeo inspect docker://registry.lab.example.com/rhel8/httpd-24 
```

### Managing containers as Services
```shell=
loginctl enable-linger
loginctl show-user user
# Linger=yes
loginctl disable-linger
loginctl show-user user
# Linger=no
```
### Creating & Managing Systemd User sevices
```shell=
ls ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable myapp.service
systemctl --user start myapp.service
```
![](https://i.imgur.com/fFw0kmR.png)
### Creating the Systemed Unit File
```shell=
cd ~/.config/systemd/user/
podman generate systemd --name web --files --new
```


### Network
