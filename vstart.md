#ceph命令 

按照默认的方式创建一个集群
```
sh ../src/vstart.sh -n -d
```

按照原来的方式创建一个集群
```
sh ../src/vstart.sh -N -d
```

-n : 创建一个新的集群
-N : 按照原有的配置创建一个集群

```
MON=3 OSD=3 MDS=1 MGR=1 RGW=1 ../src/vstart.sh -n -d --rgw_port 7788
```

停止环境
```
[root@localhost build]# sh ../src/stop.sh --
usage: ../src/stop.sh [all] [mon] [mds] [osd] [rgw]
```


```
ceph-mgr dashboard not built - disabling.
usage: ../src/vstart.sh [option]... 
ex: MON=3 OSD=1 MDS=1 MGR=1 RGW=1 ../src/vstart.sh -n -d
options:
	-d, --debug
	-s, --standby_mds: Generate standby-replay MDS for each active
	-l, --localhost: use localhost instead of hostname
	-i <ip>: bind to specific ip
	-n, --new
	-N, --not-new: reuse existing cluster config (default)
	--valgrind[_{osd,mds,mon,rgw}] 'toolname args...'
	--nodaemon: use ceph-run as wrapper for mon/osd/mds
	--smallmds: limit mds cache size
	-m ip:port		specify monitor address
	-k keep old configuration files
	-x enable cephx (on by default)
	-X disable cephx
	--hitset <pool> <hit_set_type>: enable hitset tracking
	-e : create an erasure pool
	-o config		 add extra config parameters to all sections
	--rgw_port specify ceph rgw http listen port
	--rgw_frontend specify the rgw frontend configuration
	--rgw_compression specify the rgw compression plugin
	-b, --bluestore use bluestore as the osd objectstore backend (default)
	-f, --filestore use filestore as the osd objectstore backend
	-K, --kstore use kstore as the osd objectstore backend
	--memstore use memstore as the osd objectstore backend
	--cache <pool>: enable cache tiering on pool
	--short: short object names only; necessary for ext4 dev
	--nolockdep disable lockdep
	--multimds <count> allow multimds with maximum active count
	--without-dashboard: do not run using mgr dashboard

```
