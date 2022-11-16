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

Выходим из окружения chroot и перезагружаем компьютер.

```bash

```

