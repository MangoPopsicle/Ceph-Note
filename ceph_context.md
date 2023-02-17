#ceph学习 
A CephContext represents a single view of the Ceph cluster. It comes complete with a configuration, a set of performance counters (PerfCounters), and a heartbeat map. You can find more information about CephContext in src/common/ceph_context.h.

Generally, you will have only one CephContext in your application, called g_ceph_context. However, in library code, it is possible that the library user will initialize multiple CephContexts. For example, this would happen if he called rados_create more than once.

A ceph context is required to issue log messages. Why is this? Well, without the CephContext, we would not know which log messages were disabled and which were enabled. The dout() macro implicitly references g_ceph_context, so it can’t be used in library code. It is fine to use dout and derr in daemons, but in library code, you must use ldout and lderr, and pass in your own CephContext object. The compiler will enforce this restriction.

CephContext 代表 Ceph 集群的单一视图。它带有一个配置、一组性能计数器 (PerfCounters) 和一个心跳图。您可以在 src/common/ceph_context.h 中找到有关 CephContext 的更多信息。

通常，您的应用进程中只有一个 CephContext，称为 g_ceph_context。但是，在库代码中，库用户可能会初始化多个 CephContext。例如，如果他多次调用 rados_create，就会发生这种情况。

发出日志消息需要 ceph 上下文。为什幺是这样？好吧，如果没有 CephContext，我们将不知道哪些日志消息被禁用，哪些被启用。 dout() 宏隐式引用 g_ceph_context，因此不能在库代码中使用。在守护进程中使用dout 和derr 没问题，但是在库代码中，你必须使用ldout 和lderr，并传入你自己的CephContext 对象。编译器将强制执行此限制。

```
/* A CephContext represents the context held by a single library user（库用户）.
 * There can be multiple CephContexts in the same process.
 *  在同一个进程中可以有多个cephcontext
 *  但在守护进程和实体进程（？）中只有一个cephcontext
 * For daemons and utility programs, there will be only one CephContext.  
 * The * CephContext contains the configuration, the dout object（？）, and anything else
 * that you might want to pass to libcommon with every function call（函数调用）.
 */
```
%% 位置：src\common\ceph_context.h %%