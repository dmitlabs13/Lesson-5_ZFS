# Практические навыки работы с ZFS

## Задание
1. Определить алгоритм с наилучшим сжатием:
определить, какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
создать 4 файловых системы, на каждой применить свой алгоритм сжатия;
для сжатия использовать либо текстовый файл, либо группу файлов.
2. Определить настройки пула.
С помощью команды zfs import собрать pool ZFS.
Командами zfs определить настройки:
размер хранилища;
тип pool;
значение recordsize;
какое сжатие используется;
какая контрольная сумма используется.
3. Работа со снапшотами:
скопировать файл из удаленной директории;
восстановить файл локально. zfs receive;
найти зашифрованное сообщение в файле secret_message.

## Выполнение задания

### 1. Определить алгоритм с наилучшим сжатием

```
# ставим компоненты
 sudo apt install zfsutils-linux

# создаем пулы
zpool create zfs_pool1 mirror /dev/sdb /dev/sdc
zpool create zfs_pool2 mirror /dev/sdd /dev/sde
zpool create zfs_pool3 mirror /dev/sdf /dev/sdg
zpool create zfs_pool4 mirror /dev/sdh /dev/sdi

# проверяем что получилось
root@lp-ubn1:/home/sadmin# zpool list
NAME        SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
zfs_pool1   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_pool2   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_pool3   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -
zfs_pool4   480M   111K   480M        -         -     0%     0%  1.00x    ONLINE  -

# устанавливаем разные агоритмы сжатия
zfs set compression=lzjb zfs_pool1
zfs set compression=lz4 zfs_pool2
zfs set compression=gzip-9 zfs_pool3
zfs set compression=zle zfs_pool4

# проверяем наличие методов сжатия
root@lp-ubn1:/home/sadmin# zfs get all | grep compression
zfs_pool1  compression           lzjb                   local
zfs_pool2  compression           lz4                    local
zfs_pool3  compression           gzip-9                 local
zfs_pool4  compression           zle                    local

# качаем файл во все пулы
for i in {1..4}; do wget -P /zfs_pool$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done

# проверяем
ls - l /zfs_pool*
/zfs_pool1:
total 22125
-rw-r--r-- 1 root root 41227642 Apr  2 07:31 pg2600.converter.log

/zfs_pool2:
total 18019
-rw-r--r-- 1 root root 41227642 Apr  2 07:31 pg2600.converter.log

/zfs_pool3:
total 10972
-rw-r--r-- 1 root root 41227642 Apr  2 07:31 pg2600.converter.log

/zfs_pool4:
total 40290
-rw-r--r-- 1 root root 41227642 Apr  2 07:31 pg2600.converter.log

root@lp-ubn1:/home/sadmin# zfs list
NAME        USED  AVAIL  REFER  MOUNTPOINT
zfs_pool1  21.8M   330M  21.6M  /zfs_pool1
zfs_pool2  17.7M   334M  17.6M  /zfs_pool2
zfs_pool3  10.9M   341M  10.7M  /zfs_pool3
zfs_pool4  39.5M   312M  39.4M  /zfs_pool4


root@lp-ubn1:/home/sadmin# zfs get all | grep compressratio | grep -v ref
zfs_pool1  compressratio         1.82x                  -
zfs_pool2  compressratio         2.23x                  -
zfs_pool3  compressratio         3.66x                  -
zfs_pool4  compressratio         1.00x                  -

```
получается самый сжимающий алгоритм из этих 4рех - gzip-9   

