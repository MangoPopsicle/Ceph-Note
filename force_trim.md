#ceph学习 

legacy_config_opts.h：各种option

``` c++
legacy_config_opts.h (src\common) line 284 : 
OPTION(mon_osd_force_trim_to, OPT_INT)   
// force mon to trim maps to this point, regardless of min_last_epoch_clean (dangerous)

legacy_config_opts.h (src\common) line 285 : 
OPTION(mon_mds_force_trim_to, OPT_INT)   
// force mon to trim mdsmaps to this point (dangerous)
```
