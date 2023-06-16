# OSC Debian image creation 

## Prerequisites 
- Builder vm based on debian with ssh access

## Install oras
```
wget https://github.com/oras-project/oras/releases/download/v1.0.0/oras_1.0.0_linux_amd64.tar.gz
tar -zxvf oras_1.0.0_linux_amd64.tar.gz
mv oras /usr/bin/
```

## Download mount debian raw image
```
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.raw
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
apt-get install -y ignition 
apt-get install -y dracut dracut-network
apt-get install -y selinux-basics selinux-policy-default --no-install-recommends

sed -i '/^RemainAfterExit=.*/a Environment="IGNITION_WRITE_AUTHORIZED_KEYS_FRAGMENT=false"' /usr/lib/dracut/modules.d/30ignition/ignition-files.service

for kernel in /boot/vmlinuz-*; do
   dracut -fM --kver "${kernel#*-}"
   kernel-install -v add "${kernel#*-}" "${kernel}"
done

echo 'GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200 earlyprintk=ttyS0,115200 consoleblank=0 cgroup_enable=memory swapaccount=1 ignition.firstboot=1 ignition.platform.id=qemu security=selinux"' > /etc/default/grub.d/99_cmdline.cfg
update-grub

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
growpart /dev/vda 1
ssh-keygen -A
touch /etc/firstboot.done
systemctl --force --force reboot
EOF

systemctl enable firstboot.service
```

## Create OCI image
```
oras login -u xxxx -p xxxx ghcr.io

echo '{"commandLine":"root=LABEL=ROOT ro console=tty0 console=ttyS0,115200 earlyprintk=ttyS0,115200 consoleblank=0 cgroup_enable=memory swapaccount=1 ignition.firstboot=1 ignition.platform.id=qemu security=selinux"}' > debian-12-genericcloud-amd64.config
oras push ghcr.io/t-systems/osc-images/debian:12-genericcloud-amd64-2 \
debian-12-genericcloud-amd64.raw:application/vnd.onmetal.image.rootfs.v1alpha1.rootfs \
/mnt/boot/vmlinuz-6.1.0-9-cloud-amd64:application/vnd.onmetal.image.vmlinuz.v1alpha1.vmlinuz \
/mnt/boot/initrd.img-6.1.0-9-cloud-amd64:application/vnd.onmetal.image.initramfs.v1alpha1.initramfs \
--config debian-12-genericcloud-amd64.config:application/vnd.onmetal.image.config.v1alpha1+json
```

## Unmount image
```
umount /mnt
losetup -d /dev/loop0
```
