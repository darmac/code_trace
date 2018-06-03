# Power Magager Notes

## Power Manager 的组成结构：
```
Power Supply : Battery&Charger Driver Framework

PM Core
	Generic PM
		power off
		reboot
		hibernate
		suspend(STD/STR)

	Runtime PM

	CPU Idle

	Wakelock
	
PM QOS

Drivers
	PM callback operations

Power Saving
	CPU Freq, Dev Freq : cpu/dev 电压频率调节
	OPP(Operating Performance Point) : 是指可以使SOCs或者Devices正常工作的电压和频率组合
	Clock Framework : clock driver framework
	Regulator Framework : voltage/current regulator driver framework
```

## Reboot Flow

### search reboot entry
Reboot system call:

* **SyS_reboot()** / **sys_reboot()** in System.map (查找系统调用)

* addr2line 获取代码位置 
[kernel/reboot.c:280]
```
SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,
		void __user *, arg)
```

* sys_reboot() 具体定义为
```
sys_reboot(int magic1, int magic2, unsigned int cmd, void __user* arg)
```

* 分析 sys_reboot() code，找到 cmd 列表
```
/*
 * Commands accepted by the _reboot() system call.
 *
 * RESTART     Restart system using default command and mode.
 * HALT        Stop OS and give system control to ROM monitor, if any.
 * CAD_ON      Ctrl-Alt-Del sequence causes RESTART command.
 * CAD_OFF     Ctrl-Alt-Del sequence sends SIGINT to init task.
 * POWER_OFF   Stop OS and remove all power from system, if possible.
 * RESTART2    Restart system using given command string.
 * SW_SUSPEND  Suspend system using software suspend if compiled in.
 * KEXEC       Restart system using a previously loaded Linux kernel
 */

#define	LINUX_REBOOT_CMD_RESTART	0x01234567
#define	LINUX_REBOOT_CMD_HALT		0xCDEF0123
#define	LINUX_REBOOT_CMD_CAD_ON		0x89ABCDEF
#define	LINUX_REBOOT_CMD_CAD_OFF	0x00000000
#define	LINUX_REBOOT_CMD_POWER_OFF	0x4321FEDC
#define	LINUX_REBOOT_CMD_RESTART2	0xA1B2C3D4
#define	LINUX_REBOOT_CMD_SW_SUSPEND	0xD000FCE2
#define	LINUX_REBOOT_CMD_KEXEC		0x45584543
```

* RESTART / RESTART2 分别调用如下 flow
```
	case LINUX_REBOOT_CMD_RESTART:
		kernel_restart(NULL);
		break;
		
	case LINUX_REBOOT_CMD_RESTART2:
		ret = strncpy_from_user(&buffer[0], arg, sizeof(buffer) - 1);
		...
		kernel_restart(buffer);
		break;
```

### reboot core code

* kernel_restart解析
```
[kernel/reboot.c]
214 void kernel_restart(char *cmd)
215 {
216     kernel_restart_prepare(cmd);
217     migrate_to_reboot_cpu();
218     syscore_shutdown();
219     if (!cmd)
220         pr_emerg("Restarting system\n");
221     else
222         pr_emerg("Restarting system with command '%s'\n", cmd);
223     kmsg_dump(KMSG_DUMP_RESTART);
224     machine_restart(cmd);
225 }
```

* line216 : kernel_restart_prepare()
```
[kernel/reboot.c]
 68 void kernel_restart_prepare(char *cmd)
 69 {
 70     blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
 71     system_state = SYSTEM_RESTART;
 72     usermodehelper_disable();
 73     device_shutdown();
 74 }
```
line70 : blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
依次回调注册到 reboot_notifier_list 的所有函数，通知 restart 事件。

line72 : usermodehelper_disable() 
```
[include/linux/umh.h]
 55 static inline int usermodehelper_disable(void)
 56 {
 57     return __usermodehelper_disable(UMH_DISABLED);
 58 }
```

```
[kernel/umh.c]
305 int __usermodehelper_disable(enum umh_disable_depth depth)
306 {
307     long retval;
308 
309     if (!depth)
310         return -EINVAL;
311 
312     down_write(&umhelper_sem);
313     usermodehelper_disabled = depth;
314     up_write(&umhelper_sem);
315 
...
322     retval = wait_event_timeout(running_helpers_waitq,
323                     atomic_read(&running_helpers) == 0,
324                     RUNNING_HELPERS_TIMEOUT);
325     if (retval)
326         return 0;
327 
328     __usermodehelper_set_disable_depth(UMH_ENABLED);
329     return -EAGAIN;
330 }
```

尝试禁用 usermodehelper：1）禁用 umh 2）等待 timeout 5S 确定 umh 全部结束。

line73 : device_shutdown()
```
[drivers/base/core.c]
2765 void device_shutdown(void)
2766 {
2767     struct device *dev, *parent;
...
2775     while (!list_empty(&devices_kset->list)) {
2776         dev = list_entry(devices_kset->list.prev, struct device,
2777                 kobj.entry);
...
2790         list_del_init(&dev->kobj.entry);
...
2799         pm_runtime_get_noresume(dev);
2800         pm_runtime_barrier(dev);
2801 
2802         if (dev->class && dev->class->shutdown_pre) {
2803             if (initcall_debug)
2804                 dev_info(dev, "shutdown_pre\n");
2805             dev->class->shutdown_pre(dev);
2806         }
2807         if (dev->bus && dev->bus->shutdown) {
2808             if (initcall_debug)
2809                 dev_info(dev, "shutdown\n");
2810             dev->bus->shutdown(dev);
2811         } else if (dev->driver && dev->driver->shutdown) {
2812             if (initcall_debug)
2813                 dev_info(dev, "shutdown\n");
2814             dev->driver->shutdown(dev);
2815         }
...
2825     }
...
2827 }
```
遍历 devices_kset：
1）从 devices_kset 中摘除 dev kobj : list_del_init(&dev->kobj.entry)
2）禁用 runtime suspend
3）dev->class->shutdown_pre() （如果存在）
4）dev->bus->shutdown()  若没有则 dev->driver->shutdown() （如果存在）

* line217 : migrate_to_reboot_cpu()
合并所有任务到 reboot cpu，具体实现先行跳过。

* line218 : syscore_shutdown()
```
[drivers/base/syscore.c]
117 void syscore_shutdown(void)
118 {
119     struct syscore_ops *ops;
120 
121     mutex_lock(&syscore_ops_lock);
122 
123     list_for_each_entry_reverse(ops, &syscore_ops_list, node)
124         if (ops->shutdown) {
125             if (initcall_debug)
126                 pr_info("PM: Calling %pF\n", ops->shutdown);
127             ops->shutdown();
128         }
129 
130     mutex_unlock(&syscore_ops_lock);
131 }
```
依次调用所有挂在 syscore_ops_list 的 struct syscore_ops 变量的 shutdown() 函数。

* line223 : kmsg_dump()
```
[kernel/printk/printk.c]
2848 void kmsg_dump(enum kmsg_dump_reason reason)
2849 {
2850     struct kmsg_dumper *dumper;
2851     unsigned long flags;
...
2856     rcu_read_lock();
2857     list_for_each_entry_rcu(dumper, &dump_list, list) {
...
2871         /* invoke dumper which will iterate over records */
2872         dumper->dump(dumper, reason);
...
2876     }
2877     rcu_read_unlock();
2878 }
```
调用所有挂在 dump_list 中的 struct kmsg_dumper 的 dump() 函数。

* line224 : machine_restart()
跟平台相关的 restart 函数，对 arm 来说如下：
```
[arch/arm/kernel/reboot.c]
139 void machine_restart(char *cmd)
140 {
141     local_irq_disable();
142     smp_send_stop();
143 
144     if (arm_pm_restart)
145         arm_pm_restart(reboot_mode, cmd);
146     else
147         do_kernel_restart(cmd);
148 
149     /* Give a grace period for failure to restart of 1s */
150     mdelay(1000);
151 
152     /* Whoops - the platform was unable to reboot. Tell the user! */
153     printk("Reboot failed -- System halted\n");
154     while (1);
155 }
```

1）关本地中断
2）停止 smp
3）arm_pm_restart() / do_kernel_restart() 按优先级择其一执行。
note : vexpress 平台未注册 arm_pm_restart() 。
```
[kernel/reboot.c]
183 void do_kernel_restart(char *cmd)
184 {
185     atomic_notifier_call_chain(&restart_handler_list, reboot_mode, cmd);
186 }
```
4）while(1) //正常流程不会执行至此。

###调研各个 notify list 注册的函数