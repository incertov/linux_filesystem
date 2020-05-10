## **Дисковая подсистема**

При выполнении задания использовалось следующее ПО:
####**Хост**
** ОС - Linux Mint 19.3 **
** Гипервизор - VirtualBox 6.1.6**
** Средство для создания и конфигурирования виртуальной среды - Vagrant 2.2.7 **
** Создание образа виртуальной машины - Packer 1.5.5 **
** Система контроля версий -  Git 2.26.2 **

####**Виртуальная машина**
** ОС - CentOS Linux release 7.6.1810**

#### **Установка ПО на виртуальной машине**

При разворачивании конфигурации виртуальной машины  в секции `box.vm.provision` выполняется установка необходимых для выполнения задания пакетов

```
box.vm.provision "shell", inline: <<-SHELL
          yum install -y mdadm smartmontools hdparm gdisk
        SHELL
```

#### **Cборка RAID-массива**

Определяем наличие дисков с помощью утилиты `lsblk`

```
vagrant@raid ~$ lsblk
sda      8:0    0   40G  0 disk
-sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0  250M  0 disk
```
Удаляем для надежности очищаем диски от каких-либо данных

```
sudo wipefs --all --force /dev/sd{b,c,d,e,f}
```

Выполняем сборку `RAID5` из подготовленных на предыдущем шаге пяти дисков

```
sudo mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}
```
Массив успешно собран

```
[vagrant@raid ~]$ cat /proc/mdstat
Personalities : [raid10] [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sde[3] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
```
```
[[vagrant@raid ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun May 10 16:20:26 2020
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun May 10 16:20:38 2020
             State : clean
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : raid:0  (local to host raid)
              UUID : 2a16fb77:fd551255:5d90783d:3d688009
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf

```

#### **Создание конфигурационного файла mdadm.conf**

```
[vagrant@raid ~]$ sudo -i
[root@raid ~]# mkdir -p /etc/mdadm
[root@raid ~]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@raid ~]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
Проверяем результат, в файле, в файле указана конфигурация массива - RAID10, количество дисков в нём - 5, версия формата записи данных о массиве - 1.2, название и идентификатор
```
[vagrant@raid etc]$ cat /etc/mdadm/mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=raid:0 UUID=2a16fb77:fd551255:5d90783d:3d688009
```
#### **Выход из строя диска в массиве**

Проведем проверку наджености массива и симитируем поломку диска /dev/sdd
```
[vagrant@raid etc]$ sudo mdadm /dev/md0 --fail /dev/sdd
mdadm: set /dev/sdd faulty in /dev/md0
```

Смотрим текущее состояние
```
[vagrant@raid etc]$ cat /proc/mdstat
 cat /proc/mdstat
Personalities : [raid10] [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sde[3] sdd[2](F) sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [UU_UU]
      
unused devices: <none>

```
```
[vagrant@raid etc]$ sudo mdadm /dev/md0 --fail /dev/sdd
mdadm: set /dev/sdd faulty in /dev/md0
[vagrant@raid etc]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun May 10 16:20:26 2020
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun May 10 16:24:28 2020
             State : clean, degraded
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : raid:0  (local to host raid)
              UUID : 2a16fb77:fd551255:5d90783d:3d688009
            Events : 20

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       -       0        0        2      removed
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf

       2       8       48        -      faulty   /dev/sdd

```
Удаляем сбойный диск

```
sudo mdadm /dev/md0 --remove /dev/sdd
mdadm: hot removed /dev/sdd from /dev/md0
```
Добавим новый диск взамен сбойного

```
mdadm /dev/md0 --add /dev/sdd
mdadm: added /dev/sdd

```
После добавления диска автоматически запускается пересборка массива

```
[vagrant@raid ~]$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun May 10 16:20:26 2020
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun May 10 16:30:50 2020
             State : clean, degraded, recovering
    Active Devices : 4
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

    Rebuild Status : 41% complete

              Name : raid:0  (local to host raid)
              UUID : 2a16fb77:fd551255:5d90783d:3d688009
            Events : 29

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       6       8       48        2      spare rebuilding   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf
```

#### **Создание и разметка диска GPT**

Создадим таблицу разделов

```
sudo parted -s /dev/md0 mklabel gpt
```

Создадим разделы

```
[vagrant@raid ~]$ sudo parted /dev/md0 mkpart primary ext4 0% 20%
[vagrant@raid ~]$ sudo parted /dev/md0 mkpart primary ext4 20% 40%
[vagrant@raid ~]$ sudo parted /dev/md0 mkpart primary ext4 40% 60%
[vagrant@raid ~]$ sudo parted /dev/md0 mkpart primary ext4 60% 80%
[vagrant@raid ~]$ sudo parted /dev/md0 mkpart primary ext4 80% 100%
```

Отформатируем созданные разделы

```
sudo -i
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50200 inodes, 200704 blocks
10035 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50800 inodes, 202752 blocks
10137 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2032 inodes per group
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
51200 inodes, 204800 blocks
10240 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2048 inodes per group
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50800 inodes, 202752 blocks
10137 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2032 inodes per group
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50200 inodes, 200704 blocks
10035 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks:
    8193, 24577, 40961, 57345, 73729

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

```

создадим точки монтирования

```
sudo mkdir -p /mnt/part{1,2,3,4,5}
```
Смонтируем разделы

````
sudo -i
for i in $(seq 1 5); do mount /dev/md0p$i /mnt/part$i; done
[root@raid ~]# cd /mnt/
[root@raid mnt]# ll -l
total 5
drwxr-xr-x. 3 root root 1024 May 10 16:44 part1
drwxr-xr-x. 3 root root 1024 May 10 16:44 part2
drwxr-xr-x. 3 root root 1024 May 10 16:44 part3
drwxr-xr-x. 3 root root 1024 May 10 16:44 part4
drwxr-xr-x. 3 root root 1024 May 10 16:44 part5



