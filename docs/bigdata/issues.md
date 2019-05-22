# 问题记录

###  解决Ambari 启动 Metrics Monitor 失败的问题

系统Centos7 HDP版本3.1，使用Ambari启动Metrics Monitor时失败，报错如下：

``` shell
Traceback (most recent call last):
  File "/var/lib/ambari-agent/cache/stacks/HDP/3.0/services/AMBARI_METRICS/package/scripts/metrics_monitor.py", line 78, in <module>
    AmsMonitor().execute()
  File "/usr/lib/ambari-agent/lib/resource_management/libraries/script/script.py", line 352, in execute
    method(env)
  File "/var/lib/ambari-agent/cache/stacks/HDP/3.0/services/AMBARI_METRICS/package/scripts/metrics_monitor.py", line 43, in start
    action = 'start'
  File "/usr/lib/ambari-agent/lib/ambari_commons/os_family_impl.py", line 89, in thunk
    return fn(*args, **kwargs)
  File "/var/lib/ambari-agent/cache/stacks/HDP/3.0/services/AMBARI_METRICS/package/scripts/ams_service.py", line 109, in ams_service
    user=params.ams_user
  File "/usr/lib/ambari-agent/lib/resource_management/core/base.py", line 166, in __init__
    self.env.run()
  File "/usr/lib/ambari-agent/lib/resource_management/core/environment.py", line 160, in run
    self.run_action(resource, action)
  File "/usr/lib/ambari-agent/lib/resource_management/core/environment.py", line 124, in run_action
    provider_action()
  File "/usr/lib/ambari-agent/lib/resource_management/core/providers/system.py", line 263, in action_run
    returns=self.resource.returns)
  File "/usr/lib/ambari-agent/lib/resource_management/core/shell.py", line 72, in inner
    result = function(command, **kwargs)
  File "/usr/lib/ambari-agent/lib/resource_management/core/shell.py", line 102, in checked_call
    tries=tries, try_sleep=try_sleep, timeout_kill_strategy=timeout_kill_strategy, returns=returns)
  File "/usr/lib/ambari-agent/lib/resource_management/core/shell.py", line 150, in _call_wrapper
    result = _call(command, **kwargs_copy)
  File "/usr/lib/ambari-agent/lib/resource_management/core/shell.py", line 314, in _call
    raise ExecutionFailed(err_msg, code, out, err)
resource_management.core.exceptions.ExecutionFailed: Execution of '/usr/sbin/ambari-metrics-monitor --config /etc/ambari-metrics-monitor/conf start' returned 255. psutil build directory is not empty, continuing...
Verifying Python version compatibility...
Using python  /usr/bin/python2.7
Checking for previously running Metric Monitor...
/var/run/ambari-metrics-monitor/ambari-metrics-monitor.pid found with no process. Removing 88809...
Starting ambari-metrics-monitor
Verifying ambari-metrics-monitor process status with PID : 5714
Output of PID check : 
ERROR: ambari-metrics-monitor start failed. For more details, see /var/log/ambari-metrics-monitor/ambari-metrics-monitor.out:
====================
    from metric_collector import MetricsCollector
  File "/usr/lib/python2.6/site-packages/resource_monitoring/core/metric_collector.py", line 23, in <module>
    from host_info import HostInfo
  File "/usr/lib/python2.6/site-packages/resource_monitoring/core/host_info.py", line 25, in <module>
    import psutil
  File "/usr/lib/python2.6/site-packages/resource_monitoring/psutil/build/lib.linux-x86_64-2.7/psutil/__init__.py", line 89, in <module>
    import psutil._pslinux as _psplatform
  File "/usr/lib/python2.6/site-packages/resource_monitoring/psutil/build/lib.linux-x86_64-2.7/psutil/_pslinux.py", line 20, in <module>
    from psutil import _common
ImportError: cannot import name _common
====================
Monitor out at: /var/log/ambari-metrics-monitor/ambari-metrics-monitor.out
```

参考官方论坛的 [issue](https://community.hortonworks.com/questions/235118/centos7-hdp31-metrics-monitor-start-importerror-ca.html) 解决，问题可能是由于不同操作系统的包导致冲突，解决办法是 重新构建`psutil`，步骤如下：

``` shell
[root@H33 ~]# grep 'ambari_commons.subprocess' /usr/lib/python2.6/site-packages/resource_monitoring/psutil/build.py 
from ambari_commons.subprocess32 import call  
# 修改这一行，最终结果为
[root@H33 ~]# grep 'subprocess' /usr/lib/python2.6/site-packages/resource_monitoring/psutil/build.py 
# from ambari_commons.subprocess32 import call
from subprocess import call
# 执行脚本重新build psutil
[root@H33 ~]# cd /usr/lib/python2.6/site-packages/resource_monitoring
[root@H33 resource_monitoring]# python psutil/build.py 
```

如果脚本执行成功，就可以直接重新启动了。这时遇到了新问题：

``` shell
Executing make at location: /usr/lib/python2.6/site-packages/resource_monitoring/psutil 
psutil build failed. Please find build output at: /usr/lib/python2.6/site-packages/resource_monitoring/psutil/build.out

cat /usr/lib/python2.6/site-packages/resource_monitoring/psutil/build.out

gcc -pthread -fno-strict-aliasing -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -fPIC -I/usr/include/python2.7 -c psutil/_psutil_linux.c -o build/temp.linux-x86_64-2.7/psutil/_psutil_linux.o
gcc: error trying to exec '/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/cc1': execv: 可执行文件格式错误

file '/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/cc1'
/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/cc1: empty
```

参考google[搜索结果](https://www.centos.org/forums/viewtopic.php?t=56062)

``` shell
[root@H33 resource_monitoring]# yum install cpp # 安装cpp
更新完毕:
  cpp.x86_64 0:4.8.5-36.el7_6.1                                                                                                                        

作为依赖被升级:
  gcc.x86_64 0:4.8.5-36.el7_6.1                  libgcc.x86_64 0:4.8.5-36.el7_6.1                  libgomp.x86_64 0:4.8.5-36.el7_6.1                 

完毕！
[root@H33 resource_monitoring]# python psutil/build.py  #重新构建
# 出现新的报错
gcc -pthread -fno-strict-aliasing -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -fPIC -I/usr/include/python2.7 -c psutil/_psutil_linux.c -o build/temp.linux-x86_64-2.7/psutil/_psutil_linux.o
/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/cc1: error while loading shared libraries: /lib64/libmpc.so.3: file too short
error: command 'gcc' failed with exit status 1

[root@H33 resource_monitoring]# yum provides '/lib64/libmpc.so.3'
libmpc-1.0.1-3.el7.x86_64 : C library for multiple precision complex arithmetic
源    ：installed
匹配来源：
文件名    ：/lib64/libmpc.so.3
[root@H33 psutil]# yum provides '/lib64/libmpc.so.3'
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.njupt.edu.cn
 * extras: ap.stykers.moe
 * updates: ap.stykers.moe
libmpc-1.0.1-3.el7.x86_64 : C library for multiple precision complex arithmetic
源    ：installed
匹配来源：
文件名    ：/lib64/libmpc.so.3



[root@H33 psutil]# yum reinstall libmpc-1.0.1-3.el7.x86_64
[root@H33 psutil]# python  build.py 
gcc -pthread -fno-strict-aliasing -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -DNDEBUG -O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -D_GNU_SOURCE -fPIC -fwrapv -fPIC -I/usr/include/python2.7 -c psutil/_psutil_linux.c -o build/temp.linux-x86_64-2.7/psutil/_psutil_linux.o
/usr/libexec/gcc/x86_64-redhat-linux/4.8.5/cc1: error while loading shared libraries: /lib64/libmpfr.so.4: file too short
[root@H33 psutil]# yum provides '/lib64/libmpfr.so.4'
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.njupt.edu.cn
 * extras: ap.stykers.moe
 * updates: ap.stykers.moe
mpfr-3.1.1-4.el7.x86_64 : A C library for multiple-precision floating-point computations
源    ：installed
匹配来源：
文件名    ：/lib64/libmpfr.so.4

[root@H33 psutil]# yum reinstall mpfr-3.1.1-4.el7.x86_64
[root@H33 psutil]# python  build.py
Executing make at location: /usr/lib/python2.6/site-packages/resource_monitoring/psutil 

```

终于重新build psutil成功，重启Metrics Monitor成功！