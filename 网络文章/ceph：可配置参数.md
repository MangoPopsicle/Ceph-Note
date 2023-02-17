#文章 
ceph中的可配置参数（日志级别，缓存设置等等）都定义在[http://options.cc](http://options.cc/)中，可以通过ceph daemon命令来查看或者修改这些参数配置。

```bash
ceph daemon *** config show
```

例如："log\_file"参数可以配置日志文件的路径

```cpp
Option("log_file", Option::TYPE_STR, Option::LEVEL_BASIC)
    .set_default("")
    .set_daemon_default("/var/log/ceph/$cluster-$name.log")
    .set_description("path to log file")
    .add_see_also({"log_to_stderr", "err_to_stderr", "log_to_syslog", "err_to_syslog"}),
```

参数是以Option结构体存在，Option结构体的定义如下

```cpp
struct Option {
  enum type_t {                  // 参数值的类型         
    TYPE_UINT,
    TYPE_INT,
    TYPE_STR,
    TYPE_FLOAT,
    TYPE_BOOL,
    TYPE_ADDR,
    TYPE_UUID,
  };
  enum level_t {
    LEVEL_BASIC,
    LEVEL_ADVANCED,
    LEVEL_DEV,
  };
  using value_t = boost::variant<boost::blank, std::string, uint64_t, int64_t, double, bool, entity_addr_t, uuid_d>;
  const std::string name;                   // 参数名
  const type_t type;                        // 参数类型
  const level_t level;                      // 参数级别
  std::string desc;                         // 该参数的含义
  std::string long_desc;
  value_t value;                            // 参数值
  value_t daemon_value;                     // 有daemon的参数值
  std::vector<const char*> services;        // services即服务类型，比如"mds"
  std::vector<const char*> see_also;            // 该参数对应的类似的参数
  std::vector<const char*> enum_allowed;
  bool safe;
  ...  
}
```

在[http://options.cc](http://options.cc/)中，定义的参数分为几类

```cpp
std::vector<Option> get_global_options() { ... }

std::vector<Option> get_rgw_options() { ... }

static std::vector<Option> get_rbd_options() { ... } 

static std::vector<Option> get_rbd_mirror_options() { ... }

std::vector<Option> get_mds_options() { ... }

std::vector<Option> get_mds_client_options() { ... }
```

从函数名可以看出来：get\_global\_options函数中的参数是global类型的，即基本所有的模块都可以使用这些参数。get\_rgw\_options函数中的参数是给rgw模块使用。get\_rbd\_options函数和get\_rbd\_mirror\_options函数中的参数是给rbd相关模块使用。get\_mds\_options函数中的参数是给mds模块使用。get\_mds\_client\_options函数中的参数是给mds client模块使用。

## 全局变量ceph\_options

ceph\_options包含了所有的参数，

```cpp
const std::vector<Option> ceph_options = build_options();
```

在build\_options中将所有的Option类放入ceph\_options这个vector里面。

```cpp
static std::vector<Option> build_options()
{
  std::vector<Option> result = get_global_options();
  auto ingest = [&result](std::vector<Option>&& options, const char* svc) {
    for (auto &o : options) {
      o.add_service(svc);
      result.push_back(std::move(o));
    }
  };
  ingest(get_rgw_options(), "rgw");
  ingest(get_rbd_options(), "rbd");
  ingest(get_rbd_mirror_options(), "rbd-mirror");
  ingest(get_mds_options(), "mds");
  ingest(get_mds_client_options(), "mds_client");
  return result;
}
```

ceph\_options = { Option{"log\_file"}, Option("\*\*\*")}

## md\_config\_t

md\_config\_t结构体表示当前的Ceph配置，ceph\_options中的参数都会存在这里。在md\_config\_t中有特殊的成员变量定义。

```cpp
#define OPTION_OPT_INT(name) int64_t name;
#define OPTION_OPT_LONGLONG(name) int64_t name;
#define OPTION_OPT_STR(name) std::string name;
#define OPTION_OPT_DOUBLE(name) double name;
#define OPTION_OPT_FLOAT(name) double name;
#define OPTION_OPT_BOOL(name) bool name;
#define OPTION_OPT_ADDR(name) entity_addr_t name;
#define OPTION_OPT_U32(name) uint64_t name;
#define OPTION_OPT_U64(name) uint64_t name;
#define OPTION_OPT_UUID(name) uuid_d name;
#define OPTION(name, ty) \
  public:                      \
    OPTION_##ty(name)          
#define SAFE_OPTION(name, ty) \
  protected:                        \
    OPTION_##ty(name)               
#include "common/legacy_config_opts.h"
#undef OPTION_OPT_INT
#undef OPTION_OPT_LONGLONG
#undef OPTION_OPT_STR
#undef OPTION_OPT_DOUBLE
#undef OPTION_OPT_FLOAT
#undef OPTION_OPT_BOOL
#undef OPTION_OPT_ADDR
#undef OPTION_OPT_U32
#undef OPTION_OPT_U64
#undef OPTION_OPT_UUID
#undef OPTION
#undef SAFE_OPTION
```

common/legacy\_config\_opts.h中存的变量和common/[http://options.cc](http://options.cc/)一一对应，代码如下

```text
/* note: no header guard */
OPTION(host, OPT_STR) // "" means that ceph will use short hostname
OPTION(public_addr, OPT_ADDR)
OPTION(public_bind_addr, OPT_ADDR)
OPTION(cluster_addr, OPT_ADDR)
OPTION(public_network, OPT_STR)
OPTION(cluster_network, OPT_STR)
OPTION(lockdep, OPT_BOOL)
OPTION(lockdep_force_backtrace, OPT_BOOL) // always gather current backtrace at every lock
OPTION(run_dir, OPT_STR)       // the "/var/run/ceph" dir, created on daemon startup
OPTION(admin_socket, OPT_STR) // default changed by common_preinit()
OPTION(admin_socket_mode, OPT_STR) // permission bits to set for admin socket file, e.g., "0775", "0755"
......
```

最终md\_config\_t结构体定义如下

```text
struct md_config_t {
   ...
   typedef std::multimap <std::string, md_config_obs_t*> obs_map_t;
   std::map<std::string, md_config_t::member_ptr_t> legacy_values;
   std::map<std::string, const Option&> schema;
   std::map<std::string, Option::value_t> values;
private:
  ConfFile cf;
public:
  std::deque<std::string> parse_errors;
private:
  bool safe_to_start_threads = false;
  obs_map_t observers;
  changed_set_t changed;
public:
  ceph::logging::SubsystemMap subsys;
  EntityName name;
  string data_dir_option;  ///< data_dir config option, if any
  string cluster;           // cluster = "ceph"

// 下面的成员来自common/legacy_config_opts.h中
public：
   std::string host;
   entity_addr_t public_addr;
   entity_addr_t public_bind_addr;
   entity_addr_t cluster_addr;
   std::string public_network;
   std::string cluster_network;
   bool lockdep;
   bool lockdep_force_backtrace;
   std::string run_dir;
   std::string admin_socket;
   std::string admin_socket_mode;
   std::string log_file; 
   ...
protected:
   std::string ms_type; 
   std::string plugin_dir;
   ...

}
```

其中schema保存ceph\_options中的所有Option参数，即

`schema = { <"host", Options("host")>, ..., <"log\_file", Option("log\_file")>, ...}

`legacy\_values`将`ceph\_options`中的`Option`的`name`与对应的`md\_config\_t`中的成员指针作为key-value保存，即

`legacy\_values = {<"host", &md\_config\_t::host>, ... , <"log\_file", &md\_config\_t::log\_file>, ...}

`values`将`ceph\_options`中的`Option`的`name`和`value`作为key-value保存。

libcephfs中：`values = {<"host", "">, ..., <"log\_file", "">, ...}

ceph-mds中：`values = {<"host", "">, ..., <"log\_file", "/var/log/ceph/$cluster-$name.log">, ...}

subsys用来存日志项，即
```
subsys = ceph::logging::SubsystemMap( 
	m\_subsys\[ceph\_subsys\_ == 0\] = Subsystem<name = "none",	log\_level = 0, gather\_level = 5>, 
	... , 
	m\_subsys\[ceph\_subsys\_crush == 3\] = Subsystem<name = "crush",	log\_level = 1, gather\_level = 1>, 
	m\_subsys\[ceph\_subsys\_mds == 4\] = Subsystem<name = "mds", log\_level = 1, gather\_level = 5>, 
	... }。
```


## g\_ceph\_context和g\_conf全局变量

一个模块的进程会有多个线程，比如ceph-mds，进程中有些内容需要整个进程中的所有线程都可以访问，比如参数配置和以及上下文内容，所以就有两个全局变量g\_conf和g\_ceph\_context，在src/global/[http://global\_context.cc](http://global_context.cc/)中定义如下

```cpp
CephContext *g_ceph_context = NULL;
md_config_t *g_conf = NULL;
```

md\_config\_t 实例对象指针又作为CephContext 的成员，

```cpp
class CephContext {
...
public:
  md_config_t *_conf;
...
}
```

每个模块启动时，都需要实例化CephContext，就不需要单独去实例化md\_config\_t。CephContext实例化函数流程如下：

![[Pasted image 20220920175052.png]]

在md\_config\_t的构造函数的入参为is\_daemon，它是用来判断该模块是单独起一个守护线程，或者由别的线程去调用对外接口，不单独起线程。构造函数如下

```text
md_config_t::md_config_t(bool is_daemon){ ... }
```

md\_config\_t的实例化处如下

```cpp
 _conf(new md_config_t(code_env == CODE_ENVIRONMENT_DAEMON))
```

即判断code\_env == CODE\_ENVIRONMENT\_DAEMON，在各个模块的main函数中有code\_env的入参。即[http://ceph\_mds.cc](http://ceph_mds.cc/)，[http://ceph\_fuse.cc](http://ceph_fuse.cc/)和[http://ceph\_mon.cc](http://ceph_mon.cc/)等中的md\_config\_t中is\_daemon都是true，libcephfs的md\_config\_t中is\_daemon是false。

## 参数配置方式

参数配置方式有两种：永久和临时。永久方式，就是在配置文件(如ceph.conf)中添加该参数的配置，重启进程后，参数就生效了。临时方式，就是通过ceph daemon命令设置内存中的参数。

修改ceph.conf中参数后怎么生效, 比如在ceph.conf中配置了

```text
admin_socket = $rundir/$cluster-$id.$pid.asok
```

直接看代码，从[http://ceph\_mon.cc](http://ceph_mon.cc/)中开始看

```cpp
auto cct = global_init(&def_args, args, CEPH_ENTITY_TYPE_MON, CODE_ENVIRONMENT_DAEMON,
			 flags, "mon_data")
```

global\_init

```cpp
boost::intrusive_ptr<CephContext> global_init(std::vector < const char * > *alt_def_args, std::vector < const char* >& args,
	    uint32_t module_type, code_environment_t code_env, int flags,
	    const char *data_dir_option, bool run_pre_init)
{ // module_type = CEPH_ENTITY_TYPE_MON, code_env = CODE_ENVIRONMENT_DAEMON, flags = 0
  // *data_dir_option = "mon_data", run_pre_init = true
  static bool first_run = true;
  if (run_pre_init) {
    global_pre_init(alt_def_args, args, module_type, code_env, flags);
  } else 
  ...
}
```

global\_pre\_init是关键函数：

```cpp
void global_pre_init(std::vector < const char * > *alt_def_args, std::vector < const char* >& args,
		     uint32_t module_type, code_environment_t code_env, int flags)
{ // module_type = CEPH_ENTITY_TYPE_MON, code_env = CODE_ENVIRONMENT_DAEMON, flags = 0
  std::string conf_file_list;            // ceph.conf的路径
  std::string cluster = "";
  CephInitParameters iparams = ceph_argparse_early_args(args, module_type, &cluster, &conf_file_list);
  // common_preinit函数主要是去实例化CephContext
  CephContext *cct = common_preinit(iparams, code_env, flags);
  cct->_conf->cluster = cluster;         // cct->_conf->cluster = "ceph"
  global_init_set_globals(cct);
  md_config_t *conf = cct->_conf;
  if (alt_def_args)
    conf->parse_argv(*alt_def_args);  // alternative default args
  int ret = conf->parse_config_files(c_str_or_null(conf_file_list), &cerr, flags);
  ...
}
```

调用ceph\_argparse\_early\_args来解析进程启动时设置的参数，比如ceph-mon启动时设置的参数如下

```bash
/usr/bin/ceph-mon -f --cluster ceph --id node1 --setuser ceph --setgroup ceph
```

ceph\_argparse\_early\_args函数如下，

```cpp
CephInitParameters ceph_argparse_early_args (std::vector<const char*>& args, uint32_t module_type,
	   std::string *cluster, std::string *conf_file_list)
{ //module_type = CEPH_ENTITY_TYPE_MON, *cluster = "", *conf_file_list = ""
  CephInitParameters iparams(module_type);
  // iparams里面的内容为class CephInitParameters { module_type = CEPH_ENTITY_TYPE_MON，
  //   name: struct EntityName { type = CEPH_ENTITY_TYPE_MON, id = "admin", type_id = "mon.admin" }}
  std::string val;
  vector<const char *> orig_args = args;
  for (std::vector<const char*>::iterator i = args.begin(); i != args.end(); ) {
    ...
   // 解析--conf,但是执行ceph-mon命令时，并未指定--conf，所以val为空字符串。
    else if (ceph_argparse_witharg(args, i, &val, "--conf", "-c", (char*)NULL)) {
      *conf_file_list = val;       
    }
    else if (ceph_argparse_witharg(args, i, &val, "--cluster", (char*)NULL)) {
      *cluster = val;         // *cluster = ceph
    }
    else if ((module_type != CEPH_ENTITY_TYPE_CLIENT) &&
	     (ceph_argparse_witharg(args, i, &val, "-i", (char*)NULL))) {
      iparams.name.set_id(val); 
    }
    else if (ceph_argparse_witharg(args, i, &val, "--id", "--user", (char*)NULL)) {
      iparams.name.set_id(val); // iparams.name中{_id = "node1", type_id = "mon.node1"}
    }
    ...
  return iparams;
}
```

common\_preinit

```cpp
CephContext *common_preinit(const CephInitParameters &iparams, enum code_environment_t code_env, int flags)
{ // code_env = CODE_ENVIRONMENT_DAEMON, flags = 0
  // 实例CephContext
  CephContext *cct = new CephContext(iparams.module_type, code_env, flags);
  md_config_t *conf = cct->_conf;
  conf->name = iparams.name;
  ...
  return cct;
}
```

最重要的就是parse\_config\_files

```cpp
int md_config_t::parse_config_files(const char *conf_files, std::ostream *warnings, int flags)
{ // conf_files = NULL
  ...
  if (!conf_files) {
    const char *c = getenv("CEPH_CONF");       // c = NULL
    ...
    else {
      ...
      // CEPH_CONF_FILE_DEFAULT = "$data_dir/config, /etc/ceph/$cluster.conf, ~/.ceph/$cluster.conf, $cluster.conf"
      conf_files = CEPH_CONF_FILE_DEFAULT; // CEPH_CONF_FILE_DEFAULT就是配置文件的路径
    }
  }
  std::list<std::string> cfl;
  get_str_list(conf_files, cfl);     // 将conf_files中的字符串切割放在cfl中
  auto p = cfl.begin();
  while (p != cfl.end()) { ... }
  return parse_config_files_impl(cfl, warnings);
}
```

md\_config\_t::parse\_config\_files\_impl

```cpp
int md_config_t::parse_config_files_impl(const std::list<std::string> &conf_files,std::ostream *warnings)
{
  list<string>::const_iterator c;
  // 遍历conf_files，替换"$"后面的字符串，比如将$cluster替换成ceph
  for (c = conf_files.begin(); c != conf_files.end(); ++c) {
    cf.clear();
    string fn = *c;
    // 替换"$"后面的字符串
    expand_meta(fn, warnings);
    // 将ceph.conf(如/etc/ceph/ceph.conf)中的内容读上来保存在cf中
    int ret = cf.parse_file(fn.c_str(), &parse_errors, warnings);
    if (ret == 0) break;
  }

  std::vector <std::string> my_sections;
  // 获取要解析的部分，比如mon：<"mon.node1", "mon", global">, client: <"client.admin", "admin", "global">
  // 即ceph.conf中my_sections对应的部分才会解析
  _get_my_sections(my_sections);
  for (const auto &i : schema) { ... }
  //遍历subsys, 配置debug_**之类的
  for (size_t o = 0; o < subsys.get_num(); o++) { ... }
  ...
  return 0;
}
```

md\_config\_t::expand\_meta函数用来替换"$"后面的字符串，比如admin\_socket = $rundir/$cluster-$id.$pid.asok，调用expand\_meta函数后admin\_socket = /var/run/ceph/ceph-mon.node1.2093242.asok