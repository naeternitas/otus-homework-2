# Otus Homework 2
## Prepare
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

