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

## Исполнение

Создаем виртуальную машину по имеющемуся образу и подключаемся через SSH.
Команды выполнены от имени root.

### 1. уменьшить том под / до 8G

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

```

Исправляем fstab и перезагружаемся с уменьшенного root:

```bash

[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

