#ceph学习 
#### dout(v)
即debug out，调试输出，可以在日志中查看输出内容。
原型：`dout() <- dout_impl()` 或 `dout()  <- ldout() <- dout_impl()`

**dout_impl(cct, sub, v)参数：**
- cct - ceph context
- sub - 子系统名称
- v - 日志级别
可以在ceph.conf中设置输出到日志中的日志级别，日志级别越高代表越详细。

``` C++
#define dout_impl(cct, sub, v)						\
  do {									\
  const bool should_gather = [&](const auto cctX) {			\
    if constexpr (ceph::dout::is_dynamic<decltype(sub)>::value ||	\
		  ceph::dout::is_dynamic<decltype(v)>::value) {		\
      return cctX->_conf->subsys.should_gather(sub, v);			\
    } else {								\
      /* The parentheses are **essential** because commas in angle	\
       * brackets are NOT ignored on macro expansion! A language's	\
       * limitation, sorry. */						\
      return (cctX->_conf->subsys.template should_gather<sub, v>());	\
    }									\
  }(cct);								\
									\
  if (should_gather) {							\
    static size_t _log_exp_length = 80; 				\
    ceph::logging::Entry *_dout_e = 					\
      cct->_log->create_entry(v, sub, &_log_exp_length);		\
    static_assert(std::is_convertible<decltype(&*cct), 			\
				      CephContext* >::value,		\
		  "provided cct must be compatible with CephContext*"); \
    auto _dout_cct = cct;						\
    std::ostream* _dout = &_dout_e->get_ostream();

```

