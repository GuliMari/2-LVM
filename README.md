# Домашнее задание: Работа с LVM и файловыми системами

На имеющемся образе:

1. уменьшить том под / до 8G
2. выделить том под /home
3. выделить том под /var (/var - сделать в mirror)
4. для /home - сделать том для снэпшотов
5. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами (на выбор)
6. Работа со снапшотами:

- сгенерить файлы в /home/
- снять снэпшот
- удалить часть файлов
- восстановится со снэпшота
Задание со *:
на нашей куче дисков попробовать поставить btrfs/zfs - с кешем, снэпшотами - разметить здесь каталог /opt

## Выполнение

Создаем виртуальную машину по имеющемуся образу и подключаемся через SSH.
Команды выполнены от имени root.

### Уменьшить том под / до 8G

Посмотрим расположение каталога /:

```bash
[root@tw4-VB vagrant]# lsblk

NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk

```

Подготовим временный раздел для корневого тома:

```bash
[root@tw4-VB vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@tw4-VB vagrant]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
[root@tw4-VB vagrant]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

```

Создадим файловую систему XFS и смонтируем ее в каталог /mnt:

```bash
[root@tw4-VB vagrant]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@tw4-VB vagrant]# mount /dev/vg_root/lv_root /mnt

```

Скопируем содержимое текущего корневого раздела в наш временный:

```bash
[root@tw4-VB vagrant]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsrestore: Restore Status: SUCCESS

```

Заходим в окружение chroot нашего временного корня:

```bash
[root@tw4-VB vagrant]#for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@tw4-VB vagrant]# chroot /mnt/

```

Запишем новый загрузчик:

```bash
[root@tw4-VB /]#  grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

Обновляем образы загрузки:

```bash
[root@tw4-VB /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

```

Открываем конфигурационный файл grub и меняем все значения, которые задействуют старый том lvm:

```bash
[root@tw4-VB boot]# sed -i 's@VolGroup00/LogVol00@vg_root/lv_root@g' /boot/grub2/grub.cfg

```

Выходим из окружения chroot и перезагружаем компьютер с временного раздела:

```bash
[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm  
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

```

Удаляем LV, который нужно уменьшить, создаем новый на 8G:

```bash
[root@lvm ~]#  lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
[root@lvm ~]#  lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt

```

Возвращаем загрузку с уменьшенного LV:

```bash
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
xfsrestore: Restore Status: SUCCESS
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

```

### Выделить том под /var (/var - сделать в mirror)
Не выходя из chroot, создаем зеркало под /var:

```bash
[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
[root@lvm boot]#  lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  No input from event server.
  No input from event server.
  vg_var-lv_var: event registration failed: Input/output error.
  vg_var/lv_var: raid1 segment monitoring function failed.
  Logical volume "lv_var" created.
[root@lvm boot]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk
├─sda1                     8:1    0    1M  0 part
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /
sdb                        8:16   0   10G  0 disk
└─vg_root-lv_root        253:0    0   10G  0 lvm
sdc                        8:32   0    2G  0 disk
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm
sdd                        8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm
│ └─vg_var-lv_var        253:7    0  952M  0 lvm
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm
  └─vg_var-lv_var        253:7    0  952M  0 lvm
sde                        8:64   0    1G  0 disk

```

Создаем файловую систему и переносим содержимое старого var в новый:

```bash
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt
[root@lvm boot]# cp -aR /var/* /mnt/
[root@lvm boot]# mkdir /tmp/oldvar
[root@lvm boot]# mv /var/* /tmp/oldvar
[root@lvm boot]# umount /mnt
[root@lvm boot]# mount /dev/vg_var/lv_var /var
[root@lvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /mnt/boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /mnt
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:0    0   10G  0 lvm  /
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 

```

Исправляем fstab и перезагружаемся с уменьшенного root:

```bash

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
[root@lvm ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
UUID="aede8edb-9c45-40dc-9c80-57ea5f40928c" /var ext4 defaults 0 0

```

Снова перезагружаем систему с уменьшенного рута и удаляем временный том и группу:

```bash
[root@lvm ~]# lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[root@lvm ~]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.

```
### Выделить том под /home

Теперь выделяем том под /home, создаем ФС, копируем данные, монтируем:

```bash
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/
[root@lvm ~]# cp -aR /home/* /mnt/
[root@lvm ~]# rm -rf /home/*
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

```
### Работа со снапшотами
Создаем файлы в /home и делаем снапшот:

```bash
[root@lvm ~]# touch /home/file{1..20}
[root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
[root@lvm ~]# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk 
├─sda1                            8:1    0    1M  0 part 
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  ├─VolGroup00-LogVol_Home-real 253:8    0    2G  0 lvm  
  │ ├─VolGroup00-LogVol_Home    253:2    0    2G  0 lvm  /home
  │ └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
  └─VolGroup00-home_snap-cow    253:9    0  128M  0 lvm  
    └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
sdb                               8:16   0   10G  0 disk 
sdc                               8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0         253:3    0    4M  0 lvm  
│ └─vg_var-lv_var               253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_0        253:4    0  952M  0 lvm  
  └─vg_var-lv_var               253:7    0  952M  0 lvm  /var  
sdd                               8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1         253:5    0    4M  0 lvm  
│ └─vg_var-lv_var               253:7    0  952M  0 lvm  /var  
└─vg_var-lv_var_rimage_1        253:6    0  952M  0 lvm  
  └─vg_var-lv_var               253:7    0  952M  0 lvm  /var  
sde                               8:64   0    1G  0 disk 

```

Удалим пару файлов и восстановим /home со снапшота:

```bash
[root@lvm ~]# rm -f /home/file{1..10}
[root@lvm ~]# ll /home/
total 0
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file11
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file12
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file13
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file14
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file15
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file16
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file17
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file18
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file19
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file20
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
[root@lvm ~]# umount /home
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
[root@lvm ~]# mount /home
[root@lvm ~]# ll /home/
total 0
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file1
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file10
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file11
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file12
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file13
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file14
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file15
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file16
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file17
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file18
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file19
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file2
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file20
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file3
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file4
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file5
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file6
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file7
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file8
-rw-r--r--. 1 root    root     0 Nov 23 05:28 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant





