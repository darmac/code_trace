### freeze code call flow

```
echo "mem">/sys/power/state
	state_sotre()
		decode_state()
		pm_suspend()
			enter_state()
				suspend_prepare()
				suspend_test(TEST_FREEZER)
				suspend_devices_and_enter()
				suspend_finish()
```

```
suspend_prepare()
	pm_prepare_console()
		pm_vt_switch()
		vt_move_to_console(SUSPEND_CONSOLE, 1)
		vt_kmsg_redirect(SUSPEND_CONSOLE)
	__pm_notifier_call_chain(PM_SUSPEND_PREPARE, -1, &nr_calls) //pm_chain_head
	suspend_freeze_processes()
		freeze_processes()
			__usermodehelper_disable(UMH_FREEZING)
			pm_wakeup_clear(true)
			try_to_freeze_tasks(true)
				while(true)
					for_each_process_thread(g, p)
						freeze_task(p)
							fake_signal_wake_up() //用户进程
								lock_task_sighand()
			oom_killer_disable()
		freeze_kernel_threads()
			try_to_freeze_tasks(false)
				freeze_workqueues_begin()
				while(true)
					for_each_process_thread(g, p)
						freeze_task(p)
							wake_up_state()
								try_to_wake_up()
					freeze_workqueues_busy()

```

suspend_ops定义
```
[kernel/power/suspend.c]
58 static const struct platform_suspend_ops *suspend_ops

[include/linux/suspend.h]
177 struct platform_suspend_ops {
178     int (*valid)(suspend_state_t state);
179     int (*begin)(suspend_state_t state);
180     int (*prepare)(void);
181     int (*prepare_late)(void);
182     int (*enter)(suspend_state_t state);
183     void (*wake)(void);
184     void (*finish)(void);
185     bool (*suspend_again)(void);
186     void (*end)(void);
187     void (*recover)(void);
188 };
```

```
suspend_devices_and_enter()
	platform_suspend_begin(state)
		suspend_ops->begin()
	suspend_console()
		console_lock();
		up_console_sem();
	suspend_enter(state, &wakeup)
		platform_suspend_prepare()
		dpm_suspend_late()
		platform_suspend_prepare_late()
		dpm_suspend_noirq()
		platform_suspend_prepare_noirq()
		disable_nonboot_cpus()
		arch_suspend_disable_irqs()
		syscore_suspend()
		pm_wakeup_pending()
		suspend_ops->enter()
		
		syscore_resume()
		arch_suspend_enable_irqs()
		enable_nonboot_cpus()
		platform_resume_noirq(state)
		dpm_resume_noirq()
		platform_resume_early()
		dpm_resume_early()
		platform_resume_finish()
	
	dpm_resume_end()
		dpm_resume()
		dpm_complete()
	resume_console()
	platform_resume_end()
```

```
suspend_finish()
	suspend_thaw_processes()
		thaw_processes()
			oom_killer_enable()
			__usermodehelper_set_disable_depth(UMH_FREEZING)
			thaw_workqueues()
			for_each_process_thread(g, p)
				__thaw_task(p)
					wake_up_process(p)
			usermodehelper_enable()
			schedule()
	pm_notifier_call_chain(PM_POST_SUSPEND) //notify pm_chain_head
	pm_restore_console()
		vt_move_to_console(orig_fgconsole, 0)
		vt_kmsg_redirect(orig_kmsg)
```