
## Power Management Interface

### struct dev_pm_ops & struct dev_pm_domain
- dev_pm_ops 结构体定义：
note：原 code 提供了使用说明，可以进行详细参考。
```
[include/linux/pm.h]
290 struct dev_pm_ops {
291     int (*prepare)(struct device *dev);
292     void (*complete)(struct device *dev);
293     int (*suspend)(struct device *dev);
294     int (*resume)(struct device *dev);
295     int (*freeze)(struct device *dev);
296     int (*thaw)(struct device *dev);
297     int (*poweroff)(struct device *dev);
298     int (*restore)(struct device *dev);
299     int (*suspend_late)(struct device *dev);
300     int (*resume_early)(struct device *dev);
301     int (*freeze_late)(struct device *dev);
302     int (*thaw_early)(struct device *dev);
303     int (*poweroff_late)(struct device *dev);
304     int (*restore_early)(struct device *dev);
305     int (*suspend_noirq)(struct device *dev);
306     int (*resume_noirq)(struct device *dev);
307     int (*freeze_noirq)(struct device *dev);
308     int (*thaw_noirq)(struct device *dev);
309     int (*poweroff_noirq)(struct device *dev);
310     int (*restore_noirq)(struct device *dev);
311     int (*runtime_suspend)(struct device *dev);
312     int (*runtime_resume)(struct device *dev);
313     int (*runtime_idle)(struct device *dev);
314 };
```
对struct dev_pm_ops callback 可分类为：
```
	/*suspend (suspend - resume)*/
	int (*suspend)(struct device *dev);
	int (*suspend_late)(struct device *dev);
	int (*suspend_noirq)(struct device *dev);
	/*resume (suspend - resume)*/
	int (*resume_noirq)(struct device *dev);
	int (*resume_early)(struct device *dev);
	int (*resume)(struct device *dev);

	/*freeze (freeze - thaw)*/
	int (*freeze)(struct device *dev);
	int (*freeze_late)(struct device *dev);
	int (*freeze_noirq)(struct device *dev);
	/*thaw (freeze - thaw)*/
	int (*thaw_noirq)(struct device *dev);
	int (*thaw_early)(struct device *dev);
	int (*thaw)(struct device *dev);
	
	/*power off (power_off - restore)*/
	int (*poweroff)(struct device *dev);
	int (*poweroff_late)(struct device *dev);
	int (*poweroff_noirq)(struct device *dev);
	/*restore (power_off - restore)*/
	int (*restore_noirq)(struct device *dev);
	int (*restore_early)(struct device *dev);
	int (*restore)(struct device *dev);
	
	/*all ops prepare and complete*/
	int (*prepare)(struct device *dev);
	void (*complete)(struct device *dev);
	
	/*runtime pm related*/
	int (*runtime_suspend)(struct device *dev);
	int (*runtime_resume)(struct device *dev);
	int (*runtime_idle)(struct device *dev);
```
- suspend/resume 阶段的 dev_pm_ops 调用顺序：

	**prepare()** -> suspend() -> suspend_late() -> suspend_noirq()
	wakup
	resume_noirq() -> resume_early() -> resume() -> **complete()**

- dev_pm_domain 结构体定义：
```
626 struct dev_pm_domain {
627     struct dev_pm_ops   ops;
628     void (*detach)(struct device *dev, bool power_off);
629     int (*activate)(struct device *dev);
630     void (*sync)(struct device *dev);
631     void (*dismiss)(struct device *dev);
632 };
```
(*detach)() : 从 pm domain 中移除时调用
(*activate)() : 在 bus type 或 driver 执行 probe 动作之前调用
(*sync)() : probe 动作返回成功之后调用
(*dismiss)() : probe 动作返回失败时调用

### dpm realted api
- dpm suspend/resume related

-- 对 dev_pm_ops 的封装：遍历各链表，并调用其 callback func
```
dpm_prepare();
	遍历 dpm_list，call device_prepare()
dpm_suspend();
	遍历 dpm_prepared_list，call device_suspend()
dpm_suspend_late();
	遍历 dpm_suspended_list， call device_suspend_late()
dpm_suspend_noirq();
	遍历 irq_desc[]，并 disable desc irq
	遍历 dpm_late_early_list，call device_suspend_noirq()

dpm_resume_noirq();
	遍历 dpm_noirq_list，call device_resume_noirq()
	遍历 irq_desc[]，并 enable desc irq
dpm_resume_early();
	遍历 dpm_late_early_list
dpm_resume();
	遍历 dpm_suspended_list
dpm_complete();
	遍历 dpm_prepared_list
```

-- suspend/resume list 生成及还原
```
dpm_list <-> dpm_prepared_list <-> dpm_suspended_list <-> dpm_late_early_list <-> dpm_noirq_list
```

-- callback func 优先级
struct device 结构体中各个 pm callback 优先级为：
pm_domain > type > class > bus > driver

-- 自有封装：
```
dpm_suspend_start() 包括如下 api 调用：
	dpm_prepare();
	dpm_suspend();

dpm_suspend_end() 包括如下 api 调用：
	dpm_suspend_late();
	dpm_suspend_noirq();
	
dpm_resume_start() 包括如下 api 调用：
	dpm_resume_noirq();
	dpm_resume_early();

dpm_resume_end() 包括如下 api 调用：
	dpm_resume();
	dpm_complete();
```

### struct dev_pm_info
- 结构体定义：
保存pm状态
```
[include/linux/pm.h]
553 struct dev_pm_info {
554     pm_message_t        power_state;
555     unsigned int        can_wakeup:1;
556     unsigned int        async_suspend:1;
557     bool            in_dpm_list:1;  /* Owned by the PM core */
558     bool            is_prepared:1;  /* Owned by the PM core */
559     bool            is_suspended:1; /* Ditto */
560     bool            is_noirq_suspended:1;
561     bool            is_late_suspended:1;
562     bool            early_init:1;   /* Owned by the PM core */
563     bool            direct_complete:1;  /* Owned by the PM core */
564     spinlock_t      lock;
565 #ifdef CONFIG_PM_SLEEP
566     struct list_head    entry;
567     struct completion   completion;
568     struct wakeup_source    *wakeup;
569     bool            wakeup_path:1;
570     bool            syscore:1;
571     bool            no_pm_callbacks:1;  /* Owned by the PM core */
572 #else
573     unsigned int        should_wakeup:1;
574 #endif
575 #ifdef CONFIG_PM
576     struct timer_list   suspend_timer;
577     unsigned long       timer_expires;
578     struct work_struct  work;
579     wait_queue_head_t   wait_queue;
580     struct wake_irq     *wakeirq;
581     atomic_t        usage_count;
582     atomic_t        child_count;
583     unsigned int        disable_depth:3;
584     unsigned int        idle_notification:1;
585     unsigned int        request_pending:1;
586     unsigned int        deferred_resume:1;
587     unsigned int        runtime_auto:1;
588     bool            ignore_children:1;
589     unsigned int        no_callbacks:1;
590     unsigned int        irq_safe:1;
591     unsigned int        use_autosuspend:1;
592     unsigned int        timer_autosuspends:1;
593     unsigned int        memalloc_noio:1;
594     unsigned int        links_count;
595     enum rpm_request    request;
596     enum rpm_status     runtime_status;
597     int         runtime_error;
598     int         autosuspend_delay;
599     unsigned long       last_busy;
600     unsigned long       active_jiffies;
601     unsigned long       suspended_jiffies;
602     unsigned long       accounting_timestamp;
603 #endif
604     struct pm_subsys_data   *subsys_data;  /* Owned by the subsystem. */
605     void (*set_latency_tolerance)(struct device *, s32);
606     struct dev_pm_qos   *qos;
607 };
```

### /sys/power 用户接口
/sys/power/ 下的接口包括

-- disk
>  hibernate，即 STD 功能，可支持多种模式，包括
```
[kernel/power/hibernate.c]
 906 static const char * const hibernation_modes[] = {
 907     [HIBERNATION_PLATFORM]  = "platform",
 908     [HIBERNATION_SHUTDOWN]  = "shutdown",
 909     [HIBERNATION_REBOOT]    = "reboot",
 910 #ifdef CONFIG_SUSPEND
 911     [HIBERNATION_SUSPEND]   = "suspend",
 912 #endif
 913     [HIBERNATION_TEST_RESUME]   = "test_resume",
 914 };

```
>
show : 打印 hibernation_mode
store : 设定 hibernation_mode

-- mem_sleep 
-- pm_debug_messages  
-- pm_print_times  

-- pm_trace
>
提供电源管理过程的 trace 记录，由 CONFIG_PM_TRACE 定义

-- pm_wakeup_irq 

-- resume
> hibernate 的逆向操作，读取 disk image，恢复系统状态

-- state 
>
show : 显示支持的 status，如 mem，standby，freeze，disk
store : 根据传入的字符串，进入 suspend 或 hibernate 模式

-- pm_test
>
将 pm 分为多个步骤，每个步骤可以插入 pm test code，用于判定状态切换是否成功。
用于测试的 debug level 如下：
```
[kernel/power/main.c]
156 static const char * const pm_tests[__TEST_AFTER_LAST] = {
157     [TEST_NONE] = "none",
158     [TEST_CORE] = "core",
159     [TEST_CPUS] = "processors",
160     [TEST_PLATFORM] = "platform",
161     [TEST_DEVICES] = "devices",
162     [TEST_FREEZER] = "freezer",
163 };
```
>
show : 显示当前 debug level
store : 设定当前 debug level

-- wake_unlock
-- image_size 
-- pm_async 
-- pm_freeze_timeout 
-- pm_test
-- pm_trace_dev_match

-- reserved_size 
>在 freeze，freeze_noirq 操作中保存设备驱动分配的空间，避免 STD 过程中丢失

-- resume_offset 
-- wake_lock 

-- wakeup_count
>
用于控制 sleep 和 wakeup 同步，如正确处理 sleep 过程中发生的 wakeup 操作。

### 其它 pm 接口

-- debugfs/suspend_status
> 以 debugfs 的形式提供 suspend 统计信息，打印格式如下：
```
/ # cat /sys/kernel/debug/suspend_stats 
success: 0
fail: 0
failed_freeze: 0
failed_prepare: 0
failed_suspend: 0
failed_suspend_late: 0
failed_suspend_noirq: 0
failed_resume: 0
failed_resume_early: 0
failed_resume_noirq: 0
failures:
  last_failed_dev:	
			
  last_failed_errno:	0
			0
  last_failed_step:	
```

-- /dev/snaphost
>向用户提供 software 的 STD 操作


