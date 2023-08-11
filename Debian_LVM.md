# OSC Debian image creation  with LVM

## Prerequisites 
- Builder vm based on debian with ssh access

## Prerequisites
```
apt-get update
apt-get install lvm2
```
## Install oras
```
wget https://github.com/oras-project/oras/releases/download/v1.0.0/oras_1.0.0_linux_amd64.tar.gz
tar -zxvf oras_1.0.0_linux_amd64.tar.gz
mv oras /usr/bin/
```

## Download mount debian raw image
```
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.raw
truncate -s 4G debian-12-genericcloud-amd64.raw
losetup -f -P debian-12-genericcloud-amd64.raw
losetup -l

mount /dev/loop0p1 /mnt/

mount --bind /dev /mnt/dev
mount --bind /dev/pts /mnt/dev/pts
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys

chroot /mnt
```

## Prepare image
```
mkdir -p /run/systemd/resolve/
echo "nameserver 8.8.8.8" > /run/systemd/resolve/resolv.conf

apt-get update
apt-get install -y ignition qemu-guest-agent lvm2
apt-get install -y dracut dracut-network
apt-get install -y selinux-basics selinux-policy-default --no-install-recommends
apt-get remove -y --purge --auto-remove rpcbind nfs-common apparmor

echo '%sudo       ALL=(ALL:ALL) NOPASSWD:ALL' > /etc/sudoers.d/nopw

sed -i '/^RemainAfterExit=.*/a ExecStartPre=/usr/sbin/lvm vgchange -a y vg0' /usr/lib/dracut/modules.d/30ignition/ignition-disks.service
sed -i '/^RemainAfterExit=.*/a ExecStartPre=/usr/bin/sleep 10' /usr/lib/dracut/modules.d/30ignition/ignition-disks.service
sed -i '/^RemainAfterExit=.*/a Environment="IGNITION_WRITE_AUTHORIZED_KEYS_FRAGMENT=false"' /usr/lib/dracut/modules.d/30ignition/ignition-files.service

for kernel in /boot/vmlinuz-*; do
   dracut -fM --kver "${kernel#*-}"
   kernel-install -v add "${kernel#*-}" "${kernel}"
done

echo 'GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200 earlyprintk=ttyS0,115200 consoleblank=0 cgroup_enable=memory swapaccount=1 ignition.firstboot=1 ignition.platform.id=qemu security=selinux"' > /etc/default/grub.d/99_cmdline.cfg
update-grub

cat <<EOF >>/etc/systemd/system/ssh-keygen.service
[Unit]
Description=SSH Key Generation
ConditionPathExists=|!/etc/ssh/ssh_host_ecdsa_key
ConditionPathExists=|!/etc/ssh/ssh_host_ecdsa_key.pub
ConditionPathExists=|!/etc/ssh/ssh_host_ed25519_key
ConditionPathExists=|!/etc/ssh/ssh_host_ed25519_key.pub
ConditionPathExists=|!/etc/ssh/ssh_host_rsa_key
ConditionPathExists=|!/etc/ssh/ssh_host_rsa_key.pub
Before=ssh.service

[Service]
ExecStart=/usr/bin/ssh-keygen -A
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl enable ssh-keygen.service

cat <<EOF >>/etc/systemd/network/99-default.network
[Match]
Name=en* eth*

[Network]
DHCP=yes
LLDP=true
EmitLLDP=true

[DHCPv4]
UseDomains=true
UseMTU=true
EOF

cat <<EOF >/etc/systemd/system/firstboot.service
[Unit]
Description=FirstBootSetup
ConditionPathExists=|!/etc/firstboot.done

[Service]
Type=idle
RemainAfterExit=yes
ExecStart=/bin/bash /usr/local/bin/firstboot.sh

[Install]
WantedBy=multi-user.target
EOF

cat <<EOF >/usr/local/bin/firstboot.sh
growpart /dev/vda 2
touch /etc/firstboot.done
rm -f /etc/machine-id
systemctl --force --force reboot
EOF

systemctl enable firstboot.service

cat <<EOF >/etc/selinux/config
SELINUX=disabled
SELINUXTYPE=default
SETLOCALDEFS=0
EOF

exit
rm -f /mnt/root/.bash_history
rm -f /mnt/etc/machine-id

```

## Prepare LVM
```
umount -f /mnt/dev/pts
umount -f /mnt/dev
umount -f /mnt/proc
umount -f /mnt/sys
umount -f /mnt
losetup -d /dev/loop0
losetup -l

echo -e "n\np\n2\n\n\nw" | fdisk debian-12-genericcloud-amd64.raw
fdisk -l debian-12-genericcloud-amd64.raw
losetup -f -P debian-12-genericcloud-amd64.raw
fdisk -l /dev/loop0

pvcreate /dev/loop0p2
vgcreate vg0 /dev/loop0p2

# /root
lvcreate -n root -L 100M vg0
mkfs.ext4 -I 256 /dev/vg0/root
e2label /dev/vg0/root ROOT_HOME

# /home
lvcreate -n home -L 10M vg0
mkfs.ext4 -I 256 /dev/vg0/home
e2label /dev/vg0/home HOME

# /usr
lvcreate -n usr -L 1024M vg0
mkfs.ext4 -I 256 /dev/vg0/usr
e2label /dev/vg0/usr USR

# /tmp
lvcreate -n tmp -L 100M vg0
mkfs.ext4 -I 256 /dev/vg0/tmp
e2label /dev/vg0/tmp TMP

# /var
lvcreate -n var -L 200M vg0
mkfs.ext4 -I 256 /dev/vg0/var
e2label /dev/vg0/var VAR

# /var/lib
lvcreate -n var_lib -L 200M vg0
mkfs.ext4 -I 256 /dev/vg0/var_lib
e2label /dev/vg0/var_lib VAR_LIB

# /var/log
lvcreate -n var_log -L 100M vg0
mkfs.ext4 -I 256 /dev/vg0/var_log
e2label /dev/vg0/var_log VAR_LOG

# /var/log/audit
lvcreate -n var_log_audit -L 10M vg0
mkfs.ext4 -I 256 /dev/vg0/var_log_audit
e2label /dev/vg0/var_log_audit VAR_LOG_AUDIT

# /var/tmp
lvcreate -n var_tmp -L 100M vg0
mkfs.ext4 -I 256 /dev/vg0/var_tmp
e2label /dev/vg0/var_tmp VAR_TMP

# /opt
lvcreate -n opt -L 10M vg0
mkfs.ext4 -I 256 /dev/vg0/opt
e2label /dev/vg0/opt OPT

# swap
lvcreate -n swap -L 64M vg0
mkswap -L SWAP /dev/vg0/swap

mount /dev/loop0p1 /mnt/

# /root
mkdir /mnt/root_backup
find /mnt/root/ -maxdepth 1 -name '.*' ! -name '.' ! -name '..' -exec mv {} /mnt/root_backup/ \;
mount /dev/vg0/root /mnt/root
find /mnt/root_backup/ -maxdepth 1 -name '.*' ! -name '.' ! -name '..' -exec mv {} /mnt/root/ \;
ls -la /mnt/root/
rm -rf /mnt/root_backup/

# /usr
mkdir /tmp/usr_backup
rsync -aHAX --progress /mnt/usr/* /tmp/usr_backup/
mount /dev/vg0/usr /mnt/usr
mv /tmp/usr_backup/* /mnt/usr/
ls -la /mnt/usr/
rm -rf /tmp/usr_backup/

# /var
mkdir /mnt/var_backup
mkdir /mnt/var_lib_backup
mkdir /mnt/var_log_backup
mv /mnt/var/lib/* /mnt/var_lib_backup/
mv /mnt/var/log/* /mnt/var_log_backup/
mv /mnt/var/* /mnt/var_backup/
mount /dev/vg0/var /mnt/var
mv /mnt/var_backup/* /mnt/var/
ls -la /mnt/var/
rm -rf /mnt/var_backup/

# /var/lib
mount /dev/vg0/var_lib /mnt/var/lib
mv /mnt/var_lib_backup/* /mnt/var/lib/
ls -la /mnt/var/lib/
rm -rf /mnt/var_lib_backup/

# /var/log
mount /dev/vg0/var_log /mnt/var/log
mv /mnt/var_log_backup/* /mnt/var/log/
ls -la /mnt/var/log/
rm -rf /mnt/var_log_backup/

cat /mnt/etc/fstab

cp /mnt/boot/vmlinuz-6.1.0-10-cloud-amd64 .
cp /mnt/boot/initrd.img-6.1.0-10-cloud-amd64 .
```

## Unmount image
```
umount -f /mnt/dev/pts
umount -f /mnt/dev
umount -f /mnt/proc
umount -f /mnt/sys
umount -f /mnt/root
umount -f /mnt/var/lib
umount -f /mnt/var/log
umount -f /mnt/var/log/audit
umount -f /mnt/var/tmp
umount -f /mnt/var
umount -f /mnt/opt
umount -f /mnt/usr
umount -f /mnt

vgchange -a n vg0

losetup -d /dev/loop0
losetup -l
```

## Create OCI image
```

oras login -u xxxx -p xxxx ghcr.io

echo '{"commandLine":"root=LABEL=ROOT ro console=tty0 console=ttyS0,115200 earlyprintk=ttyS0,115200 consoleblank=0 cgroup_enable=memory swapaccount=1 ignition.firstboot=1 ignition.platform.id=qemu security=selinux"}' > debian-12-genericcloud-amd64.config
oras push ghcr.io/t-systems/osc-images/debian:12-genericcloud-amd64-lvm-31 \
debian-12-genericcloud-amd64.raw:application/vnd.onmetal.image.rootfs.v1alpha1.rootfs \
vmlinuz-6.1.0-10-cloud-amd64:application/vnd.onmetal.image.vmlinuz.v1alpha1.vmlinuz \
initrd.img-6.1.0-10-cloud-amd64:application/vnd.onmetal.image.initramfs.v1alpha1.initramfs \
--config debian-12-genericcloud-amd64.config:application/vnd.onmetal.image.config.v1alpha1+json
```


