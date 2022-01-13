# Otus Homework 2
## Prepare
- [x] добавить в Vagrantfile еще дисков;
- [x] сломать/починить raid;
- [x] собрать R0/R5/R10 на выбор;
- [x] прописать собранный рейд в конф, чтобы рейд собирался при загрузке;
- [x] создать GPT раздел и 5 партиций.
- [x] Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться
- [ ] перенести работающую систему с одним диском на RAID 1. Даунтайм на загрузку с нового диска предполагается. В качестве проверки принимается вывод команды lsblk до и после и описание хода решения (можно воспользоваться утилитой Script).

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
Последняя задача в списке 
