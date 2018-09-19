一个scsi驱动，可以让linux当作scsi的指令的接受机使用。比如虚拟机DVD的，虚拟磁带，iscsi主机等。听说准备被整合进2.6.38版本的内核去了。要写虚拟scsi设备驱动的，可以参考一下实现。 主页是http://stgt.sourceforge.net/   ，支持设备和虚拟设备很多的：
 

Tgt supports various target drivers
iSCSI target driver for Ethernet NICs
iSER target driver for Infiniband and RDMA NICs
Virtual SCSI target driver for IBM pSeries
FCoE target driver for Ethernet NICs (in progress)
Qlogic qla2xxx FC target driver (in progress)
LSI logic FC target driver (not yet)
Qlogic qla4xxx iSCSI target driver (not yet)
Tgt can emulate various device types
SBC: a virtual disk drive that can use a file to store the content.
SMC: a virtual media jukebox that can be controlled by the "mtx" tool (partially functional).
MMC: a virtual DVD drive that can read DVD-ROM iso files and create burnable DVD+R. It can be combined with SMC to provide a fully operational DVD jukebox.
SSC: a virtual tape device (aka VTL) that can use a file to store the content (in progress).
OSD: a virtual object-based storage device that can use a file to store the content.
