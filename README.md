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

### 2. Определить настройки пула.

```
# скачиваем файл
root@lp-ubn1:/home/sadmin# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
--2026-04-19 19:09:36--  https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 108.177.14.132, 2a00:1450:4010:c0e::84
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|108.177.14.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/octet-stream]
Saving to: ‘archive.tar.gz’

archive.tar.gz                     100%[=============================================================>]   6.94M  8.61MB/s    in 0.8s

2026-04-19 19:09:43 (8.61 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]

# разархивируем и проверяем возможность импорта
root@lp-ubn1:/home/sadmin# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
root@lp-ubn1:/home/sadmin# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                                ONLINE
          mirror-0                          ONLINE
            /home/sadmin/zpoolexport/filea  ONLINE
            /home/sadmin/zpoolexport/fileb  ONLINE

Сделаем импорт данного пула к нам в ОС:
root@lp-ubn1:/home/sadmin# zpool import -d zpoolexport/ otus
root@lp-ubn1:/home/sadmin# zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                                STATE     READ WRITE CKSUM
        otus                                ONLINE       0     0     0
          mirror-0                          ONLINE       0     0     0
            /home/sadmin/zpoolexport/filea  ONLINE       0     0     0
            /home/sadmin/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

# определяем настройки
zpool get all otus
 zfs get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      11664099306335417737           -
otus  autotrim                       off                            default
otus  compatibility                  off                            default
otus  bcloneused                     0                              -
otus  bclonesaved                    0                              -
otus  bcloneratio                    1.00x                          -
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
otus  feature@redaction_bookmarks    disabled                       local
otus  feature@redacted_datasets      disabled                       local
otus  feature@bookmark_written       disabled                       local
otus  feature@log_spacemap           disabled                       local
otus  feature@livelist               disabled                       local
otus  feature@device_rebuild         disabled                       local
otus  feature@zstd_compress          disabled                       local
otus  feature@draid                  disabled                       local
otus  feature@zilsaxattr             disabled                       local
otus  feature@head_errlog            disabled                       local
otus  feature@blake3                 disabled                       local
otus  feature@block_cloning          disabled                       local
otus  feature@vdev_zaps_v2           disabled                       local
root@lp-ubn1:/home/sadmin#
root@lp-ubn1:/home/sadmin#
root@lp-ubn1:/home/sadmin#
root@lp-ubn1:/home/sadmin#
root@lp-ubn1:/home/sadmin#
root@lp-ubn1:/home/sadmin#  zfs get all otus
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclmode               discard                default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              on                     default
otus  redundant_metadata    all                    default
otus  overlay               on                     default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default




```

### 3. Работа со снапшотом, поиск сообщения от преподавателя

```
# качаем файл
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download


# Восстановим файловую систему из снапшота:
zfs receive otus/test@today < otus_task2.file

#Нашли файл
root@lp-ubn1:/home/sadmin# find /otus/test/ -name "secret_message"
/otus/test/task1/file_mess/secret_message


```





