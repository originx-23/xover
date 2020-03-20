---
title: "Openstack Attach Volume Flow"
date: 2020-03-19T11:41:16+08:00
draft: false
---

# Openstack attach volume flow

## Nova服务

 Nova组件为OpenStack提供计算服务(Compute as Service)，类似AWS的EC2服务。Nova管理的主要对象为云主机（server)，用户可通过Nova API申请云主机(server)资源。云主机通常对应一个虚拟机，但不是绝对，也有可能是一个容器(docker driver)或者裸机(对接ironic driver)。 

## Cinder服务

 Cinder组件为OpenStack提供块存储服务(Block Storage as Service)，类似AWS的EBS服务。Cinder管理的主要对象为数据卷(volume)，用户通过Cinder API可以对volume执行创建、删除、扩容、快照、备份等操作。 

 Cinder目前最典型的应用场景就是为Nova云主机提供云硬盘功能，用户可以把一个volume卷挂载到Nova的云主机中，当作云主机的一个虚拟块设备使用。 

 Cinder除了能够为Nova云主机提供云硬盘功能，还能为裸机、容器等提供数据卷功能。 

## 如何挂载块设备到虚拟机

 iSCSI设备需要先把lun设备映射到宿主机本地，然后当做本地设备挂载即可。 会创建一个xml文件，例如：

```xml
<disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/disk/by-path/ip-10.0.0.2:3260-iscsi-iqn.2010-10.org.openstack:volume-2ed1b04c-b34f-437d-9aa3-3feeb683d063-lun-0'/>
      <target dev='vdb' bus='virtio'/>
      <serial>2ed1b04c-b34f-437d-9aa3-3feeb683d063</serial>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</disk>
```

其中 `source`就是lun设备映射到本地的路径。

如果是分布式存储，以Ceph rbd为例，可以直接挂载rbd `image`（宿主机需要包含rbd内核模块），通过rbd协议访问image，而不需要先`map`到宿主机本地，一个demo xml文件为: 

```xml
<disk type='network' device='disk'>
      <driver name='qemu' type='raw' cache='writeback'/>
      <auth username='admin'>
        <secret type='ceph' uuid='bdf77f5d-bf0b-1053-5f56-cd76b32520dc'/>
      </auth>
      <source protocol='rbd' name='nova-pool/962b8560-95c3-4d2d-a77d-e91c44536759_disk'>
        <host name='10.0.0.2' port='6789'/>
        <host name='10.0.0.3' port='6789'/>
        <host name='10.0.0.4' port='6789'/>
      </source>
      <target dev='vda' bus='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
</disk>
```

Cinder如果使用Cinder macrosan driver，需要先把LV加到iSCSI target中，然后映射到计算节点的宿主机，而如果使用rbd driver，不需要映射到计算节点，直接挂载即可。 

 接下来介绍OpenStack如何将volume挂载到虚拟机中，分析Nova和Cinder之间的两种挂载方式。 

## iscsi方式挂载

### 跨设备存储的架构

#### 什么是iSCSI

iSCSI server端称为`Target`，client端称为`Initiator`，一台服务器可以同时运行多个Target，一个Target可以认为是一个物理存储池，它可以包含多个`backstores`，backstore就是实际要共享出去的设备，实际应用主要有两种类型：

- block。即一个块设备，可以是本地的一个硬盘，如`/dev/sda`，也可以是一个`LVM`(LogicalVolumeManager)卷。
- fileio。把本地的一个文件当作一个块设备，如一个raw格式的虚拟硬盘。

backstore需要添加到指定的target中，target会把这些物理设备映射成逻辑设备，并分配一个id，称为LUN(逻辑单元号)。

#### 架构示意图

![1571107234685](E:\文档\生产PPT\Openstack attach volume flow.assets\1571107234685.png)

### 挂载流程

#### 1. 基本流程

![volume attach](E:\文档\生产PPT\Openstack attach volume flow.assets\5a1fda3a-e17b-11e5-98be-2bca8a26e0bb.png)

当Nova volume-attach server volume执行后，主要经过以下几步：
a. Nova Client解析指令，通过RESTful接口访问nova-api；
b. Nova API解析响应请求获取虚拟机的基本信息，然后向cinder-api发出请求保留，并向nova-compute发送RPC异步调用请求卷挂载；
c. Nova-compute向cinder-api初始化信息，并根据初始化连接调用Libvirt的接口完成挂卷流程；
d. 进而调用cinder-volume获取连接，获取了连接后，通过RESTFUL请求cinder-api进行数据库更新操作。

#### 2.源码分析

##### 1. 源码函数

在Nova Client进程中，由VolumeAttachmentController接受挂载请求

```python
nova client
    ↓
POST /v2.1/servers/c2486c45-5a8f-4028-8cd1-51c0425f677a/os-volume_attachments
    ↓
nova.api.openstack.compute.volumes.VolumeAttachmentController.create
    self.compute_api.attach_volume
        self.compute_api 可用的有两个：
            1. nova.compute.cells_api.ComputeCellsAPI
            2. nova.compute.api.API
```

根据 nova.conf 配置项决定使用哪一个，默认情况下是 nova.compute.api.API

```python
nova.compute.api.API.attach_volume
    self._attach_volume
        ↓
nova.compute.api.API._attach_volume
    step1: volume_bdm = self.compute_rpcapi.reserve_block_device_name
    step2: volume = self.volume_api.get(context, volume_id)
    step3: self.volume_api.check_attach(context, volume, instance=instance)
    step4: self.volume_api.reserve_volume(context, volume_id)
    step5: self.compute_rpcapi.attach_volume(context, instance=instance,
                    volume_id=volume_id, mountpoint=device, bdm=volume_bdm)    
```

**step 1**: 创建 BDM 数据库表数据

```python
nova.compute.api.API._attach_volume
    volume_bdm = self.compute_rpcapi.reserve_block_device_name
        ↓
nova.compute.rpcapi.ComputeApi.reserve_block_device_name
    volume_bdm = cctxt.call(ctxt, 'reserve_block_device_name', **kw)
        ↓
nova.compute.manager.reserve_block_device_name
    bdm = objects.BlockDeviceMapping(...)
    bdm.create() # 创建 BDM 数据库表数据  
```

 即在`block_device_mapping`表中创建对应的记录，由于API节点无法拿到目标虚拟机挂载后的设备名，比如`/dev/vdb`，只有计算节点才知道自己虚拟机映射到哪个设备。因此bdm不是在API节点创建的，而是通过RPC请求到虚拟机所在的计算节点创建，请求方法为`reserve_block_device_name`，该方法首先调用libvirt分配一个设备名，比如`/dev/vdb`，然后创建对应的bdm实例。 

**step 2**: 调用 cinderclient 获取需要挂载的 volume 信息

```python
nova.compute.api.API._attach_volume
    volume = self.volume_api.get(context, volume_id)
        ↓
nova.volume.API.get
    item = cinderclient(context).volumes.get(volume_id)
```

**step 3**: 检查cinder status和可用域

```python
nova.compute.api.API._attach_volume
    self.volume_api.check_attach(context, volume, instance=instance)
        ↓
nova.volume.API.check_attach
    if (volume['status'] not in ("available", 'in-use', 'attaching'))
    if instance_az != volume['availability_zone']
```

 `check_attach`就是检查这个volume能不能挂载，比如status必须为`avaliable`，或者支持多挂载情况下状态为`in-use`或者`avaliable`。 

**step 4**: 调用 cinderclient 更新 cinder DB -> 'status': 'attaching'（以防止其他操作进入）

```python
nova.compute.api.API._attach_volume
    self.volume_api.reserve_volume(context, volume_id)
        ↓
nova.volume.API.reserve_volume
     cinderclient(context).volumes.reserve(volume_id)
```

`reserve_volume`是由Cinder完成的，nova-api会调用cinder API。该方法其实不做什么工作，仅仅是把volume的status置为`attaching`。 

**step 5**: 挂载卷

 此时nova-api会向目标计算节点发起RPC请求，由于`rpcapi.py`的`attach_volume`方法调用的是`cast`方法，因此该RPC是异步调用。由此，nova-api的工作结束，剩下的工作由虚拟机所在的计算节点完成。 

```python
nova.compute.api.API._attach_volume
    self.compute_rpcapi.attach_volume(context, instance=instance,
                    volume_id=volume_id, mountpoint=device, bdm=volume_bdm)  
        ↓
nova.compute.rpcapi.ComputeAPI.attach_volume
     cctxt = self.client.prepare(server=_compute_host(None, instance),
                version=version)
     cctxt.cast(ctxt, 'attach_volume', **kw)
        ↓
nova.compute.manager.ComputeManager.attach_volume
     step6: driver_bdm = driver_block_device.convert_volume(bdm) 
     step7: self._attach_volume(context, instance, driver_bdm)  
```



**step 6**: 根据 source_type 获取 BDM 驱动 DriverVolumeBlockDevice

```python
nova.compute.manager.ComputeManager.attach_volume
     driver_bdm = driver_block_device.convert_volume(bdm) 
        ↓
nova.virt.block_device.convert_volume
     source_volume = convert_volumes(volume_bdms)
     convert_volumes = functools.partial(_convert_block_devices,
                                   DriverVolumeBlockDevice)
```

**step 7**: BDM driver attach

```python
nova.compute.manager.ComputeManager.attach_volume
     self._attach_volume(context, instance, driver_bdm)
        ↓
nova.compute.manager.ComputeManager._attach_volume  
     bdm.attach(context, instance, self.volume_api, self.driver,
                       do_check_attach=False, do_driver_attach=True)
        ↓           
nova.virt.block_device.DriverVolumeBlockDevice.attach
     volume_api.check_attach(context, volume, instance=instance) # 检查cinder status和可用域
     step8:  connector = virt_driver.get_volume_connector(instance)
     step9:  connection_info = volume_api.initialize_connection(context,volume_id,connector)
     step10: virt_driver.attach_volume()  
```

**step 8**: 获取 connector，为 initialize_connection 提供参数

```python
nova.virt.block_device.DriverVolumeBlockDevice.attach
     connector = virt_driver.get_volume_connector(instance)
        ↓
nova.virt.libvirt.driver.LibvirtDriver.get_volume_connector
     self._initiator = libvirt_utils.get_iscsi_initiator()
     self._fc_wwnns = libvirt_utils.get_fc_wwnns()
     self._fc_wwpns = libvirt_utils.get_fc_wwpns()
     return connector
```

`get_volume_connector() `就是调用os-brick的`get_connector_properties`， os-brick是从Cinder项目分离出来的，专门用于管理各种存储系统卷的库，代码仓库为[os-brick](https://github.com/openstack/os-brick)。其中`get_connector_properties`方法位于`os_brick/initiator/connector.py` 

```python
def get_connector_properties(root_helper, my_ip, multipath, enforce_multipath,
                             host=None):
    iscsi = ISCSIConnector(root_helper=root_helper)
    fc = linuxfc.LinuxFibreChannel(root_helper=root_helper)

    props = {}
    props['ip'] = my_ip
    props['host'] = host if host else socket.gethostname()
    initiator = iscsi.get_initiator()
    if initiator:
        props['initiator'] = initiator
    wwpns = fc.get_fc_wwpns()
    if wwpns:
        props['wwpns'] = wwpns
    wwnns = fc.get_fc_wwnns()
    if wwnns:
        props['wwnns'] = wwnns
    props['multipath'] = (multipath and
                          _check_multipathd_running(root_helper,
                                                    enforce_multipath))
    props['platform'] = platform.machine()
    props['os_type'] = sys.platform
    return props
```

 该方法最重要的工作就是返回该计算节点的信息（如ip、操作系统类型等)以及`initiator name` 。

**step 9**: 存储端进行挂载返回 connection_info

```python
nova.virt.block_device.DriverVolumeBlockDevice.attach
     connection_info = volume_api.initialize_connection(context,volume_id,connector)
        ↓
nova.volume.cinder.API.initialize_connection
     return cinderclient(context).volumes.initialize_connection(volume_id,
                                                                   connector)
```

 会调用Cinder API的`initialize_connection`方法，该方法又会RPC请求给volume所在的`cinder-volume`服务节点。  该方法的重要工作就是调用`cinder-rtstool`的`add-initiator`子命令，即把计算节点的initiator增加到刚刚创建的target acls中。 

 因此Cinder的主要工作就是创建volume的iSCSI target以及acls。cinder-volume工作结束，我们返回到nova-compute。 

**step 10**: 主机端根据 connection_info 挂载卷到目标主机（iSCSI 为例）

```python
nova.virt.block_device.DriverVolumeBlockDevice.attach
     virt_driver.attach_volume() 
        ↓
nova.virt.libvirt.driver.LibvirtDriver.attach_volume
     self._connect_volume(connection_info, disk_info)
        ↓
nova.virt.libvirt.volume.iscsi.LibvirtISCSIVolumeDriver.connect_volume
     if self.use_multipath:
        run_iscsiadm_update_discoverydb()
        out = self._run_iscsiadm_bare()
        self._connect_to_iscsi_portal(props)
        self._rescan_iscsi()
     else:
        self._connect_to_iscsi_portal(iscsi_properties)
        self._run_iscsiadm(iscsi_properties, ("--rescan",))
     host_device = self._get_host_device(iscsi_properties)
```

在单路径的调用中会执行login、检查session、设置自启动等操作，如果一次未连接成功则还会每tries（2秒）重复调用，直到达到调用的限制。其中牵扯到的指令有：
a. 尝试连接

```shell
iscsiadm -m node -T target_iqn -p target_protal
```

b. 连接失败重新建立连接

```shell
iscsiadm -m node -T target_iqn -p target_protal -op new
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.authmethod -v auth_method
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.username -v auth_username
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.session.auth.password -v auth_password
```

c. 检查session，登陆

```shell
iscsiadm -m session检查是否登录成功
iscsiadm –m node –T targetname –p ip —login 登陆建立session
```

d. 设置为随机器启动而启动

```shell
iscsiadm -m node -T target_iqn -p target_protal —op update -n node.startup -v automatic
iscsiadm -m node -T target_iqn -p target_protal –rescan
```

**step 11**: 更新 cinder DB 数据

```python
nova.virt.block_device.DriverVolumeBlockDevice.attach
     volume_api.attach(context, volume_id, instance.uuid,
                       self['mount_device'], mode=mode)
        ↓
nova.volume.API.attach()
     cinderclient(context).volumes.attach(volume_id, instance_uuid,
                                          mountpoint, mode=mode)
```

最后调用了`volume_api.attach()`方法，该方法向Cinder发起API请求。此时cinder-api通过RPC请求到`cinder-volume`，代码位于`cinder/volume/manager.py`，该方法没有做什么工作，其实就是更新数据库，把volume状态改为`in-use`，并创建对应的`attach`记录。 

##### 2. 流程图

![img](E:\文档\生产PPT\Openstack attach volume flow.assets\clipboard.png)



## 分布式存储架构的执行流程(以Ceph-RBD为例)

### 分布式存储的架构

Nova与实例还有存储都运行在同一个设备上，不需要像iSCSI那样创建target、创建portal

![1570862669571](E:\文档\生产PPT\Openstack attach volume flow.assets\1570862669571.png)

如Ceph rbd

![img](E:\文档\生产PPT\Openstack attach volume flow.assets\952555-20170303101714329-1614216142-1571038953729.png)

通过 libvirt 可以把 Ceph 块设备用于 OpenStack ，它配置了 QEMU 到 librbd 的接口。 Ceph 把块设备映像条带化为对象并分布到集群中，这意味着大容量的 Ceph 块设备映像其性能会比独立服务器更好。 

### 挂载流程

因为不需要像iSCSI那样创建target、创建portal，因此rbd driver的`create_export()`方法为空: 

```python
def create_export(self, context, volume, connector):
    """Exports the volume."""
    pass
```

 `initialize_connection()`方法也很简单，直接返回image信息，如pool、image name、mon地址以及配置信息等。 

```python
def initialize_connection(self, volume, connector):
    hosts, ports = self._get_mon_addrs()
    data = {
        'driver_volume_type': 'rbd',
        'data': {
            'name': '%s/%s' % (self.configuration.rbd_pool,
                               volume.name),
            'hosts': hosts,
            'ports': ports,
            'auth_enabled': (self.configuration.rbd_user is not None),
            'auth_username': self.configuration.rbd_user,
            'secret_type': 'ceph',
            'secret_uuid': self.configuration.rbd_secret_uuid,
            'volume_id': volume.id,
            'rbd_ceph_conf': self.configuration.rbd_ceph_conf,
        }
    }
    LOG.debug('connection data: %s', data)
    return data
```

rbd不需要映射虚拟设备到宿主机，因此`connect_volume`方法也是为空。

剩下的工作其实就是nova-compute节点libvirt调用`get_config()`方法获取ceph的mon地址、rbd image信息、认证信息等，并转为成xml格式，最后调用`guest.attach_device()`即完成了volume的挂载。

```python
nova.virt.libvirt.driver.LibvirtDriver.attach_interface
	cfg = self.vif_driver.get_config(instance, vif, image_meta,
                                     instance.flavor,
                                     CONF.libvirt.virt_type,
                                     self._host)
 	↓
nova.virt.libvirt.driver.LibvirtDriver.attach_volume
	guest.attach_device(conf, persistent=True, live=live):
    	flags = persistent and libvirt.VIR_DOMAIN_AFFECT_CONFIG or 0
        flags |= live and libvirt.VIR_DOMAIN_AFFECT_LIVE or 0
        self._domain.attachDeviceFlags(conf.to_xml(), flags=flags)
```

因此，相对iSCSI，rbd挂载过程简单得多。


