**Дисковая подсистема**

**Домашняя работа**

Добавить в Vagrantfile еще дисков.
Собрать R0/R5/R10 на выбор.
Сломать/починить raid.
Прописать собранный рейд в конфигурационный файл, чтобы рейд собирался при загрузке.
Создать GPT раздел и 5 партиций.
Vagrantfile, который сразу собирает систему с подключенным рейдом.

1. Работа с Vagrant
```
C:\Vagrant>vagrant -v

Vagrant 2.2.9

C:\Vagrant>vagrant box list

C:\Vagrant>vagrant box list
centos-7-8 (virtualbox, 0)
centos/7   (virtualbox, 2004.01)
```
Создан Vagrantfile(приложен к репозиторию RAID с добавленными дисками), установим виртуальную машину.

```
c:\Vagrant\otuslinux\otus-linux>vagrant up
```
Gроверим 

```
C:\Vagrant>vagrant status

Current machine states:

otuslinux running (virtualbox)
```
Все работает корректно, подключимся по ssh:
```
c:\Vagrant\otuslinux\otus-linux>vagrant ssh
Last login: Tue Aug 11 07:51:18 2020
Last login: Tue Aug 11 07:51:18 2020
```
Подключение прошло штатно:
```
[vagrant@otuslinux ~]$ ls -l /dev/ | grep sd
brw-rw----. 1 root disk      8,   0 Aug 11 08:06 sda
brw-rw----. 1 root disk      8,   1 Aug 11 08:06 sda1
brw-rw----. 1 root disk      8,  16 Aug 11 08:06 sdb
brw-rw----. 1 root disk      8,  32 Aug 11 08:06 sdc
brw-rw----. 1 root disk      8,  48 Aug 11 08:06 sdd
brw-rw----. 1 root disk      8,  64 Aug 11 08:06 sde

[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk

[vagrant@otuslinux ~]$ sudo lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb   disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc   disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd   disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde   disk        262MB VBOX HARDDISK
```
2.Работа с mdadm, сборка RAID
Собираем из дисков RAID. Выбран RAID 10. Опция -l какого уровня RAID создавать,опция - n указывает на кол-во устройств в RAID:
```
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
Смотрим состояние рэйда после сборки:
```
[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE   MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part   /
sdb      8:16   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sdc      8:32   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sdd      8:48   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sde      8:64   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
unused devices: <none>

[vagrant@otuslinux ~]$ sudo mdadm --detail --scan --verbose
ARRAY /dev/md0 level=raid10 num-devices=4 metadata=1.2 name=otuslinux:0 UUID=0ea2ecc5:1b14c2e8:605d41ac:d32c14f9
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde
```

3.Работа с mdadm, поломать/починить RAID5

Прописываем собранный рейд в конфигурационный файл, чтобы рейд собирался при загрузке.

создаем папку
```
[vagrant@otuslinux ~]$ sudo mkdir /etc/mdadm
```
проверяем наличие
```
[vagrant@otuslinux ~]$ ls -al /etc
```

создаем файл mdam.conf
```
[vagrant@otuslinux ~]$ sudo echo "DEVICE partitions" | sudo tee -a /etc/mdadm/mdadm.conf

[vagrant@otuslinux ~]$ sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' |sudo tee -a /etc/mdadm/mdadm.conf
ARRAY /dev/md/0 level=raid10 num-devices=4 metadata=1.2 name=otuslinux:0 UUID=0ea2ecc5:1b14c2e8:605d41ac:d32c14f9
```

Ломаем RAID:
```
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0
```
Проверяем, что получилось:
```
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sdc[1](F) sdd[2] sde[3] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/3] [U_UU]

unused devices: <none>
[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Tue Aug 11 10:14:53 2020
        Raid Level : raid10
        Array Size : 507904 (496.00 MiB 520.09 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Tue Aug 11 11:12:35 2020
             State : clean, degraded
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 0ea2ecc5:1b14c2e8:605d41ac:d32c14f9
            Events : 19

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       -       0        0        1      removed
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

       1       8       32        -      faulty   /dev/sdc

```
Удалим и установим новый:

```
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --remove /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0
[vagrant@otuslinux ~]$ sudo mdadm /dev/md0 --add /dev/sdc
mdadm: added /dev/sdc
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sdc[4] sdd[2] sde[3] sdb[0]
      507904 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]

unused devices: <none>

[vagrant@otuslinux ~]$ mdadm -D /dev/md0
mdadm: must be super-user to perform this action
[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Tue Aug 11 10:14:53 2020
        Raid Level : raid10
        Array Size : 507904 (496.00 MiB 520.09 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Tue Aug 11 11:28:24 2020
             State : clean
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : near=2
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 0ea2ecc5:1b14c2e8:605d41ac:d32c14f9
            Events : 39

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync set-A   /dev/sdb
       4       8       32        1      active sync set-B   /dev/sdc
       2       8       48        2      active sync set-A   /dev/sdd
       3       8       64        3      active sync set-B   /dev/sde

[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE   MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part   /
sdb      8:16   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sdc      8:32   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sdd      8:48   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
sde      8:64   0  250M  0 disk
└─md0    9:0    0  496M  0 raid10
```

4.Создание GPT раздела и 5 партиций
Создаем раздел GPT на RAID:
```
[vagrant@otuslinux ~]$ sudo parted -s /dev/md0 mklabel gpt
```
Создаем партиции:
```
[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 0% 20%
Information: You may need to update /etc/fstab.

[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 20% 40%
Information: You may need to update /etc/fstab.

[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 40% 60%
Information: You may need to update /etc/fstab.

[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 60% 80%
Information: You may need to update /etc/fstab.

[vagrant@otuslinux ~]$ sudo parted /dev/md0 mkpart primary ext4 80% 100%
Information: You may need to update /etc/fstab.
```
Создаем на этих партициях фс:
```
[vagrant@otuslinux ~]$ sudo mkfs.ext4 /dev/md0
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1024 blocks
126976 inodes, 507904 blocks
25395 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=34078720
62 block groups
8192 blocks per group, 8192 fragments per group
2048 inodes per group
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

[vagrant@otuslinux ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
/dev/sda1        40G  4.3G   36G  11% /
tmpfs           100M     0  100M   0% /run/user/1000
```
Монтируем их по каталогам:
```
[vagrant@otuslinux ~]$ sudo mkdir -p /raid/part{1,2,3,4,5}
```
5.Vagrantfile, который сразу собирает систему с подключенным рейдом приложен в репозиторий RAID.
