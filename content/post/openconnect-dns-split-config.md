---
typora-copy-images-to: ./openconnect-dns-split-config.assets
typora-root-url: ./openconnect-dns-split-config.assets
tags: ["dns"]
---

# OpenConnect DNS Split 配置

Cisco的VPN连接程序Anyconnect Secure Mobility Client是没有配置任何DNS Split/Route Split的, 也就是在开启后,任何连接,包括DNS解析和所有TCP/IP流量都会从VPN的通道走,而且一般会因为VPN服务端的配置,任何本地局域网的出入流量都会被禁用,这样一来所有的下载(比如BT),和局域网内的服务(如AirDrop/AirPlay/SMB文件服务等依赖局域网联通和FQDN本地域名解析的服务)都不能使用了.幸好有openconnect这个实现了anyconnect协议的工具,我们可以方便的配置本地的路由方案和DNS解析方案.

## openconnect 脚本配置原理

默认情况下, openconnect本体程序只负责建立隧道连接,本地的路由规则和DNS解析都不会被配置,但是openconnect提供了用于回调的vpnc-script脚本来负责完成这些事情.vpnc-script脚本是openconnect在完成连接/断开连接等动作后会调用的脚本,而openconnect在编译时就执行了搜索这个脚本的位置,一般来说是/etc/vpnc-script.在MacOS下,有开源的OpenConnect-GUI使用,OpenConnect-GUI是一个将openconnect包装了一层的Mac标准程序,同时OpenConnect-GUI提供了默认的vpnc-script脚本来完成一般类型的初始化工作(主要是路由表的配置),这个vpnc-script脚本被打包进了OpenConnect-GUI程序本身.可以发现,OpenConnect-GUI在启动时会要求一个root权限,这是因为vpnc-script脚本会在root用户权限下被执行.因为openconnect在启动时只能指定一个脚本使用,不过OpenConnect-GUI的vpnc-script脚本留了一个口子,就是它会在执行脚本内容时,搜索在/etc/vpnc/目录下存在的Hooks来调用.通过研究vpnc-script脚本,发现它会在openconnect的不同执行目标下使用不同的Hooks脚本来完成额外的工作.这个对应关系如下:

- pre-init -> /etc/vpnc/pre-init.d/*

- connect -> /etc/vpnc/connect.d/*

- post-connect -> /etc/vpnc/post-connect.d/*
- disconnect -> /etc/vpnc/disconnect.d/*
- post-disconnect -> /etc/vpnc/post-disconnect.d/*
- reconnect -> /etc/vpnc/reconnect.d/*

这样通过添加额外的Hooks脚本就能达成我们定制化的目的.对我来说,我主要要在connect前和disconnect前做一些工作,我创建了一个叫做*override-dns-config.sh*的脚本,同时放在了*connect.d*和*disconnect.d*目录下(通过软连接方式,这样我不必在改动时同时改动两个相同的文件),文件内容如下:

```shell
modify_resolvconf_generic() {
	if [ "$OS" = "Darwin" ]; then
		case "`uname -r`" in
			# Skip for pre-10.4 systems
			4.*|5.*|6.*|7.*)
				;;
			# 10.4 and later require use of scutil for DNS to work properly
			*)
				scutil >/dev/null 2>&1 <<-EOF
					open
					d.init
					d.add ServerAddresses * $INTERNAL_IP4_DNS
					d.add DomainName sankuai.com
					d.add SupplementalMatchDomains * ${CISCO_SPLIT_DNS//,/ } qahome.dp
					set State:/Network/Service/$TUNDEV/DNS
					d.init
					d.add Addresses * $INTERNAL_IP4_ADDRESS
					d.add SubnetMasks * 255.255.255.255
					d.add InterfaceName $TUNDEV
					set State:/Network/Service/$TUNDEV/IPv4
					close
				EOF
				;;
		esac
	fi
}

restore_resolvconf_generic() {
	if [ "$OS" = "Darwin" ]; then
		case "`uname -r`" in
			# Skip for pre-10.4 systems
			4.*|5.*|6.*|7.*)
				;;
			# 10.4 and later require use of scutil for DNS to work properly
			*)
				scutil >/dev/null 2>&1 <<-EOF
					open
					remove State:/Network/Service/$TUNDEV/IPv4
					remove State:/Network/Service/$TUNDEV/DNS
					close
				EOF
				;;
		esac
	fi
}
```

下面解释一下脚本完成的工作.其中的*modify_resolvconf_generic/restore_resolvconf_generic*函数是在*connect/disconnect*在自身执行完的最后阶段所调用的函数,而根据**Bash Shell**的语法,函数是可以被覆盖的,于是我就在*connect/disconnect*的前面的Hooks放置了包含这两个函数的脚本来覆盖原有脚本的配置.

## 系统DNS配置

上面所说的这两个函数主要是基于[*System Dynamic Configuration*][sys-dynamic-conf]进行的一些配置,其中又主要配置了`State:/Network/Service/utun5/DNS`和`State:/Network/Service/utun5/IPv4`这两个字段.

接下来解释一下MacOS的*System Dynamic Configuration*系统,这个系统主要用来对网络环境进行配置.这个系统本身是一个配置系统,配置本身通过磁盘上的文件*.plist和内存中的一些数据结构来存储,这些配置内容通过configd这个守护进程来管理,在平时对这些内容操作时可以使用`scutil`工具.`scutil`可以通过标准输入来获取配置,但同时也是一个命令行中交互式的工具.在不带任何参数的情况下使用`scutil`会进入交互模式,在交互模式中输入`help`命令可以看到如下面截图中所示的所有可用命令.

![截屏2020-02-22上午11.42.20](/image-11.42.20-2344549.png)

通过使用`list`命令,我们能看到所有当前的Key,这其中被Setup和State开头的Key占了多半部分,其中Setup开头的是指目前持久化到磁盘上的配置,而State也就是我们将用来修改的,是当前在内存中那部分配置.

MacOS的DNS配置系统会通过查看`State:/Network/Service/**/DNS`的配置来搜索可以使用的配置.openconnect连接后,会创建类似*utunX*这样的网络设备,我这里是*utun5*,所以我们就在`State:/Network/Service/utun5/DNS`这个Key中进行配置(我猜测理论上应该在任何的`State:/Network/Service/*/DNS`中配置都能被MacOS系统扫描到,只是在`Service/utun5/DNS`里面配置在使用逻辑上更为接近).

`State:/Network/Service/utun5/DNS`内部的数据结构如下(使用`show State:/Network/Service/utun5/DNS`命令查看):

```
<dictionary> {
  DomainName : google.com
  SearchDomains: google.com google.cn id.google.com
  ServerAddresses : <array> {
    0 : 192.168.4.251
    1 : 192.168.6.251
  }
  SupplementalMatchDomains : <array> {
    0 : google.com.hk
    1 : facebook.com
  }
}
```

这里面有两个重要的字段,**DomainName/ServerAddresses**一般来情况下的DNS配置就只使用这两个字段,含义相当于“当遇到**DomainName**域名的时候,要使用**ServerAddresses**来解析”.但是当我们有多个域名都想用这个**ServerAddresses**来解析怎么办呢?方法是使用[**SupplementalMatchDomains**][match-domains]字段.通过在这个字段中添加多个域名来达到让这些域名都使用**ServerAddresses**来解析的目的,所以**SupplementalMatchDomains**和**DomainName**一起,定义了一个需要被当前**ServerAddresses**解析的域名列表.(这个时候就可以随便发挥了, 比如**DomainName**随便填写一个域名,不存在的也行,所有需要解析的域名都在**SupplementalMatchDomains**中填写;或者是**DomainName**填写一个域名,剩下的在**SupplementalMatchDomains**中填写;或者是在**DomainName**定义的域名,在**SupplementalMatchDomains**中再定义一遍,因为重复没有什么问题)

这里顺便解释一下困惑了我很久的问题——什么是[*Search Domain*][search-domain-explain].即当我们输入一个域名的子域名部分时, 系统能帮忙我们自动将子域名对应的剩下的域名填充完整.显然,这个功能目前已经没什么作用了,因为浏览器中提供的自动补全已经很方便了,不过这个功能可能在局域网FQDN的名字补全的时候还是方便不少.

## scutil命令使用

scutil命令可以在交互模式下使用,也可以从标准输入获取命令来进行配置;同时,scutil还通过对一些参数的支持直接在命令行进行简单的查看配置或者修改配置动作.

不使用任何参数时执行scutil可以进入交互模式,而从标准输入获取命令的方式可以参考上面示例脚本中的方法.不管使用这两种方式中的哪一种,都是使用同一组scutil指令.下面简单的介绍一下这些指令.

scutil围绕着键值对进行配置,而键值对的表现形式就是字典(dict),值则可以为字符串/数字/数组几种类型.

```
list
```

这个命令(list)用于获取配置服务(configd)中的所有配置列表项,这些列表项的名字都是类似目录结构的字符串.

```
show State:/Network/Service/utun5/DNS
```

这个命令(show)用于查看某个配置项的内容,内容会用可视化的字典(dict)来展示出来.

```
get State:/Network/Service/utun5/DNS
d.show
d.add DomainName google.com
d.remove SearchDomains
d.add SupplementalMatchDomains * google.com.hk facebook.com
set State:/Network/Service/utun5/DNS
```

一般上面的命令都是成组使用的

-  `get State:/Network/Service/utun5/DNS`获取执行配置项的内容并保存在当前的一个变量`d`中,然后使用`d`的命令来操作内容.
- `d.show`来查看这个变量中的内容,这个效果和执行`show State:/Network/Service/utun5/DNS`是相同的.
- 要修改内容字典中的某个值,使用`d.add`命令,指定要修改的字典中的键和修改后的值,这个`d.add`命令是一个类似“设置”的命令,而不是字面上的“添加”的含义,这个在之后的设置“数组”类型的值时也是这个含义,比如如果要向某键对应的数组中附加一个值,实际的操作其实是将这个数数组类型的值重新设置一遍`d.add SupplementalMatchDomains * google.com.hk facebook.com`,注意这里的`*`代表这个值的类型是数组.
- `d.remove`命令来删除内容字典中的一个键值对.
- 当对这个内容字典都设置完成后,使用`set State:/Network/Service/utun5/DNS`将当前的内容字典变量`d`中的内容都设置到指定的配置项中去.

使用完成后,可以使用`quit`命令退出scutil的交互模式(从标准输入获取命令时,不需要把`quit`命令放在所有命令的最后)

当我们配置了**SupplementalMatchDomains**后,可以在命令行通过`scutil --dns`命令来查看配置的效果,比如我这边配置后会出现下面类似的展示.

![image-20200222143448176](/image-20200222143448176.png)

因为这里是实验,所以我使用了`google.com`这个域名,实际情况下,这个一般都是公司内网的域名地址.这个时候可以测试下域名是否可以被解析了.

```
dns-sd -q google.com
dns-sd -G v4 google.com
ping google.com
```

这里使用`dns-sd`命令,或者`ping`命令,是因为这两个命令在macos的实现会使用系统的DNS配置框架来查找合适的DNS服务器来进行域名解析.而`nslookup`和`dig`命令没有专门为macos系统的DNS配置框架进行适配,只会使用默认的DNS服务器进行域名解析.另外也可以使用浏览器试下,看看是不是自己公司的域名已经可以打开了.

## 其他DNS配置位置

上面提到的DNS配置系统是围绕[*System Dynamic Configuration*][sys-dynamic-conf]说明的,而macos上还有一个地方可以影响配置,即[`resolver`][reslover](我现在不太确认这个resolver是不是*System Dynamic Configuration*系统的一部分).

根据文档说明,我们可以在*/etc/resolver/*目录下(默认情况下*/etc/resolver*目录不存在,需要自己手动创建,记得需要root权限)创建任意数量的文件,DNS配置系统会从这些文件中读取配置.使用这种方式配置指定域名所使用的DNS服务器,有两种方式.

1. 创建一个以域名(google.com)为文件名的文件(注意必须是包括类似com这种的全域名),然后在文件中写上`nameserver 192.168.4.251`这样的域名解析服务器
2. 创建一个任意名称的文件,然后在文件中写上`domain google.com`和`nameserver 192.168.4.251`这两行

上面这两种方式都能完成对指定域名使用指定域名服务器的配置.经过验证,这种配置文件中的`search`命令并没有任何作用,所以使用这种方式并不能在一个文件中配置多个需要解析的域名,只能使用多个文件来完成这种操作.

## 总结

上面的所有方式提到的域名只需要配置一级域名,比如如果要解析*mail.google.com*和*photos.google.com*则只需要配置*google.com*这个域名的解析即可.

下面的关系图描述了我理解的macos的DNS解析配置系统:

![macos的DNS解析配置系统](http://assets.processon.com/chart_image/5e51334fe4b0362764fcc50e.png)

## 其他域名解析查询命令使用

nslookup

dig

[sys-dynamic-conf]: https://developer.apple.com/library/archive/documentation/Networking/Conceptual/SystemConfigFrameworks/SC_Intro/SC_Intro.html#//apple_ref/doc/uid/TP40001065-CH201-TPXREF101	"System Dynamic Configuration"

[search-domain-explain]: https://support.apple.com/guide/mac-help/enter-dns-and-search-domain-settings-on-mac-mh14127/mac	"Search Domain Explain"
[match-domains]: https://developer.apple.com/documentation/devicemanagement/vpn/dns	"SupplementalMatchDomains"
[reslover]: https://www.manpagez.com/man/5/resolver/ "resolver man page"