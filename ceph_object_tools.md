Allowed options:
`--help`
produce help message

`--type arg`
Arg is one of `[bluestore (default), filestore, memstore]`

`--data-path arg`             
path to object store, mandatory

`--journal-path arg`          
path to journal, use if tool can't find it

`--pgid arg`                  
PG id, mandatory for info, log, remove, export,export-remove, mark-complete, trim-pg-log, and mandatory for apply-layout-settings if --pool is not specified

`--pool arg` 
Pool name, mandatory for apply-layout-settings if --pgid is not specified
`--op arg`
Arg is one of `info`, `log`, `remove`, `mkfs`,` fsck`, `repair`, `fuse`, `dup`, `export`, `export-remove`, `import`,`list`, `fix-lost`, `list-pgs`, `dump-journal`, `dump-super`, `meta-list`, `get-osdmap`, `set-osdmap`, `get-inc-osdmap`, `set-inc-osdmap`, `mark-complete`, `apply-layout-settings`, `update-mon-db`, `dump-import`, `trim-pg-log`


`--epoch arg` epoch# for get-osdmap and get-inc-osdmap, the 
current epoch in use if not specified

`--file arg`
path of file to export, export-remove, import, get-osdmap, set-osdmap, get-inc-osdmap or set-inc-osdmap

`--mon-store-path arg`
path of monstore to update-mon-db

`--fsid arg` 
fsid for new store created by mkfs

`--target-data-path arg` 
path of target object store (for --op dup)
`--mountpoint arg`
fuse mountpoint

`--format arg` 
(=json-pretty) Output format which may be json, json-pretty, xml, xml-pretty

`--debug` 
Enable diagnostic output to stderr

`--force`
Ignore some types of errors and proceed with operation - USE WITH CAUTION: CORRUPTION POSSIBLE NOW OR IN THE FUTURE

`--skip-journal-replay`
Disable journal replay

`--skip-mount-omap` 
Disable mounting of omap

`--head`
Find head/snapdir when searching for objects by name

`--dry-run`
Don't modify the objectstore

`--namespace arg`
Specify namespace when searching for objects


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
ceph-objectstore-tool ... <object> remove-clone-metadata <cloneid>
