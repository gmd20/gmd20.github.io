# 发现有的虚拟机系统不能接收“正常关机”的菜单命令，就来找一下资料

# 用户按下物理按键，会发送一个ACPI按键信号, 内核里面的acpi驱动这个event.c进行处理的。
https://elixir.bootlin.com/linux/latest/source/drivers/acpi/event.c#L148
```c

static struct genl_family acpi_event_genl_family __ro_after_init = {
	.module = THIS_MODULE,
	.name = ACPI_GENL_FAMILY_NAME,
	.version = ACPI_GENL_VERSION,
	.maxattr = ACPI_GENL_ATTR_MAX,
	.mcgrps = acpi_event_mcgrps,
	.n_mcgrps = ARRAY_SIZE(acpi_event_mcgrps),
};

int acpi_bus_generate_netlink_event(const char *device_class,
				      const char *bus_id,
				      u8 type, int data)
{
	struct sk_buff *skb;
	struct nlattr *attr;
	struct acpi_genl_event *event;
	void *msg_header;
	int size;

	/* allocate memory */
	size = nla_total_size(sizeof(struct acpi_genl_event)) +
	    nla_total_size(0);

	skb = genlmsg_new(size, GFP_ATOMIC);
	if (!skb)
		return -ENOMEM;

	/* add the genetlink message header */
	msg_header = genlmsg_put(skb, 0, acpi_event_seqnum++,
				 &acpi_event_genl_family, 0,
				 ACPI_GENL_CMD_EVENT);
	if (!msg_header) {
		nlmsg_free(skb);
		return -ENOMEM;
	}

	/* fill the data */
	attr =
	    nla_reserve(skb, ACPI_GENL_ATTR_EVENT,
			sizeof(struct acpi_genl_event));
	if (!attr) {
		nlmsg_free(skb);
		return -EINVAL;
	}

	event = nla_data(attr);
	memset(event, 0, sizeof(struct acpi_genl_event));

	strscpy(event->device_class, device_class, sizeof(event->device_class));
	strscpy(event->bus_id, bus_id, sizeof(event->bus_id));
	event->type = type;
	event->data = data;

	/* send multicast genetlink message */
	genlmsg_end(skb, msg_header);

	genlmsg_multicast(&acpi_event_genl_family, skb, 0, 0, GFP_ATOMIC);
	return 0;
}

EXPORT_SYMBOL(acpi_bus_generate_netlink_event);

static int __init acpi_event_genetlink_init(void)
{
	return genl_register_family(&acpi_event_genl_family);
}
```


# acpid，以前这个内核驱动出了netlink，还支持procfs接口的事件通知的
参考acpid和busybox源码的处理。    
https://linux.die.net/man/8/acpid   
https://git.busybox.net/busybox/tree/util-linux/acpid.c    
但新版本的/proc接口被废弃了，基本所有内核事件都要换成netlink了吧，我看了一下centos 8是不支持procfs的acpi event了。

应用都换成netlink接口了，比如
https://sourceforge.net/projects/acpid2/     
```c
/* initialize the ACPI IDs */
static void
acpi_ids_init(void)
{
	genl_get_ids(ACPI_EVENT_FAMILY_NAME);  // netlink  acpi_event 

	initialized = 1;
}
```

# systemd是在logind里面进行处理的
https://github.com/systemd/systemd/blob/master/src/login/logind-button.c
```c
static int button_dispatch(sd_event_source *s, int fd, uint32_t revents, void *userdata) {
        Button *b = userdata;
        struct input_event ev;
        ssize_t l;

        assert(s);
        assert(fd == b->fd);
        assert(b);

        l = read(b->fd, &ev, sizeof(ev));
        if (l < 0)
                return errno != EAGAIN ? -errno : 0;
        if ((size_t) l < sizeof(ev))
                return -EIO;

        if (ev.type == EV_KEY && ev.value > 0) {

                switch (ev.code) {

                case KEY_POWER:
                case KEY_POWER2:
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("Power key pressed."),
                                   "MESSAGE_ID=" SD_MESSAGE_POWER_KEY_STR);

                        manager_handle_action(b->manager, INHIBIT_HANDLE_POWER_KEY, b->manager->handle_power_key, b->manager->power_key_ignore_inhibited, true);
                        break;

                /* The kernel is a bit confused here:
                   KEY_SLEEP   = suspend-to-ram, which everybody else calls "suspend"
                   KEY_SUSPEND = suspend-to-disk, which everybody else calls "hibernate"
                */

                case KEY_SLEEP:
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("Suspend key pressed."),
                                   "MESSAGE_ID=" SD_MESSAGE_SUSPEND_KEY_STR);

                        manager_handle_action(b->manager, INHIBIT_HANDLE_SUSPEND_KEY, b->manager->handle_suspend_key, b->manager->suspend_key_ignore_inhibited, true);
                        break;

                case KEY_SUSPEND:
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("Hibernate key pressed."),
                                   "MESSAGE_ID=" SD_MESSAGE_HIBERNATE_KEY_STR);

                        manager_handle_action(b->manager, INHIBIT_HANDLE_HIBERNATE_KEY, b->manager->handle_hibernate_key, b->manager->hibernate_key_ignore_inhibited, true);
                        break;
                }

        } else if (ev.type == EV_SW && ev.value > 0) {

                if (ev.code == SW_LID) {
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("Lid closed."),
                                   "MESSAGE_ID=" SD_MESSAGE_LID_CLOSED_STR);

                        b->lid_closed = true;
                        button_lid_switch_handle_action(b->manager, true);
                        button_install_check_event_source(b);

                } else if (ev.code == SW_DOCK) {
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("System docked."),
                                   "MESSAGE_ID=" SD_MESSAGE_SYSTEM_DOCKED_STR);

                        b->docked = true;
                }

        } else if (ev.type == EV_SW && ev.value == 0) {

                if (ev.code == SW_LID) {
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("Lid opened."),
                                   "MESSAGE_ID=" SD_MESSAGE_LID_OPENED_STR);

                        b->lid_closed = false;
                        b->check_event_source = sd_event_source_unref(b->check_event_source);

                } else if (ev.code == SW_DOCK) {
                        log_struct(LOG_INFO,
                                   LOG_MESSAGE("System undocked."),
                                   "MESSAGE_ID=" SD_MESSAGE_SYSTEM_UNDOCKED_STR);

                        b->docked = false;
                }
        }

        return 0;
}
```
好像systemd是读取/dev/input的按键事件，不是用netlink acpi event ？
https://github.com/systemd/systemd/tree/master/src/libsystemd/sd-netlink


# systemd 的配置文件
```text
[root@localhost]# cat /etc/systemd/logind.conf 
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See logind.conf(5) for details.

[Login]
#NAutoVTs=6
#ReserveVT=6
#KillUserProcesses=no
#KillOnlyUsers=
#KillExcludeUsers=root
#InhibitDelayMaxSec=5
#HandlePowerKey=poweroff
#HandleSuspendKey=suspend
#HandleHibernateKey=hibernate
#HandleLidSwitch=suspend
#HandleLidSwitchExternalPower=suspend
#HandleLidSwitchDocked=ignore
#PowerKeyIgnoreInhibited=no
#SuspendKeyIgnoreInhibited=no
#HibernateKeyIgnoreInhibited=no
#LidSwitchIgnoreInhibited=yes
#HoldoffTimeoutSec=30s
#IdleAction=ignore
#IdleActionSec=30min
#RuntimeDirectorySize=10%



[root@localhost]# cat /usr/lib/systemd/system/poweroff.target
#  SPDX-License-Identifier: LGPL-2.1+
#
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Power-Off
Documentation=man:systemd.special(7)
DefaultDependencies=no
Requires=systemd-poweroff.service
After=systemd-poweroff.service
AllowIsolate=yes
JobTimeoutSec=30min
JobTimeoutAction=poweroff-force

[Install]

#RemoveIPC=no
#InhibitorsMax=8192
#SessionsMax=8192
```
