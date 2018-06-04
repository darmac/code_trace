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

### API outline
```
SYSCALL_DEFINE4(reboot, int, magic1, int, magic2, unsigned int, cmd,void __user *, arg)
```
即
```
sys_reboot(int magic1, int magic2, unsigned int cmd, void __user* arg)
	kernel_restart()
		kernel_restart_prepare()
			blocking_notifier_call_chain(&reboot_notifier_list, SYS_RESTART, cmd);
			system_state = SYSTEM_RESTART;
			usermodehelper_disable();
			device_shutdown() : 遍历devices_kset
				list_del_init(&dev->kobj.entry);
				pm_runtime_get_noresume(dev);
				dev->class->shutdown_pre(dev);
				dev->bus->shutdown(dev) or dev->driver->shutdown(dev);
		migrate_to_reboot_cpu()
		syscore_shutdown()
			从后往前依次 call syscore_ops_list : ops->shutdown()
		kmsg_dump()
			依次遍历 call dump_list : dumper->dump()
		machine_restart()
			local_irq_disable();
			smp_send_stop();
			arm_pm_restart(); or do_kernel_restart(cmd);
```
