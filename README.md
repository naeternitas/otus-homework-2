# Otus Homework 2
## Prepare
- [x] добавить в Vagrantfile еще дисков;
- [x] сломать/починить raid;
- [x] собрать R0/R5/R10 на выбор;
- [x] прописать собранный рейд в конф, чтобы рейд собирался при загрузке;
- [x] создать GPT раздел и 5 партиций.
- [x] Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться
- [x] перенести работающую систему с одним диском на RAID 1. Даунтайм на загрузку с нового диска предполагается. В качестве проверки принимается вывод команды lsblk до и после и описание хода решения (можно воспользоваться утилитой Script).

### Raid 5 Build 
Проверим наличие существующих mdadm масивов:
```
cat /proc/mdstat
```
массивы отсутствуют:
```
Personalities :
unused devices: <none>
```
Наличие дисков подключенных к системе:
```
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```
```
NAME    SIZE FSTYPE TYPE MOUNTPOINT
sda      40G        disk
└─sda1   40G xfs    part /
sdb     250M        disk
sdc     250M        disk
sdd     250M        disk
sde     250M        disk
sdf     250M        disk
```
Создадим рейд массив 5 из 5 дисков:
```
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=5 /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf
```
создано и запущенно:
```
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
проверяем наличие массива:
```
cat /proc/mdstat
```
```
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sdf[5] sde[3] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]

unused devices: <none>
```
Создаем файловую систему ext4
```
sudo mkfs.ext4 -F /dev/md0
```
```
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=128 blocks, Stripe width=512 blocks
63488 inodes, 253952 blocks
12697 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=260046848
8 block groups
32768 blocks per group, 32768 fragments per group
7936 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```
Создание каталога для монтирование и монтирование соответственно:
```
sudo mkdir -p /mnt/md0
sudo mount /dev/md0 /mnt/md0
``` 
проверяем правильно ли все примонтировалось
```
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        111M     0  111M   0% /dev
tmpfs           118M     0  118M   0% /dev/shm
tmpfs           118M  4.6M  113M   4% /run
tmpfs           118M     0  118M   0% /sys/fs/cgroup
/dev/sda1        40G  3.2G   37G   8% /
tmpfs            24M     0   24M   0% /run/user/1000
tmpfs            24M     0   24M   0% /run/user/0
/dev/md0        961M  2.5M  893M   1% /mnt/md0
```
### Raid Fault and recovery
```
mdadm --detail /dev/md0
```
```
/dev/md0:
           Version : 1.2
     Creation Time : Thu Jan 13 11:42:53 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Jan 13 11:46:26 2022
             State : clean
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 62f15965:b0bfaa29:e2ba6a65:4176f881
            Events : 42

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       6       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf
```
Запускаем mdadm для перевода диска sdd в режим fault
```
sudo mdadm /dev/md0 -f /dev/sdc
```
```
mdadm: set /dev/sdc faulty in /dev/md0
```

проверяем статус дисков в массиве md0
```
sudo mdadm --detail /dev/md0
```
```
/dev/md0:
           Version : 1.2
     Creation Time : Thu Jan 13 11:42:53 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Jan 13 11:48:41 2022
             State : clean, degraded
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 62f15965:b0bfaa29:e2ba6a65:4176f881
            Events : 44

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       -       0        0        1      removed
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf

       6       8       32        -      faulty   /dev/sdc
```

удаляем диск sdc
```
sudo mdadm /dev/md0 -r /dev/sdc
```
```
mdadm: hot removed /dev/sdc from /dev/md0
```
добавляем диск в тот же массив
```
sudo mdadm /dev/md0 -a /dev/sdc
```
```
mdadm: added /dev/sdc
```
проверяем статус дисков в массиве
```
sudo mdadm --detail /dev/md0
```
```
/dev/md0:
           Version : 1.2
     Creation Time : Thu Jan 13 11:42:53 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Jan 13 11:57:44 2022
             State : clean
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 62f15965:b0bfaa29:e2ba6a65:4176f881
            Events : 64

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       6       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       5       8       80        4      active sync   /dev/sdf
```
добавим конфигурацию mdadm в автозапуск
```
mdadm --verbose --detail -scan > /etc/mdadm.conf
```
### Vagrantfile
находим нужный нам блок
```
      box.vm.provision "shell", inline: <<-SHELL
	      mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
	      yum install -y mdadm smartmontools hdparm gdisk 
  	  SHELL
```
и приводим его к следующему виду
```
      box.vm.provision "shell", inline: <<-SHELL
	      mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
	      yum install -y mdadm smartmontools hdparm gdisk
		  mdadm --create --verbose /dev/md0 --level=5 --raid-devices=5 /dev/sdb /dev/sdc /dev/sdd /dev/sde /dev/sdf
		  mkfs.ext4 -F /dev/md0
		  mkdir /mnt/md0
		  mount /dev/md0 /mnt/md0
		  mdadm --verbose --detail -scan > /etc/mdadm.conf
		  echo "/dev/md0    /mnt/md0    ext4    defaults    0    1" >> /etc/fstab
  	  SHELL
```
проверяем примонтировались ли диски
```
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        111M     0  111M   0% /dev
tmpfs           118M     0  118M   0% /dev/shm
tmpfs           118M  4.5M  114M   4% /run
tmpfs           118M     0  118M   0% /sys/fs/cgroup
/dev/sda1        40G  3.1G   37G   8% /
/dev/md0        961M  2.5M  893M   1% /mnt/md0
tmpfs            24M     0   24M   0% /run/user/1000
```
проверяем результат после перезагрузки
```
sudo -i
reboot
```
```
df -h
```
```
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        111M     0  111M   0% /dev
tmpfs           118M     0  118M   0% /dev/shm
tmpfs           118M  4.5M  114M   4% /run
tmpfs           118M     0  118M   0% /sys/fs/cgroup
/dev/sda1        40G  3.1G   37G   8% /
/dev/md0        961M  2.5M  893M   1% /mnt/md0
tmpfs            24M     0   24M   0% /run/user/1000
```
### Single Disk to RAID
Последняя задача в списке перенос диска на Raid1 для этого в Vagrantfile увеличем объем первого диска сата до 40960(40GB) и сменим тип дисков с Fixed на Standard
```
	:disks => {
		:sata1 => {
			:dfile => home + '/VirtualBox VMs/disks/sata1.vdi',
			:size => 40960,
			:port => 1
		},
```
```
				vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Standard', '--size', dconf[:size]]
```

```
lsblk
```
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0   40G  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0  250M  0 disk
```
далее все команды стартуют от рута
```
sudo -i
```
копирум таблицу разделов с sda на sdb
```
sfdisk -d /dev/sda | sfdisk /dev/sdb
```
проверяем результат
```
fdisk -l
```
```
Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0009ef1a

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    83886079    41942016   83  Linux

Disk /dev/sdb: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x00000000

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1   *        2048    83886079    41942016   83  Linux

Disk /dev/sdc: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdd: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sde: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdf: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```
изменяем тип раздела на FD(Raid Autodetect MBR) у /dev/sdb 
```
sudo sfdisk --change-id /dev/sdb 1 fd
```
```
Done

```
так как разметка нашего диска подразумевает только один раздел на весь диск то только его и добавляем в новый массив
```
mdadm --create /dev/md0 --level=1 --raid-devices=2 missing /dev/sdb1
```
```
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
проверяем создался ли раздел
```
cat /proc/mdstat
```
```
Personalities : [raid1]
md0 : active raid1 sdb1[1]
      41908224 blocks super 1.2 [2/1] [_U]

unused devices: <none>
```
создаем файловую систему ext4 на нашем новом массиве
```
mkfs.ext4 /dev/md0
```
примонтируем новый массив в mnt
```
mount /dev/md0 /mnt/
```
использую rsync скопируем данные из (sysroot)/ в /mnt исключая виртуальные каталоги (сохраняняя максимальное кол-во параметров файлов в том числе ссылки и т.д. и т.п.)
```
rsync -auxHAXSv --exclude=/dev/* --exclude=/proc/* --exclude=/sys/* --exclude=/tmp/* --exclude=/mnt/* /* /mnt
```
```
LONG OUTPUT
```
подключим в наш каталог с системными файлами виртуальные директории
```
mount --bind /proc /mnt/proc
mount --bind /dev /mnt/dev
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```
используя chroot подключимся к нашей ОС на массиве 
```
chroot /mnt/
```
проверим UUID для диска md0 он нам понадобится во время правок fstab
```
blkid /dev/md*
```
```
/dev/md0: UUID="bee54ceb-2b02-4807-9cf9-9219c15b5ae7" TYPE="ext4"
```
вносим правки в fstab
```
vi /etc/fstab
```
приводим к следующему виду
```
#
# /etc/fstab
# Created by anaconda on Thu Apr 30 22:04:55 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=bee54ceb-2b02-4807-9cf9-9219c15b5ae7 /                       ext4     defaults        0 0
/swapfile none swap defaults 0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
```
добавляем в автозагрузку наш mdadm конфиг
```
mdadm --detail --scan > /etc/mdadm.conf
```
теперь очередь загрузки и файла initamfs
```
cp /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bck
dracut --mdadmconf --fstab --add="mdraid" --filesystems "xfs ext4 ext3" --add-drivers="raid1" --force /boot/initramfs-$(uname -r).img $(uname -r) -M
```
немного тюнинга дефолтных параметров grub
```
vi /etc/default/grub
```
приводим файл в следующий вид
```
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.auto rd.auto=1 rhgb quiet"
GRUB_PRELOAD_MODULES="mdraid1x"
GRUB_DISABLE_RECOVERY="true"
```
генерируем конфиг файл груба
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
ну и наконец устанавливаем grub на /dev/sdb
```
grub2-install /dev/sdb
```
тот самый момент для перезагрузки и выбора диска sdb как загрузочного
следующий шаг это добавление нашего предыдущего диска в новый массив 
```
sudo -i
mdadm --manage /dev/md0 --add /dev/sda1
```
отслеживаем статус зеркалирования дисков
```
watch -n1 "cat /proc/mdstat"
```
по окончанию устанавливаем grub и на первый диск
```
grub2-install /dev/sda
```
ну и результат работы
```
lsblk
```
```
NAME    MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda       8:0    0   40G  0 disk
└─sda1    8:1    0   40G  0 part
  └─md0   9:0    0   40G  0 raid1 /
sdb       8:16   0   40G  0 disk
└─sdb1    8:17   0   40G  0 part
  └─md0   9:0    0   40G  0 raid1 /
sdc       8:32   0  250M  0 disk
sdd       8:48   0  250M  0 disk
sde       8:64   0  250M  0 disk
sdf       8:80   0  250M  0 disk
```
на этом дз завершено.
