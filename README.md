# 炒冷饭之在你上网的时候不经意间使用被动扫描(xray)挖洞。
@Author [众安天下](http://www.allsec.cn)-[凤凰安全实验室](http://www.allsec.cn)-[连长](http://lianzhang.org)

@Email admin@lianzhang.org

@Date 2020年06月30日
**文章辣鸡，未经同意禁止转载谢谢**

## 0x00 说明
  北京疫情严重端午节在家没事干，就整了这个冷饭版的东西，现在挖洞不好挖，各种waf，ajax请求，动态爬虫都不好使，大家伙也知道挖洞多不多取决于你的信息收集能力和平时积攒的一些东西。总有扫不到的地方，这就很难受了，就想着吧路由层面的http和https的流量转发到被动扫描上面扫描，把平时上网的流量转发到 被动上面，也可以多个被动一起转发，市面上被动有多不想造轮子，没撒意思，虽然自己也是个菜鸡，说不定运气好，挖到一个洞就当捡漏，也可以当做记录web日志的工具，流量都存在mangodb里面，方便查看，定时查看一些流量可以分析出谁在偷你的数据。也好，一拍即合开始肝。

## 0x01 实践
 传统的代理扫描，无法部署在其他节点上，比如路由器，如果直接转发代理流量很可能会造成因为扫描行为造成无法上网的尴尬情况，这个时候需要一个代理转发的东西，只转发流量不会影响直接上网，最主要是不能影响正常上网，不然没什么意义对吧，我们要的只是我们认为去触发的那些包。还可以支持多级代理转发，转发到多个被动扫描，
golang 实现透明代理。没啥技术难度，我这么菜鸡的人都能写出来https劫持必须在客户端上安装证书。毕竟是自己的设备无妨，辛苦一点手动装个证书。
然后分别实现http和https的代理。如下：
![-w628](http://mweb.03sec.com/15931926573639.jpg)
直接github 干一个出来http和https的代理，不过自己平时上网的设备也不多，辛苦就手动安装证书。
然后把代理过来的get和post重新代理被动扫描的代理发一次包。这样实现也很简单，代码比较废物，直接忽略吧。
![-w599](http://mweb.03sec.com/15932380400120.jpg)
然后我们
![-w1272](http://mweb.03sec.com/15932380620161.jpg)
得到了完整 请求。
然后完善一下基本功能编译。因为时间问题没加上入库mangodb。
流量是转发过去了，但是因为法律的问题，我们不能都扫描，不然会吃枪子，这时候就需要白名单机制，这对挖src有好处，不必担心法律问题，
## 0x02 FunkProxy 使用方法
FunkProxy 下载地址：[https://github.com/hackxx/FunkProxy/releases](https://github.com/hackxx/FunkProxy/releases/tag/2.0.1)
**_有问题和建议请提issues，觉得好请star~_**
```shell script
Usage: FunkProxy [options]

  -TowProxy string
        set tow proxy http://127.0.0.1:8080/.
  -addr string
        Bind address. (default "0.0.0.0")
  -ca-cert string
        Path to root CA certificate. (default "ca/rootCA.crt")
  -ca-key string
        Path to root CA key. (default "ca/rootCA.key")
  -dns string
        Custom DNS server that bypasses the OS settings
  -flist string
         Forward white list. (default ".baidu.com|.qq.com|.vulnweb.com|.javaweb.org")
  -h    Shows usage options.
  -http uint
        Bind port (HTTP mode). (default 10080)
  -https uint
        Bind port (SSL/TLS mode). Requires --ca-cert and --ca-key. (default 10443)
  -v    Shows FunkProxy version.
```
主要功能有：
1、http和https代理
2、域名白名单，白名单用户的包开启转发
3、资源文件简单过滤、过滤png、jpg、css、js、字体文件等等文件转发，降低xray负载。
4、通过返回头过滤需要转发的数据包，提高效率。
用法：
`./funkproxy_linux_amd64 -ca-cert ca/rootCA.crt -ca-key ca/rootCA.key -http 10080 -https 10443 -TowProxy http://xrayip:xrayport -flist .baidu.com|.qq.com`

**推荐方法：openwrt 路由不运行FunkProxy 通过iptables转发**

## 0x03 实践方案
### 一、openwrt 路由器
  不一定需要openwrt的路由器 也可以是其他的路由，这个方案有点像广告过滤的东西。前提你是要有路由器的root控制权限，具体看自己怎么操作了吧:
下载 FunkProxy_linux_mipsle，看你路由器的架构吧，但是由于包过大可能有些路由器放不下这么大的文件，有2个选择一个是使用下方的linux转折方案，一个路由器上挂个sd卡，把程序放到sd卡上跑。
首先吧xray跑起来。
```shell script
sudo ./xray_darwin_amd64 webscan --listen 0.0.0.0:7777 --html-output Funk.html
```
然后使用iptables转发。

```shell script
# 所有转发
iptables -A PREROUTING -t nat -i br-lan -p tcp --destination-port 80 -j REDIRECT --to-port 10080
iptables -A PREROUTING -t nat -i br-lan -p tcp --destination-port 443 -j REDIRECT --to-port 10443

# 指定ip转发
iptables -A PREROUTING -t nat -s 指定ip -i br-lan -p tcp --destination-port 80 -j REDIRECT --to-port 10080
iptables -A PREROUTING -t nat -s 指定ip -i br-lan -p tcp --destination-port 443 -j REDIRECT --to-port 10443
```
然后上传文件和证书至路由器执行命令

```shell script
./FunkProxy_linux_mipsle -ca-cert ca/rootCA.crt -ca-key ca/rootCA.key -http 10080 -https 10443 -TowProxy http://xrayip地址:端口
```
然后xray就会接受到参数。
![-w1067](http://mweb.03sec.com/15935001915490.jpg)
代理也能接受到参数。
![-w631](http://mweb.03sec.com/15935008309028.jpg)





### 二、linux 转发
本次方案是路由器没办法修改防火墙配置或者没有root权限的转折方案。
实验环境是 centos7 和 win server2008一台。
linux作为网关、server2008作客户端。或者其他设备都可以。
首先开启centos7 的 ip转发。

```shell script
cat /proc/sys/net/ipv4/ip_forward # 查看IP转发功能是否开启
sysctl -w net.ipv4.ip_forward = 1 # 只能临时生效，永久生效请自行百度
```
并且关闭原有的防火墙
```shell script
firewall-cmd --state
systemctl stop firewalld.service
systemctl disable firewalld.service
```
使用iptables 防火墙进行端口转发，把80和443端口转发到我们代理。这里可以设置成指定IP段的端口转发。

转发所有的IP：
```shell script
iptables -A PREROUTING -t nat -i eth0 -p tcp --destination-port 80 -j REDIRECT --to-port 10080
iptables -A PREROUTING -t nat -i eth0 -p tcp --destination-port 443 -j REDIRECT --to-port 10443
```
转发指定的IP：
```shell script

iptables -A PREROUTING -t nat -s 需要转发的ip -i eth0 -p tcp --destination-port 80 -j REDIRECT --to-port 10080
iptables -A PREROUTING -t nat -s 需要转发的ip -i eth0 -p tcp --destination-port 443 -j REDIRECT --to-port 10443
```


运行FunkProxy代理

```shell script
./hyperfox_linux_amd64 -ca-cert ca/rootCA.crt -ca-key ca/rootCA.key -http 10080 -https 10443 -TowProxy http://你的xrayip:端口

```
客户端配置
我们这里用的客户端是win server 2008 
![-w391](http://mweb.03sec.com/15933334023790.jpg)
把网关配置成我们的Centos7IP地址，因为centos7已经开启ip转发所有可以直接上网，
我们也可以在DHCP服务器更改网关地址。
![-w1127](http://mweb.03sec.com/15933347771851.jpg)
安装上更证书然后我们就收到数据了完整数据包。
因为是手动指定的网关，我们也可以修改DHCP服务器吧网关变成我们目标ip。
具体自己设置。


### 三、openwrt 路由不运行FunkProxy 通过iptables转发
因为路由器的闪存比较小，上传不了FunkYou主程序咋办？
我们可以利用iptables 吧端口转发到某个ip的端口上。
首先利用内网必须有台机器：

```shell script
./FunkProxy_linux_amd64  -ca-cert ca/rootCA.crt -ca-key ca/rootCA.key -http 10080 -https 10443 -TowProxy http://127.0.0.1:7777
```
![-w1073](http://mweb.03sec.com/15935076924264.jpg)

```shell script
./xray_linux_amd64 webscan --listen 127.0.0.1:7777 --html-output ali111.html
```

![-w929](http://mweb.03sec.com/15935076824739.jpg)

登陆到你的路由器执行ip转发

```shell script
iptables -A PREROUTING -t nat -s 被转发目标 -i br-lan -p tcp --destination-port 80 -j DNAT --to-destination 你的内网服务器ip:10080
iptables -A PREROUTING -t nat -s 被转发目标 -i br-lan -p tcp --destination-port 443 -j DNAT --to-destination 你的内网服务器ip:10443
```
![-w1117](http://mweb.03sec.com/15935082015537.jpg)
然后你的代理就是收到啦。
![-w851](http://mweb.03sec.com/15935082237383.jpg)
xray正常工作。

经测试openwrt的防火墙策略需要更改如下，否则转发不出来包。
![](http://mweb.03sec.com/15935738082243.jpg)


## 0x04 结尾
  还有很多功能点会逐一完善，比如分布式转发到多个代理扫描，等等功能，但是由于时间的原因只能将就的去弄。程序有什么问题可以直接提交 issues，觉得好请star。
  友情提示，不要全流量做转发，因为可能涉及到法律问题。因为带了你很多cookie信息和其他的个人信息很容易定位到人。还有xray尽量配置一个动态代理会比较好，或者吧他打在外网的服务器上会好很多。
  经过测试还有有些问题的，比如还是有些资源文件会进入到xray扫描器，性能方面会收一些影响。
