# ALSA ASoC Driver

本文档用于讲解ALSA中ASOC部分的程序实现



## 序

​	在DSP开发中，大多数情况的DSP并不使用任何操作系统，即No-OS的应用居多，尤其以安全性能需求最为突出的汽车应用，更是如此。大多数的汽车应用，更倾向于使用MCU+DSP的架构。但随着车机科技的进步，车机的计算能力正在超越其他ECU部件飞速发展。在可预见的几年内，车机的APP开发将是汽车行业的新的增长点。许多非安全功能出于成本的考虑，将向车机集成。

​	在此大背景下，对于声学从业人员，掌握车机Linux音频架构是非常有必要的。在国内，车机基于Android开发将是未来主流，而Android的底层的Kernel部分为Linux，音频相关的驱动也是基于Linux开发。学习Linux ALSA Driver的重要性也就呼之欲出了。

​	大多数情况我们不需要学习ALSA driver的开发，因为通常芯片厂商已经提供了成套的源代码供使用，只需要使用`ALSA-lib`中的API即可。但学习ALSA driver可以更好的让我们理解Linux ALSA的架构。



## Platform Device & Driver

​	在介绍ALSA 驱动之前，*Platform Device* 和 *Platform Driver*的注册过程需要详细了解。声卡驱动注册和初始化的过程中，基本都是基于Platform Device。这与**《Linux设备驱动开发》**一书中简单的char device不同，自2.6内核后，Linux驱动程序中大量的使用了`platform_device`、`platform_ driver` 和 `device`、 `bus`、 `driver`的方式对设备和驱动进行管理。`platform device`主要针对soc设备等带有外设的场景，由于许多SoC带有音频外设，所以许多设备，尤其是移动设备的ALSA中大量使用了`platform device`和 `platform driver`，我们以最常用的S3C6410开发板中的platform_LED为例，介绍Platform device driver的注册和配对过程，下面的顺序图分别从`platform device` 和 `platform driver`的角度，列出了其从创建和匹配到`probe`的过程。

```mermaid
sequenceDiagram
	participant mdev as my_device
	participant lplat	as platform.c
	participant lcore as core.c
	participant lbus as bus.c
	
	note left of mdev:arch/arm/mach-s3c24xx/mach-smdk2410.c
	
	mdev->>lplat:platform_add_devices
	 
	lplat->>lplat:platform_device_register
	lplat->>lcore:device_initialize
 	
 	lplat->>lplat:arch_setup_pdev_archdata
	lplat->>lplat:platform_device_add
	lplat->>lcore:device_add
 
	lcore->>lcore:device_private_init
	lcore-->>lcore:add node symbol etc.
	lcore->>lbus:bus_add_device
	
	lbus-->>lbus:init etc
  
  note over lbus:bus_probe_device
	
	
	
```

```mermaid
sequenceDiagram
	participant mdri as my_driver
	participant lplat	as platform.c
	participant ldriver as driver.c
	participant lbus as bus.c
	
	note left of mdri:drivers/leds/led-s3c24xx.c
	
	mdri->>lplat:module_platform_driver	

	lplat->>lplat:	__platform_driver_register
	lplat->>lplat:driver_register
	lplat->>ldriver:driver_find
  
	ldriver->>lbus:bus_add_driver
	
	lbus-->>lbus:init etc
	
	note over lbus:driver_attach/driver_probe_device
	

	

```



​	从上述的注册过程，在各个device或driver挂载到对应的`bus`上之后，便会调用对应的`probe`函数。这里需要展开讨论两个小点：

- bus

- probe

  

### bus

​	通常的设备，都是通过总线挂载到MCU上，如`I2C`、`SPI`、`PCI`等，对于我们关心声卡设备，很多Codec设备也是通过I2C或SPI总线挂在MCU上。

- 这里的挂载，主要指的是控制总线挂载在上述的总线上。数据通常不是通过上述总线控制的

为了使得代码有更好的内聚性和模块适用性，对于哪些本身具有一些功能的，以音频为例，本身就能输出模拟音频或数字音频的，不需要经过上述的总线对输出进行控制。有鉴于此，Linux采用了一种`Platform bus`的方式对配置等过程进行了抽象分隔，使得原来的`device`和`driver`的程序逻辑仍然能够保持一致。下图列出了`bus_type`以及`platform_bus_type`的数据结构，可见`platform_bus_type`只是`name = "platform"`的总线结构。

```mermaid
 classDiagram
class BusType
	BusType: const char		*name
	BusType: const char		*dev_name
	BusType: struct device		*dev_root
	BusType: struct device_attribute	*dev_zttrs
	BusType: const struct attribute_group **bus_groups
	BusType: const struct attribute_group **dev_groups
	BusType: const struct attribute_group **drv_groups

	BusType: int (*match)(struct device *dev, struct device_driver *drv)
	BusType: int (*uevent)(struct device *dev, struct kobj_uevent_env *env)
	BusType: int (*probe)(struct device *dev)
	BusType: int (*remove)(struct device *dev)
	BusType: void (*shutdown)(struct device *dev)

	BusType: int (*online)(struct device *dev)
	BusType: int (*offline)(struct device *dev)

	BusType: int (*suspend)(struct device *dev, pm_message_t state)
	BusType: int (*resume)(struct device *dev)

	BusType: const struct dev_pm_ops *pm

	BusType: const struct iommu_ops *iommu_ops

	BusType: struct subsys_private *p
	BusType: struct lock_class_key lock_key
	
class PlatformBusType
	PlatformBusType: .name		= "platform"
	PlatformBusType: .dev_groups	= platform_dev_groups
	PlatformBusType: .match		= platform_match
	PlatformBusType: .uevent		= platform_uevent
	PlatformBusType: .pm		= &platform_dev_pm_ops
	
	PlatformBusType..>BusType: Instance of

```

### probe

probe函数的调用在一开始的流程图中，有所显示。以driver为例，其attach 函数如下

```c
static int __driver_attach(struct device *dev, void *data)
{
	struct device_driver *drv = data;

	/*
	 * Lock device and try to bind to it. We drop the error
	 * here and always return 0, because we need to keep trying
	 * to bind to devices and some drivers will return an error
	 * simply if it didn't support the device.
	 *
	 * driver_probe_device() will spit a warning if there
	 * is an error.
	 */

	if (!driver_match_device(drv, dev))
		return 0;

	if (dev->parent)	/* Needed for USB */
		device_lock(dev->parent);
	device_lock(dev);
	if (!dev->driver)
		driver_probe_device(drv, dev);
	device_unlock(dev);
	if (dev->parent)
		device_unlock(dev->parent);

	return 0;
}
```

对于platform device match函数有一系列的match方法，而name的match仅仅是最后一个通用选项。

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* When driver_override is set, only bind to the matching driver */
	if (pdev->driver_override)
		return !strcmp(pdev->driver_override, drv->name);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try ACPI style match */
	if (acpi_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```

​	不管是device还是driver在attach之前，都会先调用函数将drv或dev挂载到bus上，这样就回答了，在device初始化时，没有任何driver挂载在bus上。对于driver，在`bus_add_driver`函数中：

```c
	klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers)
  if (drv->bus->p->drivers_autoprobe) {
	error = driver_attach(drv);
	if (error)
		goto out_unregister;
	}
```

​	当probe完成后，device和driver的配对就完成了，用户可以通过driver的方法去实现相应的功能。	

​	下面列举了，platform_device 和 platform driver的数据结构

```mermaid
classDiagram

class PlatformDevice
      PlatformDevice : const char* name
      PlatformDevice : int id
      PlatformDevice : bool	id_auto
      PlatformDevice : struct device	dev
      PlatformDevice : u32		num_resources
      PlatformDevice : struct resource	*resource
      PlatformDevice : const struct platform_device_id	*id_entry
      PlatformDevice : char *driver_override
      PlatformDevice : struct mfd_cell *mfd_cell
      PlatformDevice : struct pdev_archdata	archdata
      
class Device
      Device: struct device		*parent
      Device: struct device_private	*p
      Device: struct kobject kobj
      Device: const char		*init_name
      Device: const struct device_type *type
      Device: struct mutex		mutex
      Device: struct device_driver *driver
      Device: void		*platform_data
      Device: void		*driver_data
      Device: struct dev_pm_info	power
      Device: struct dev_pm_domain	*pm_domain
      Device: struct dev_pin_info	*pins
      Device: struct int		numa_node
      Device: struct dev_pm_domain	*pm_domain
      Device: u64		*dma_mask
      Device: u64		coherent_dma_mask
      Device: struct unsigned long	dma_pfn_offset
      Device: struct  device_dma_parameters *dma_parms
      Device: struct list_head	dma_pools
      Device: struct dma_coherent_mem	*dma_mem
      Device: struct cma *cma_area
      Device: truct dev_archdata	archdata
      Device: struct device_node	*of_node
      Device: struct acpi_dev_node	acpi_node
      Device:	dev_t			devt
      Device:	u32			id
      Device:	spinlock_t		devres_lock
      Device:	struct list_head	devres_head
      Device:	dev_t			devt
      Device:	struct klist_node	knode_class
      Device:	struct class		*class
      Device:	const struct attribute_group **groups
      Device:	void	(*release)(struct device *dev)
      Device:	truct iommu_group	*iommu_group
      Device:	bool offline_disabled
      Device: bool offline
      
class Resource
			Resource: resource_size_t start
			Resource: resource_size_t end
			Resource: const char *name
			Resource: unsigned long flags
			Resource: struct resource *parent, *sibling, *child
      
class PlatformDeviceId
			PlatformDeviceId: char name[PLATFORM_NAME_SIZE]
			PlatformDeviceId: kernel_ulong_t driver_data
			
class MdfCell
			MdfCell: const char		*name
			MdfCell: int			id
			MdfCell: atomic_t		*usage_count
			MdfCell: int (*enable)(struct platform_device *dev)
			MdfCell: int (*disable)(struct platform_device *dev)
			MdfCell: int (*suspend)(struct platform_device *dev)
			MdfCell: int (*resume)(struct platform_device *dev)
			MdfCell: void *platform_data
			MdfCell: size_t	pdata_size
			MdfCell: const char		*of_compatible
			MdfCell: const char		*acpi_pnpid
			MdfCell: int num_resources
			MdfCell: const struct resource	*resources
			MdfCell: bool ignore_resource_conflicts
			MdfCell: bool			pm_runtime_no_callbacks
			MdfCell: const char * const	*parent_supplies
			MdfCell: int num_parent_supplies
			
class PdevArchdata
PdevArchdata:struct omap_device *od


class OmapDevice
		OmapDevice: struct platform_device		*pdev
    OmapDevice: struct omap_hwmod		**hwmods
    OmapDevice: unsigned long			_driver_status
    OmapDevice: u8 hwmods_cnt
    OmapDevice: u8 _state
    OmapDevice: u8 flags
    


 

			
			
			
			PlatformDevice <.. Device : struct device
      PlatformDevice <.. Resource: struct resource
      PlatformDevice <.. PlatformDeviceId: struct platform_device_id
      PlatformDevice <.. MdfCell: struct mfd_cell
      PlatformDevice <.. PdevArchdata: struct pdev_archdata
      MdfCell <.. Resource : struct resource
			OmapDevice ..> PdevArchdata
			
      
```

```mermaid
classDiagram
 
class PlatformDriver
	PlatformDriver : int (*remove)(struct platform_device *)
	PlatformDriver : void (*shutdown)(struct platform_device *)
	PlatformDriver : int (*suspend)(struct platform_device *, pm_message_t state)
	PlatformDriver : int (*resume)(struct platform_device *)
	PlatformDriver : struct device_driver driver
	PlatformDriver : const struct platform_device_id *id_table
	PlatformDriver : bool prevent_deferred_probe
	
class DeviceDriver
	DeviceDriver: const char		*name
	DeviceDriver: struct bus_type		*bus
	DeviceDriver: struct module		*owner
	DeviceDriver: const char		*mod_name
	DeviceDriver: bool suppress_bind_attrs
	DeviceDriver: const struct of_device_id	*of_match_table
	DeviceDriver: const struct acpi_device_id	*acpi_match_table
	DeviceDriver: int (*probe) (struct device *dev)
	DeviceDriver: int (*remove) (struct device *dev)
	DeviceDriver: void (*shutdown) (struct device *dev)
	DeviceDriver: int (*suspend) (struct device *dev, pm_message_t state)
	DeviceDriver: int (*resume) (struct device *dev)
	DeviceDriver: const struct attribute_group **groups
	DeviceDriver: const struct dev_pm_ops *pm
	DeviceDriver: struct driver_private *p
	
class BusType
BusType: 	const char		*dev_name
BusType: 	struct device		*dev_root
BusType: 	struct device_attribute	*dev_attrs
BusType: 	const struct attribute_group **bus_groups
BusType: 	const struct attribute_group **dev_groups
BusType: 	const struct attribute_group **drv_groups

BusType: 	int (*match)(struct device *dev, struct device_driver *drv)
BusType: 	int (*uevent)(struct device *dev, struct kobj_uevent_env *env)
BusType: 	int (*probe)(struct device *dev)
BusType: 	int (*remove)(struct device *dev)
BusType: 	void (*shutdown)(struct device *dev)

BusType: 	int (*online)(struct device *dev)
BusType: 	int (*offline)(struct device *dev)

BusType: 	int (*suspend)(struct device *dev, pm_message_t state)
BusType: 	int (*resume)(struct device *dev)

BusType: 	const struct dev_pm_ops *pm

BusType: 	const struct iommu_ops *iommu_ops

BusType: 	struct subsys_private *p
BusType: 	struct lock_class_key lock_key

	
class PlatformDeviceID
	PlatformDeviceID: char name[PLATFORM_NAME_SIZE]
	PlatformDeviceID: kernel_ulong_t driver_data
	
	
	
	PlatformDriver<..DeviceDriver: struct device_driver 
	PlatformDriver<..PlatformDeviceID: struct platform_device_id
	BusType..>DeviceDriver:struct bus_type
```

​	从上述的数据结构可以看到，对应platform相关的resource相关代码，全部涵盖在device中，而driver，依赖probe函数，完成与device的配对后，即可通过方法，控制deivce中的寄存器等资源。



## ALSA



### ALSA驱动层



#### generic ALSA driver

​	ALSA驱动的部分，可以参照Linux [官方的文档](https://www.kernel.org/doc/html/v4.15/sound/kernel-api/writing-an-alsa-driver.html)。以经常使用的PCM设备为例，声卡驱动创建主要的流程均可以用下面的流程图来表示。

​	通常由于音频都是经过解码后的pcm裸流混音后输出的，即多个`pcm_substream`合成一个`pcm_stream`。所以通常来说对具体音频流数据的操作函数，都在`pcm_substream`中定义。

```mermaid
sequenceDiagram
	participant card as sound card
	participant chip as my chip sepcific
	participant bus as bus specific
  participant pcm as pcm device
  participant sub as pcm substream
  activate card
	note over card: probe (bus_dev\platform_dev)
	card-->card:Create a card instance(snd_card_new)
	card->>chip:Cread a main component
	activate chip
	chip-->chip:fill device data (snd_device_ops)
	chip-->bus:initialzie bus entry
	activate bus
	bus-->bus: bus resource allocation
	note over bus,chip: bus pass bus_device to chip
	deactivate bus
	chip-->chip:register device to sound card(snd_device_new)
	deactivate chip
	card-->card:Set driver ID and name Strings
	card->>pcm:create pcm device (snd_pcm_new (*card))
	activate pcm
	pcm-->pcm:set playback and caputure ops 
	note over pcm: snd_pcm_ops
	note over pcm,sub: op {.open .close .hwparam etc}
	note over pcm: pcm_runtime {all the alsa related stuff}
	note over pcm: snd_pcm_hardware {hardware settings}
	note over pcm,card:pcm pass pcm_device to card
	deactivate pcm
	card-->card:register other devices...
	card-->card:register sound card
	card-->card:set bus driver data
	deactivate card
```

#### ASoC ALSA Driver

​	与挂载在外设总线的音频设备不同，由于外设音频上的音频设备很多进行了ASIC封装，所以可以省去很多代码；而SoC（或者原先挂载在外设总线上的SoC），则需要对其中的音频接口`DAI`、音频接口数据`DMA`进行说明。另外由于SoC通用性，与其相连接的Codec芯片，其配置代码也可以进行一般化。所以在ASoC的ALSA驱动代码中，相较普通的ALSA驱动多了如下几个重要部分：

- DAI外设口设置如（SPort与SAI）

- DMA（DAI）

- Codec外设总线（I2C、SPI）
  - 由于Codec内部有很多控制Codec行为的寄存器，ALSA中讲一组特定的作用与Codec的总线指令，抽象为`DAPM`

  ~~ASoC 的描述~~
  
  ```mermaid
  sequenceDiagram
  participant machine as ASoC Machine
  participant codec as ASoC Codec
  participant pcm as ASoC PCM
  participant dai as ASoC DAI
  
  note over machine:Card define {.dai_link}
  note over machine: dailink{.codec_name .platform_name .init .ops}
  
  note over codec:register bus
  note over codec:register bus_driver
  
  note over pcm:define snd_pcm_ops
  note over pcm:register platform device (snd_soc_platform_driver)
  
  machine-->machine:register sound soc card (platform device)
  machine-->machine:init card lists:card->codec_dev_list,widgets,paths,dapm
  machine-->machine:alloc dai_link spaces
  machine-->machine:instantiate card,bind dai links
  
  codec->>machine:bind codec for snd_soc_pcm_runtime
  pcm->>machine:bind dai(platform) snd_soc_pcm_runtime
  
  machine-->machine:probe codec,platform 
  machine->>pcm:add new pcm
  
  
  
  
  
  
  ```
  
  

## ALSA-ASOC

​	ALSA是Advanced Linux Sound Architecture的缩写，高级[Linux](https://baike.baidu.com/item/Linux/27050)声音架构的简称,它在Linux操作系统上提供了音频和MIDI（Musical Instrument Digital Interface，音乐设备数字化接口）的支持。

​	而对于嵌入式系统来说，犹如`platform_device` 一般，Linux将一般设备的snd_card结构进行封装，并且将实现进行抽象，将`asoc`分成`mach`、`platform`、`codec`三层。其中`platform`和`codec`层不包含策略，具体的匹配通常都在`mach`层中确定。

- mach通常代指某款确定的板子
- platform通常代指某款MCU或DSP，主要由三部分组成
  - DMA驱动
  - DAI驱动
  - DSP驱动
- codec为MCU或DSP的外设解码器，主要包括：
  - DAI、PCM设置
  - codec 控制 （利用RegMap API）
  - 混音器与音频控制
  - DAPM描述信息
  - DAPM事件控制器

下图为snd_soc_card的数据结构，除了`snd_card`外，还包含`snd_soc_dai_link`和`snd_soc_pcm_runtime`两个重要的数据结构。

```mermaid
classDiagram

class SndSocCard
	SndSocCard: const char *long_name
	SndSocCard: const char *driver_name
	SndSocCard: struct device *dev
	SndSocCard: struct snd_card *snd_card
	SndSocCard: struct module *owner

	SndSocCard: struct mutex mutex
	SndSocCard: struct mutex dapm_mutex

	SndSocCard: bool instantiated

	SndSocCard: int (*probe)(struct snd_soc_card *card)
	SndSocCard: int (*late_probe)(struct snd_soc_card *card)
	SndSocCard: int (*remove)(struct snd_soc_card *card)

	SndSocCard: int (*suspend_pre)(struct snd_soc_card *card)
	SndSocCard: int (*suspend_post)(struct snd_soc_card *card)
	SndSocCard: int (*resume_pre)(struct snd_soc_card *card)
	SndSocCard: int (*resume_post)(struct snd_soc_card *card)

	SndSocCard: int (*set_bias_level)(struct snd_soc_card *,struct snd_soc_dapm_context *dapm,enum snd_soc_bias_level level)
	SndSocCard: int (*set_bias_level_post)(struct snd_soc_card *,struct snd_soc_dapm_context *dapm,enum snd_soc_bias_level level)

	SndSocCard: long pmdown_time

	SndSocCard: struct snd_soc_dai_link *dai_link
	SndSocCard: int num_links
	SndSocCard: struct snd_soc_pcm_runtime *rtd
	SndSocCard: int num_rtd

	SndSocCard: struct snd_soc_codec_conf *codec_conf
	SndSocCard: int num_configs

	SndSocCard: struct snd_soc_aux_dev *aux_dev
	SndSocCard: 	int num_aux_devs
	SndSocCard: 	struct snd_soc_pcm_runtime *rtd_aux
	SndSocCard: 	int num_aux_rtd

	SndSocCard: 	const struct snd_kcontrol_new *controls
	SndSocCard: 	int num_controls
	SndSocCard: 	const struct snd_soc_dapm_widget *dapm_widgets
	SndSocCard: 	int num_dapm_widgets
	SndSocCard: 	const struct snd_soc_dapm_route *dapm_routes
	SndSocCard: 	int num_dapm_routes
	SndSocCard: 	bool fully_routed

	SndSocCard: 	struct work_struct deferred_resume_work

	SndSocCard: 	struct list_head codec_dev_list

	SndSocCard: 	struct list_head widgets
	SndSocCard: 	struct list_head paths
	SndSocCard: 	struct list_head dapm_list
	SndSocCard: 	struct list_head dapm_dirty

	SndSocCard: 	struct snd_soc_dapm_context dapm
	SndSocCard: 	struct snd_soc_dapm_stats dapm_stats
	SndSocCard: 	struct snd_soc_dapm_update *update

	SndSocCard: 	struct dentry *debugfs_card_root
	SndSocCard: 	struct dentry *debugfs_pop_time
	SndSocCard: 	u32 pop_time
	SndSocCard: 	void *drvdata
	
class SndCard
	SndCard: int number
	SndCard: char id[16]
	SndCard: char driver[16]
	SndCard: char shortname[32]
	SndCard: char longname[80]
	SndCard: char mixername[80]
	SndCard: char components[128]
	SndCard: struct module *module

	SndCard: void *private_data
	SndCard: void (*private_free) (struct snd_card *card)
	SndCard: struct list_head devices

	SndCard: struct device ctl_dev
	SndCard: unsigned int last_numid
	SndCard: struct rw_semaphore controls_rwsem
	SndCard: rwlock_t ctl_files_rwlock
	SndCard: int controls_count
	SndCard: int user_ctl_count
	SndCard: struct list_head controls
	SndCard: struct list_head ctl_files
	SndCard: struct mutex user_ctl_lock

	SndCard: struct snd_info_entry *proc_root
	SndCard: struct snd_info_entry *proc_id
	SndCard: struct proc_dir_entry *proc_root_link

	SndCard: struct list_head files_list
	SndCard: struct snd_shutdown_f_ops *s_f_ops
	SndCard: spinlock_t files_lock
	SndCard: int shutdown
	SndCard: struct completion *release_completion
	SndCard: struct device *dev
	SndCard: 	struct device card_dev
	SndCard: 	const struct attribute_group *dev_groups[4]
	SndCard: bool registered

	SndCard: unsigned int power_state
	SndCard: struct mutex power_lock
	SndCard: wait_queue_head_t power_sleep

	SndCard: struct snd_mixer_oss *mixer_oss
	SndCard: int mixer_oss_change_count
	
class SndSocDaiLink
	SndSocDaiLink: const char *name
	SndSocDaiLink: const char *stream_name

	SndSocDaiLink: const char *cpu_name
	SndSocDaiLink: struct device_node *cpu_of_node
	SndSocDaiLink: const char *cpu_dai_name

	SndSocDaiLink: const char *codec_name
	SndSocDaiLink: struct device_node *codec_of_node
	SndSocDaiLink: const char *codec_dai_name

	SndSocDaiLink: struct snd_soc_dai_link_component *codecs
	SndSocDaiLink: unsigned int num_codecs
	SndSocDaiLink: const char *platform_name
	SndSocDaiLink: struct device_node *platform_of_node
	SndSocDaiLink: int be_id

	SndSocDaiLink: const struct snd_soc_pcm_stream *params

	SndSocDaiLink: unsigned int dai_fmt

	SndSocDaiLink: enum snd_soc_dpcm_trigger trigger[2]

	SndSocDaiLink: unsigned int ignore_suspend

	SndSocDaiLink: unsigned int symmetric_rates
	SndSocDaiLink: unsigned int symmetric_channels
	SndSocDaiLink: unsigned int symmetric_samplebits

	SndSocDaiLink: unsigned int no_pcm

	SndSocDaiLink: unsigned int dynamic

	SndSocDaiLink: unsigned int dpcm_capture
	SndSocDaiLink: unsigned int dpcm_playback

	SndSocDaiLink: unsigned int ignore_pmdown_time

	SndSocDaiLink: int (*init)(struct snd_soc_pcm_runtime *rtd)

	SndSocDaiLink: int (*be_hw_params_fixup)(struct snd_soc_pcm_runtime *rtd,struct snd_pcm_hw_params *params)

	SndSocDaiLink: const struct snd_soc_ops *ops
	SndSocDaiLink: const struct snd_soc_compr_ops *compr_ops

	SndSocDaiLink: bool playback_only
	SndSocDaiLink: bool capture_only
	
class SndSocPcmRuntime
	SndSocPcmRuntime: struct device *dev
	SndSocPcmRuntime: struct snd_soc_card *card
	SndSocPcmRuntime: struct snd_soc_dai_link *dai_link
	SndSocPcmRuntime: struct mutex pcm_mutex
	SndSocPcmRuntime: enum snd_soc_pcm_subclass pcm_subclass
	SndSocPcmRuntime: struct snd_pcm_ops ops

	SndSocPcmRuntime: unsigned int dev_registered

	SndSocPcmRuntime: struct snd_soc_dpcm_runtime dpcm[2]
	SndSocPcmRuntime: int fe_compr

	SndSocPcmRuntime: long pmdown_time
	SndSocPcmRuntime: unsigned char pop_wait

	SndSocPcmRuntime: struct snd_pcm *pcm
	SndSocPcmRuntime: struct snd_compr *compr
SndSocPcmRuntime: 	struct snd_soc_codec *codec
	SndSocPcmRuntime: struct snd_soc_platform *platform
	SndSocPcmRuntime: struct snd_soc_dai *codec_dai
	SndSocPcmRuntime: struct snd_soc_dai *cpu_dai
	SndSocPcmRuntime: struct snd_soc_component *component

	SndSocPcmRuntime: struct snd_soc_dai **codec_dais
	SndSocPcmRuntime: unsigned int num_codecs

	SndSocPcmRuntime: struct delayed_work delayed_work
	SndSocPcmRuntime: struct dentry *debugfs_dpcm_root
	SndSocPcmRuntime: struct dentry *debugfs_dpcm_state


	SndCard..>SndSocCard: Strcut snd_card
	SndSocDaiLink..>SndSocCard: Strcut snd_soc_dai_link
	SndSocPcmRuntime..>SndSocCard: struct snd_soc_pcm_runtime
	

	
```

### ASOC初始化过程

我们以ADI 的SC573 ezkit的声卡注册为例，下图为`snd_soc_card`的初始化和注册过程,其中有大量的`platform_driver`和`platform_device` 的使用。

```mermaid
sequenceDiagram
	participant card as sc5xx-asoc-card.c
	participant ldrvres as soc-drvres.c
	participant lcore as soc-core.c
	participant linit as sound/core/init.c


	
	activate card
	note over card:.name="sc5xx-asoc-card"
	note over card:.dai_link=sc5xx_asoc_dai_links
	
	note left of card:dai_links={{sc5xx_adau1962a_init,&adau1962_ops},{1979_init,&1979_ops}}
	card->>ldrvres:devm_snd_soc_register_card
	deactivate card
	
	activate ldrvres
	ldrvres->>lcore:snd_soc_register_card
	deactivate ldrvres
	
	activate lcore
	lcore->>lcore:snd_soc_init_multicodec
	lcore->>lcore:snd_soc_instantiate_card
	lcore->>lcore:soc_bind_dai_link
	lcore->>lcore:soc_bind_aux_dev
	lcore->>lcore:snd_soc_init_codec_cache
	lcore->>linit:snd_card_new
	deactivate lcore
	
	activate linit
	deactivate linit
	
	activate lcore
	lcore->>lcore:soc_probe_link_components
	lcore->>lcore:soc_probe_link_dais
	lcore->>lcore:...
	lcore->>lcore:snd_card_register
	deactivate lcore



	
	
```
