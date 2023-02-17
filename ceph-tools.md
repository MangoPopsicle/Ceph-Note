#ceph命令

![[ceph命令.xlsx]]
[【腾讯文档】object_tools](https://docs.qq.com/sheet/DSVpaaEZZbm93Rlhz?tab=BB08J2)


#### ceph-objectstore-tool
[ceph/ceph-objectstore-tool.rst at main · ceph/ceph (github.com)](https://github.com/ceph/ceph/blob/main/doc/man/8/ceph-objectstore-tool.rst)
⚠ **==命令使用之前需要先停掉当前osd，否则执行命令会报错==** ⚠️
针对单个pg，列出当前pg的元数据信息
```
./ceph-objectstore-tool --data-path [osd-path] --type bluestore --pgid [pgid] --op log
```

```
--help
--
```


```
./ceph-objectstore-tool --help

Allowed options:
  --help                      produce help message
  --type arg                  Arg is one of [bluestore (default), filestore, 
                              memstore]
  --data-path arg             path to object store, mandatory
  --journal-path arg          path to journal, use if tool can't find it
  --pgid arg                  PG id, mandatory for info, log, remove, export, 
                              export-remove, mark-complete, trim-pg-log, and 
                              mandatory for apply-layout-settings if --pool is 
                              not specified
  --pool arg                  Pool name, mandatory for apply-layout-settings if
                              --pgid is not specified
  --op arg                    Arg is one of [info, log, remove, mkfs, fsck, 
                              repair, fuse, dup, export, export-remove, import,
                              list, fix-lost, list-pgs, dump-journal, 
                              dump-super, meta-list, get-osdmap, set-osdmap, 
                              get-inc-osdmap, set-inc-osdmap, mark-complete, 
                              reset-last-complete, apply-layout-settings, 
                              update-mon-db, dump-export, trim-pg-log]
  --epoch arg                 epoch# for get-osdmap and get-inc-osdmap, the 
                              current epoch in use if not specified
  --file arg                  path of file to export, export-remove, import, 
                              get-osdmap, set-osdmap, get-inc-osdmap or 
                              set-inc-osdmap
  --mon-store-path arg        path of monstore to update-mon-db
  --fsid arg                  fsid for new store created by mkfs
  --target-data-path arg      path of target object store (for --op dup)
  --mountpoint arg            fuse mountpoint
  --format arg (=json-pretty) Output format which may be json, json-pretty, 
                              xml, xml-pretty
  --debug                     Enable diagnostic output to stderr
  --force                     Ignore some types of errors and proceed with 
                              operation - USE WITH CAUTION: CORRUPTION POSSIBLE
                              NOW OR IN THE FUTURE
  --skip-journal-replay       Disable journal replay
  --skip-mount-omap           Disable mounting of omap
  --head                      Find head/snapdir when searching for objects by 
                              name
  --dry-run                   Don't modify the objectstore
  --namespace arg             Specify namespace when searching for objects
  --rmtype arg                Specify corrupting object removal 'snapmap' or 
                              'nosnapmap' - TESTING USE ONLY


Positional syntax:

ceph-objectstore-tool ... <object> (get|set)-bytes [file]
ceph-objectstore-tool ... <object> set-(attr|omap) <key> [file]
ceph-objectstore-tool ... <object> (get|rm)-(attr|omap) <key>
ceph-objectstore-tool ... <object> get-omaphdr
ceph-objectstore-tool ... <object> set-omaphdr [file]
ceph-objectstore-tool ... <object> list-attrs
ceph-objectstore-tool ... <object> list-omap
ceph-objectstore-tool ... <object> remove|removeall
ceph-objectstore-tool ... <object> dump
ceph-objectstore-tool ... <object> set-size
ceph-objectstore-tool ... <object> clear-data-digest
ceph-objectstore-tool ... <object> remove-clone-metadata <cloneid>

<object> can be a JSON object description as displayed
by --op list.
<object> can be an object name which will be looked up in all
the OSD's PGs.
<object> can be the empty string ('') which with a provided pgid 
specifies the pgmeta object

The optional [file] argument will read stdin or write stdout
if not specified or if '-' specified.
```



#### ceph-kvstore-tool
⚠ **==命令使用之前需要先停掉mon，否则执行命令会报错==** ⚠️
[object_tools (qq.com)](https://docs.qq.com/sheet/DSVpaaEZZbm93Rlhz?tab=8v8kaz&u=55df0bb604a241398fa4f8f4fba6408b)
[ceph-kvstore-tool 工具使用详解_z_stand的博客-CSDN博客](https://blog.csdn.net/Z_Stand/article/details/98967671)
```
Usage: ./ceph-kvstore-tool <leveldb|rocksdb|bluestore-kv> <store path> command [args...]

Commands:
  list [prefix]
  list-crc [prefix]
  dump [prefix]
  exists <prefix> [key]
  get <prefix> <key> [out <file>]
  crc <prefix> <key>
  get-size [<prefix> <key>]
  set <prefix> <key> [ver <N>|in <file>]
  rm <prefix> <key>
  rm-prefix <prefix>
  store-copy <path> [num-keys-per-tx] [leveldb|rocksdb|...] 
  store-crc <path>
  compact
  compact-prefix <prefix>
  compact-range <prefix> <start> <end>
  destructive-repair  (use only as last resort! may corrupt healthy data)
  stats
```


#ceph-dencoder 脚本：
``` bash
#! /bin/bash

for i in `../bin/ceph-dencoder list_types`;
 do echo --------------$i-----------------;
 ../bin/ceph-dencoder type $i import $1 decode dump_json 2>/dev/null ;
 done
```

prefix: mon维护的表
key: prefix内容的版本
value:  内容
