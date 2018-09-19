```text
    




  下载LOFTER我的照片书  |
想到服务器上去用kprobe调试一下结果发现这2.6.9的内核上面，kdb和kprobe是有冲突的。


/* 默认的kdb patch和 kprobe有冲突 要修改这个才能cblade版本里使用 kprobe
一开始以为是linux的crash dump 和kdb注册的panic事件通知有关。
内核的panic注册函数都放在
panic_notifier_list
这个全局链表里面，当内核panic的时候，他就调用这个链表下面的注册函数，linux crash dump 和 kdb都是注册了自己的处理函数的，所以才能在panic的时候得到事件通知，然后作dump安，启动调试那些工作。

所以一开始的都是自己写jprobe模块的时候，使用这样一个函数来把系统注册的panic通知事件全部卸载掉

void notifier_chain_remove(char * dumpfuncionname)
{
    struct notifier_block * nl = panic_notifier_list;
    struct notifier_block * vmdump_block = NULL;
    void *     vmdump_func = (void *) kallsyms_lookup_name(dumpfuncionname);
    if (vmdump_func == NULL)  {
       printk ("can nnot find  %s function\n", dumpfuncionname);
       return ; 
   }

        while(nl !=NULL)
        {        
         printk ("dump funcion=%p\n", nl->notifier_call);
        
         if((void *) nl->notifier_call == vmdump_func )
         {
            vmdump_block = nl;            
         }
                 nl= nl->next;
        }
        
        if (vmdump_block) {
            printk ("find %s notifier block=%p, notifier_chain_unregister\n",dumpfuncionname ,vmdump_block);
            notifier_chain_unregister(&panic_notifier_list, vmdump_block);
        } else  {
            printk ("can nnot find  %s function's notifier block\n", dumpfuncionname);
        }

        return;
}

然后调用
 notifier_chain_remove ("vmdump_do_dump");
 notifier_chain_remove("kdb_panic");
就可以把那两个 函数给反注册了。

但后来发现kprobe其实注册的是另外一种事件 "die_chain", kprobe修改代码加上int3调试指令，代码运行到相应的地方就会触发int3中断，然后系统调用do_int3函数，do_int3函数调用kprobe的注册的die事件通知函数修复int3错误，系统就可以继续往下运行了。

查看一下系统的i386die_chain链表就饿可以知道有哪些程序注册了die通知事件了。

void list_die_chain(void)
{
        struct notifier_block * nl =NULL;
        struct  notifier_block * die_chain = NULL;
        
        die_chain = (struct notifier_block *) kallsyms_lookup_name("i386die_chain");
        if  ( die_chain ==NULL )
        {
               printk ("can nnot find die_chain\n");
               return ; 
        }

    nl = die_chain;
        while(nl !=NULL)
        {        
         printk ("die handler funcion=%p\n", nl->notifier_call);
                 nl= nl->next;
        }


       return ;
}

不过新的内核这个panic和die链表都改成原子性的了。
新的结构遍历  panic_notifier_list->head 就可以了，具体可以参考以下这两个函数的代码：
notifier_chain_register
notifier_chain_unregister

其实linux内核还有很多reboot等事件也是这样实现的，自己要注册事件通知的，用notifier_chain_register往系统注册就可以了。




久版本kdb和kprobe不兼容的原因主要是在这个文件里面的 do_int3  函数上，kdb的patch绕过了kprobe的修复die注册函数了，导致kprobe加了调试点之后没办法恢复，系统进入panic出错的情况。


 ./arch/i386/kernel/traps.c
 
 do_int3 函数中的问题
  
 在2.6.9版本的内核，如果你配置了kdb的话，他把do_int3文件改为
 #ifdef  CONFIG_KDB
asmlinkage void do_int3(struct pt_regs * regs, long error_code)
{
    if (kdb(KDB_REASON_BREAK, error_code, regs))
        return;
    do_trap(3, SIGTRAP, "int3", 1, regs, error_code, NULL);
}
#endif   

 
 其实配置了使用kprobe的内核的do_int3函数应该是这样的
  #if defined(CONFIG_KPROBES) && !defined(CONFIG_KDB)
asmlinkage int do_int3(struct pt_regs *regs, long error_code)
{
    if (notify_die(DIE_INT3, "int3", regs, error_code, 3, SIGTRAP)
            == NOTIFY_STOP)
        return 1;
    // This is an interrupt gate, because kprobes wants interrupts
    //disabled.  Normal trap handlers don't. 
    restore_interrupts(regs);
    do_trap(3, SIGTRAP, "int3", 1, regs, error_code, NULL);
    return 0;
}
#endif
 
 
 
如果把kdb版本do_int3函数编译进去，没办法调用kprobe的die注册函数了，所以kprobe在相应的函数加了int3调试指令之后就没办法恢复了。我看比较新的kdb的patch的代码，这个问题应该是解决的了，就是把do_int3函数改为这样，这样kdb和kprobe就可以同时使用了。
 
asmlinkage int  do_int3(struct pt_regs *regs, long error_code)
{

#ifdef CONFIG_KDB
    if (kdb(KDB_REASON_BREAK, error_code, regs))
        return 1;
#endif

#if defined(CONFIG_KPROBES)
    if (notify_die(DIE_INT3, "int3", regs, error_code, 3, SIGTRAP)
            == NOTIFY_STOP)
        return 1;
#endif

   //This is an interrupt gate, because kprobes wants interrupts
   // disabled.  Normal trap handlers don't. 
    restore_interrupts(regs);
    do_trap(3, SIGTRAP, "int3", 1, regs, error_code, NULL);
    return 0;
}
*/

 

 

修改之后，kprobe和 kdb都能正常工作。

//echo '0' > /proc/sys/kernel/kdb            关闭kdb
//echo '1' > /proc/sys/kernel/kdb            开启kdb
//echo '2' > /proc/sys/kernel/kdb            任何键盘按键都不能触发kdb

//echo 1 >/proc/sys/kernel/sysrq
//echo c >/proc/sysrq-trigger

触发进入kdb没有问题，

发一个2.6.9可用kprobe的例子，还是新版的用法简便一点阿！

#include "qla_def.h"

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>

#include <linux/socket.h>
#include <linux/netdevice.h>

#include <net/ip.h>
#include <linux/skbuff.h>

#include <linux/version.h>
#include <linux/kallsyms.h>
#include <scsi/scsi_cmnd.h>

/*
 * Jumper probe for do_fork.
 * Mirror principle enables access to arguments of the probed routine
 * from the probe handler.
 */

/* Proxy routine having the same arguments as actual do_fork() routine */

static int   jdo_qla2xxx_eh_abort (struct scsi_cmnd *cmd)
{
  static int first =0;
 
    os_lun_t    *q;
    scsi_qla_host_t *ha;
    scsi_qla_host_t *vis_ha;
    srb_t        *sp;
    unsigned int    b, t, l;
        fc_port_t    *fcport;
        os_tgt_t    *tq;
       
if (first < 1 )
   dump_stack();
first ++;


    /* Get the SCSI request ptr */
    sp = (srb_t *) CMD_SP(cmd);

        if (sp == NULL ) {
          goto out;
       }        

        ha = (scsi_qla_host_t *)cmd->device->host->hostdata;
        vis_ha = (scsi_qla_host_t *)cmd->device->host->hostdata;
        
    /* Generate LU queue on bus, target, LUN */
    b = cmd->device->channel;
    t = cmd->device->id;
    l = cmd->device->lun;
    q = GET_LU_Q(vis_ha, t, l);

        if (q == NULL) { goto out;}

        printk ( "scsi(%ld:%d:%d:%d) srb_t timeout =%ld\n" ,ha->host_no, (int)b, (int)t, (int)l, (jiffies - sp->r_start) /HZ);
        
        tq = TGT_Q(vis_ha, t);
        fcport = q->fclun->fcport;
        
        printk ("scsi(%ld:%d:%d:%d),ha->flags.online =%d, fcport->state=%d\n " ,ha->host_no, (int)b, (int)t, (int)l,  ha->flags.online , atomic_read(&fcport->state));
     
 out:

        /* Always end with a call to jprobe_return(). */
        jprobe_return();
        return 0;
}

static int jprobe_qla2x00_queuecommand(struct scsi_cmnd *cmd, void (*fn)(struct scsi_cmnd *))
{
  static int first =0;
  if (first < 1 )
      dump_stack();
  first ++;
 

        /* Always end with a call to jprobe_return(). */
        jprobe_return();
        return 0;

}


static struct jprobe my_jprobe = {
        .entry                  = (kprobe_opcode_t *) jdo_qla2xxx_eh_abort,
        
 #if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,35)     
        .kp = {
                .symbol_name    = "qla2xxx_eh_abort",
      },
       
 #else
        
#endif       
        
};

static struct jprobe jprobe2 = {
        .entry                  = (kprobe_opcode_t *)  jdo_qla2xxx_eh_abort,
        
};

static struct jprobe  jprobe3 = {
        .entry                  = (kprobe_opcode_t *)  jprobe_qla2x00_queuecommand,
        
};




void list_die_chain(void)
{
        struct notifier_block * nl =NULL;
        struct  notifier_block * die_chain = NULL;
        
        die_chain = (struct notifier_block *) kallsyms_lookup_name("i386die_chain");
        if  ( die_chain ==NULL )
        {
               printk ("can nnot find die_chain\n");
               return ;
        }

    nl = die_chain;
        while(nl !=NULL)
        {        
         printk ("die handler funcion=%p\n", nl->notifier_call);
                 nl= nl->next;
        }


       return ;
}


void notifier_chain_remove(char * dumpfuncionname)
{
    struct notifier_block * nl = panic_notifier_list;
    struct notifier_block * vmdump_block = NULL;
    void *     vmdump_func = (void *) kallsyms_lookup_name(dumpfuncionname);
    if (vmdump_func == NULL)  {
       printk ("can nnot find  %s function\n", dumpfuncionname);
       return ;
   }

        while(nl !=NULL)
        {        
         printk ("dump funcion=%p\n", nl->notifier_call);
        
         if((void *) nl->notifier_call == vmdump_func )
         {
            vmdump_block = nl;            
         }
                 nl= nl->next;
        }
        
        if (vmdump_block) {
            printk ("find %s notifier block=%p, notifier_chain_unregister\n",dumpfuncionname ,vmdump_block);
            notifier_chain_unregister(&panic_notifier_list, vmdump_block);
        } else  {
            printk ("can nnot find  %s function's notifier block\n", dumpfuncionname);
        }

        return;
}
 




static int __init jprobe_init(void)
{
int ret = 0;

list_die_chain();
notifier_chain_remove ("vmdump_do_dump");
//notifier_chain_remove("kdb_panic");

#if LINUX_VERSION_CODE >= KERNEL_VERSION(2,6,35)

#else
    my_jprobe.kp.addr = (kprobe_opcode_t *) kallsyms_lookup_name("qla2xxx_eh_abort");
    if (my_jprobe.kp.addr == NULL) {
        printk("kallsyms_lookup_name could not find address for the specified symbol name\n");
        return 1;
    }
    
    jprobe2.kp.addr = (kprobe_opcode_t *) kallsyms_lookup_name("qla2xxx_eh_device_reset");
    if (jprobe2.kp.addr == NULL) {
        printk("kallsyms_lookup_name could not find address for the specified symbol name\n");
        return 1;
    }
    
        jprobe3.kp.addr = (kprobe_opcode_t *) kallsyms_lookup_name("qla2x00_queuecommand");
    if (jprobe3.kp.addr == NULL) {
        printk("kallsyms_lookup_name could not find address for the specified symbol name\n");
        return 1;
    }
    
    
#endif

        ret = register_jprobe(&my_jprobe);
        if (ret < 0) {
                printk(KERN_INFO "register_jprobe qla2xxx_eh_abort failed, returned %d\n", ret);
                return -1;
        }
        
        ret = register_jprobe(&jprobe2);
        if (ret < 0) {
                unregister_jprobe(&my_jprobe);
                printk(KERN_INFO "register_jprobe jprobe_qla2xxx_eh_device_reset failed, returned %d\n", ret);
                return -1;
        }
        
                
        ret = register_jprobe(&jprobe3);
        if (ret < 0) {
                unregister_jprobe(&my_jprobe);
                unregister_jprobe(&jprobe2);
                printk(KERN_INFO "register_jprobe  qla2x00_queuecommand failed, returned %d\n", ret);
                return -1;
        }
        
        
        printk(KERN_INFO "Planted jprobe at %p, handler addr %p\n",
        my_jprobe.kp.addr, my_jprobe.entry);
        
        
        return 0;
}

static void __exit jprobe_exit(void)
{
        unregister_jprobe(&my_jprobe);
        unregister_jprobe(&jprobe2);
        unregister_jprobe(&jprobe3);
         
        printk(KERN_INFO "jprobe at %p unregistered\n", my_jprobe.kp.addr);
}

module_init(jprobe_init)
module_exit(jprobe_exit)
MODULE_LICENSE("GPL");
```
