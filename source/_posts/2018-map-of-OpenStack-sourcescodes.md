---
title: OpenStack组件源码地图
date: 2017-11-21 12:07:22
tags: [OpenStack]
---
## Python模块分发--setup.cfg
OpenStack子项目源码根目录存在setup.py文件和setup.cfg，用于Python模块分发；

以Neutron源码为例：https://github.com/openstack/neutron

### setup. py

```python
# File : ~/allen/Codes/neutron/setup.py

import setuptools

try:
    import multiprocessing  # noqa
except ImportError:
    pass

setuptools.setup(
    setup_requires=['pbr>=1.8'],
    pbr=True)
```

- 调用setuptools的setup()函数执行安装文件
- setup函数的参数设置保存在setup.cfg文件中
- 使用pbr工具去读取和过滤setup.cfg中的数据，将解析结果作为参数

> **PBR is a library that injects some useful and sensible default behaviors into your setuptools run.** It started off life as the chunks of code that were copied between all of the OpenStack projects. Around the time that OpenStack hit 18 different projects each with at least 3 active branches, it seemed like a good time to make that code into a proper reusable library.

### setup.cfg

```python
#  File : ~/allen/Codes/neutron/setup.cfg

[metadata]
name = neutron
summary = OpenStack Networking
description-file = 
	README.rst
author = OpenStack
...

[files]
packages = 
	neutron
data_files = 
	etc/neutron =
	etc/api-paste.ini
	etc/policy.json
...

[entry_points]
console_scripts = 
    neutron-l3-agent = neutron.cmd.eventlet.agents.l3:main
    neutron-openvswitch-agent = neutron.cmd.eventlet.plugins.ovs_neutron_agent:main
    neutron-server = neutron.cmd.eventlet.server:main
    ...
neutron.core_plugins = 
	ml2 = neutron.plugins.ml2.plugin:Ml2Plugin
neutron.service_plugins = 
    router = neutron.services.l3_router.l3_router_plugin:L3RouterPlugin
    ...
neutron.ml2.type_drivers = 
	flat = neutron.plugins.ml2.drivers.type_flat:FlatTypeDriver
	local = neutron.plugins.ml2.drivers.type_local:LocalTypeDriver
	vlan = neutron.plugins.ml2.drivers.type_vlan:VlanTypeDriver
	vxlan = neutron.plugins.ml2.drivers.type_vxlan:VxlanTypeDriver
	...
```

### entry points
从上可见，openstack各个子项目中setup.cfg注册了非常多的entry points，用于作为其在Python项目中的入口点；

对于一个Python包来说，entry point可以理解为它通过setuptools注册的外部可以直接调用的接口，以**ml2**的**type_drivers**为例：

```python
neutron.ml2.type_drivers = 
	flat = neutron.plugins.ml2.drivers.type_flat:FlatTypeDriver
	local = neutron.plugins.ml2.drivers.type_local:LocalTypeDriver
	vlan = neutron.plugins.ml2.drivers.type_vlan:VlanTypeDriver
	vxlan = neutron.plugins.ml2.drivers.type_vxlan:VxlanTypeDriver
	...
```



因Python安装模块位于/usr/lib/python2.7/site-packages/

则将前去 .../neutron/plugins/ml2.drivers/ 文件夹中，加载type_vxlan.py中的代码
```python
class VxlanTypeDriver(type_tunnel.EndpointTunnelTypeDriver):

    def get_endpoints(self):
        """Get every vxlan endpoints from database."""
        vxlan_endpoints = self._get_endpoints()
        
    def add_endpoint(self, ip, host, udp_port=p_const.VXLAN_UDP_PORT):
        return self._add_endpoint(ip, host, udp_port=udp_port)
    
    ...
```

而type_drivers的选择，则是在/etc/neutron/neutron.conf中去配置


**这就有了一个大概的OpenStack每个组件与其插件、配置文件中的调用逻辑和流程。**