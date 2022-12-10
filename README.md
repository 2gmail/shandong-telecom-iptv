# 山东电信IPTV收录索引与使用指南

![chinatelecom](https://www.189.cn/image/189cnv2/indexv2/img_head/logo.png "China Telecom")

本项目将收录单播（Unicast）与多播（Multicast）两种直播源，其中多播需要使用udpxy或其他工具进行转发代理，代理服务器可以自行搭建或寻找。

## 适用

适用于：

1. 光猫需要有telcomadmin权限
2. 光猫不改桥接
3. 光猫到路由器使用单线连接

## 光猫设置

#### 使用前记录下IPTV的VLAN ID，这里为43

![image](https://user-images.githubusercontent.com/1812118/88484574-15771a00-cfa2-11ea-9bea-3cb6f8146426.png)

#### 山东电信默认使用LAN2口作为IPTV口，此教程的目的是将互联网绑定4个LAN口，IPTV公用LAN1。因此可以先将互联网的LAN口都勾选上

![image](https://user-images.githubusercontent.com/1812118/88484953-0f366d00-cfa5-11ea-9265-6b75db7349e1.png)

#### 修改IPTV网络配置，将端口都解绑。这一步比较重要，因为互联网和IPTV在LAN口的的VLAN ID不能一样，否则会DHCP冲突

![image](https://user-images.githubusercontent.com/1812118/88484632-874f6380-cfa2-11ea-9b13-9dac4596c91d.png)

#### 修改VLAN绑定，手动将LAN1、VLAN ID 43、IPTV网络进行绑定。此时LAN1口同时携带不带VLAN ID的互联网数据和带VLAN ID 43的IPTV数据

![image](https://user-images.githubusercontent.com/1812118/88484655-bb2a8900-cfa2-11ea-963e-c2a2a1d3c1f8.png)

> 对于其他地区（如北京联通），可能互联网已经和IPTV绑定在同一个LAN口了，可以直接跳过以上几个步骤，同时也就不需要管理员账号了。

## 路由器设置

#### 安装几个必要的包

```bash
opkg install luci-app-udpxy igmpproxy
```

#### 配置 `/etc/config/igmpproxy`，其中

##### `list altnet 239.21.1.0/24` 需要写组播服务器的地址段（可以搜索别人做的好组播地址列表，然后找到合适的IP段，一般情况下运营商不会使用太复杂的IP段）

##### `option network iptv` 为IPTV接口（注意不是link名，是 OpenWRT后台——网络——接口 处的名字）

##### `option network lan` 为LAN接口（注意不是link名，是 OpenWRT后台——网络——接口 处的名字）

```conf
config igmpproxy
 option quickleave 1
# option verbose [0-3](none, minimal[default], more, maximum)

config phyint
 option network iptv
 option zone wan
 option direction upstream
 list altnet 239.21.1.0/24

config phyint
 option network lan
 option zone lan
 option direction downstream
```

##### 重启 igmpproxy （可以直接重启路由器）

#### 添加一个新接口IPTV

##### 物理接口使用eth0.xx，其中xx为IPTV的VLAN ID

##### 协议使用DHCP

##### 在高级设置中关闭“使用默认网关”。这一步比较重要，原因是我们只需要对组播IP段进行路由即可，关闭默认路由并使用静态路由会更加简单

![image](https://user-images.githubusercontent.com/1812118/88484794-ec578900-cfa3-11ea-80d7-541ded1431c6.png)

![image](https://user-images.githubusercontent.com/1812118/88484794-ec578900-cfa3-11ea-80d7-541ded1431c6.png)

##### 在“防火墙设置”中选择“不指定或新建”；或者选择 `wan` ，并添加一条防火墙规则

##### 在 OpenWRT后台——网络——防火墙——通信规则 添加一个新的转发规则。配置如下

![image](https://user-images.githubusercontent.com/1812118/124494322-48986f00-dde9-11eb-91fc-9d5daf29b741.png)
![image](https://user-images.githubusercontent.com/1812118/124494397-5e0d9900-dde9-11eb-81ae-61087662a45c.png)

名称：`Allow-udpxy`
传输协议：`UDP`
源区域：`WAN`
目标区域：`设备（输入）`
目标地址：`224.0.0.0/4`

#### 添加一条交换机配置，使用IPTV的VLAN ID，将CPU和WAN设置为“已标记”，其他为“关”。原因是光猫会将IPTV数据打上VLAN ID并与互联网数据一起从LAN4传输，路由器里需要做同样的事情——数据进入时解除VLAN ID、流出时添加VLAN ID

![image](https://user-images.githubusercontent.com/1812118/88485789-3abc5600-cfab-11ea-948a-ef610a81cec2.png)

添加一条静态路由，将组播IP段路由到IPTV

```conf
config route
        option interface 'iptv'
        option type 'multicast'
        option target '224.0.0.0/4'
```

#### 开启udpxy服务，并且接口设置为IPTV的eth0.xx

![image](https://user-images.githubusercontent.com/1812118/88484871-8c151700-cfa4-11ea-8c83-95343aa860be.png)

## 检查服务

访问udpxy地址 <http://192.168.x.y:4022/status> ，IP为路由器的IP

如果未能成功访问udpxy地址，或igmp没有生效，请检查并确保这两个服务正常启动了

```bash
/etc/init.d/udpxy enable
/etc/init.d/igmpproxy enable
/etc/init.d/udpxy start
/etc/init.d/igmpproxy start
```
