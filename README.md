# lvm
Сразу установим пакет

Installed:
  xfsdump.x86_64 0:3.1.7-4.el7_9

Dependency Installed:
  attr.x86_64 0:2.4.46-13.el7

Updated:
  lvm2.x86_64 7:2.02.187-6.el7_9.5

Dependency Updated:
  device-mapper.x86_64 7:1.02.170-6.el7_9.5  device-mapper-event.x86_64 7:1.02.170-6.el7_9.5  device-mapper-event-libs.x86_64 7:1.02.170-6.el7_9.5  device-mapper-libs.x86_64 7:1.02.170-6.el7_9.5  lvm2-libs.x86_64 7:2.02.187-6.el7_9.5

Complete!

[root@lvm vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
  
[root@lvm vagrant]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
 
Проверим разделы 
[root@lvm vagrant]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g      0
  vg_root      1   0   0 wz--n- <10.00g <10.00g
  
[root@lvm vagrant]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
 
[root@lvm vagrant]# lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g
  LogVol01 VolGroup00 -wi-ao----   1.50g
  lv_root  vg_root    -wi-a----- <10.00g
 
 Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:
[root@lvm vagrant]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0


[root@lvm vagrant]# mount /dev/vg_root/lv_root /mnt

Этой командой копируем все данные с / раздела в /mnt:
[root@lvm vagrant]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Fri Mar 29 12:29:58 2024
xfsdump: session id: c0396ad1-d90a-495b-9173-0a738f92b61b
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 900014528 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description:
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Fri Mar 29 12:29:58 2024
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: c0396ad1-d90a-495b-9173-0a738f92b61b
xfsrestore: media id: e0954025-417f-4519-a1f7-ae63f10f2cc5
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2701 directories and 23722 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 876932304 bytes
xfsdump: dump size (non-dir files) : 863702640 bytes
xfsdump: dump complete: 8 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 8 seconds elapsed
xfsrestore: Restore Status: SUCCESS

Затем сконфигурируем grub для того, чтобы при старте перейти в новый /.
Сымитируем текущий root, сделаем в него chroot и обновим grub:

[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
> do mount --bind $i /mnt/$i; done

[root@lvm vagrant]# chroot /mnt/

[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

Обновим образ initrd

[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; \
> do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> s/.img//g"` --force; done
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Ну и для того, чтобы при загрузке был смонтирован нужны root нужно в файле
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

[root@lvm boot]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:2    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

Нужно изменить размер старой VG и вернуть на него рут. Для этого удаляем старый LV размером в 40G и создаём новый на 8G:
Перед изменением нужно перезагрузить, чтобы корневой раздел освободился
[root@lvm vagrant]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed

  Создаем раздел 8gb:
[root@lvm vagrant]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.

  [root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0    8G  0 lvm
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

Проделываем на нем те же операции, что и в первый раз:
[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm vagrant]# mount /dev/VolGroup00/LogVol00 /mnt

[root@lvm vagrant]# xfsdump -J - /dev/vg_root/lv_root | \
> xfsrestore -J - /mnt
xfsrestore: Restore Status: SUCCESS

cконфигурируем grub, за исключением правки /etc/grub2/grub.cfg
[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
> do mount --bind $i /mnt/$i; done

[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; \
> do mount --bind $i /mnt/$i; done

[root@lvm vagrant]# chroot /mnt/
  
  [root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done

[root@lvm boot]# cd /boot ; for i in `ls initramfs-*img`; \
> do dracut -v $i `echo $i|sed "s/initramfs-//g; \
> s/.img//g"` --force; done
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

На свободных дисках создаем зеркало:
 [root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.

 [root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created

[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.

Создаем на нем ФС и перемещаем туда /var:

[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

[root@lvm boot]# mount /dev/vg_var/lv_var /mnt

[root@lvm boot]# cp -aR /var/* /mnt/

На всякий случай сохраняем содержимое старого var
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

монтируем новый var в каталог /var:
[root@lvm boot]# umount /mnt

[root@lvm boot]# mount /dev/vg_var/lv_var /var

Правим fstab для автоматического монтирования /var:
[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` \
> /var ext4 defaults 0 0" >> /etc/fstab

можно успешно перезагружаться в новый (уменьшенный root) и удалять
временную Volume Group:

[root@lvm vagrant]# lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed

[root@lvm vagrant]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed

[root@lvm vagrant]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.

Выделить том под /home

[root@lvm vagrant]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.

[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

[root@lvm vagrant]# mount /dev/VolGroup00/LogVol_Home /mnt/
[root@lvm vagrant]# cp -aR /home/* /mnt/
[root@lvm vagrant]# rm -rf /home/*
[root@lvm vagrant]# umount /mnt
[root@lvm vagrant]# mount /dev/VolGroup00/LogVol_Home /home/

Правим fstab для автоматического монтирования /home:
[root@lvm vagrant]# echo "`blkid | grep Home | awk '{print $2}'` \
> /home xfs defaults 0 0" >> /etc/fstab

Работа со снапшотами

Генерируем файлы в /home/:
[root@lvm vagrant]# touch /home/file{1..20}

Снять снапшот:
[root@lvm vagrant]# lvcreate -L 100MB -s -n home_snap \
> /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.

Удалить часть файлов:

[root@lvm vagrant]# rm -f /home/file{11..20}

восстановления из снапшота:
[root@lvm vagrant]# umount /home

[root@lvm vagrant]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%

[root@lvm vagrant]# mount /home

[root@lvm vagrant]# ls -al /home
total 0
drwxr-xr-x.  3 root    root    152 Mar 29 13:53 .
drwxr-xr-x. 18 root    root    239 Mar 29 13:19 ..
-rw-r--r--.  1 root    root      0 Mar 29 13:51 file1
-rw-r--r--.  1 root    root      0 Mar 29 13:51 file10
-rw-r--r--.  1 root    root      0 Mar 29 13:51 file11
-rw-r--r--.  1 root    root      0 Mar 29 13:51 file12
-rw-r--r--.  1 root    root      0 Mar 29 13:51 file13



  


