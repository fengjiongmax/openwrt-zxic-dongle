# openwrt-zxic-dongle
openwrt通过usb使用zxic随身wifi

## 安装依赖包
```shell
opkg update
opkg install usb-modeswitch kmod-usb-serial kmod-usb-serial-option kmod-usb-net-cdc-ether usbutils
```
## 获取usb设备标识
以我手上的随身wifi为例子，在未进行任何改动的情况下，将其插入usb口后，在openwrt输入命令`lsusb`得到的是:
```
Bus 001 Device 002: ID 19d2:0548 ALK,Incorporated ALK Mobile Boardband
```
这时候是一个存储设备，识别符为`19d2:0548`。

但是如果将其插入到linux电脑的usb口，在以太网端口出现后，输入`lsusb`得到的是:
```
Bus 001 Device 046: ID 19d2:0536 ZTE WCDMA Technologies MSM ALK Mobile Boardband
```
所以对于这个设备，初始识别符为`19d2:0548`，目标识别符为`19d2:0536`。
## 写临时配置文件，测试
根据openwrt的文档:https://openwrt.org/docs/guide-user/network/wan/wwan/usb-modeswitching
，我们可以临时写一个usb-modeswitch的配置文件来测试能否正常切换模式。

在写配置文件的时候，要将目标识别符的冒号两端分别由16进制转换为10进制，同样以我手上的设备为例：  
`19d2:0536`:  

`0x19d2`->`6610`  

`0x0536`->`1334`  

我们就可以写一个测试用的json文件：
```json
{
	"messages" : [
		"55534243123456780000000000000011062000000100000000000000000000",
],
	"devices" : {
		"19d2:0548": {
			"*": {
				"t_vendor": 6610,
				"t_product": [ 1334 ],
				"msg": [ 0 ]
			}
		},
	},
}
```
我将这个文件放在`/root`目录，文件名为`usb-mode-custom.json`。
然后就来试一下这个文件:
```
usbmode -s -v -c /root/usb-mode-custom.json
```
这个时候我们再看`lsusb`的输出内容:
```
Bus 001 Device 047: ID 19d2:0536 ALK,Incorporated ALK Mobile Boardband
```
已经成功切换为以太网设备了，然后在看`ip a`，可以看到一个新增`eth1`，这个时候就可以通过luci来添加一个新的以太网设备，使用dhcp，防火墙为wan，使用随身wifi的数据了。

## 写入usb-modeswitch系统配置
打开`/etc/usb-mode.json`，将临时配置文件中`devices`里面的内容写到系统配置里面的`devices`里面，我的就是：
```json
#/etc/usb-mode.json
...
  "devices":{
#这里开始粘贴
  		"19d2:0548": {
  			"*": {
  				"t_vendor": 6610,
  				"t_product": [ 1334 ],
  				"msg": [ 0 ]
  			}
  		},
#这里结束粘贴
  }

...

```

这时候就可以自动切换usb模式了。


