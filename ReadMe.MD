# Clash For Merlin记录

> 设备：华硕AC-86U
>
> 固件：KoolShare改版Merlin 384.16
>
> Clash版本：0.19
>
> 1.0版 可以和UU加速器共存，并且Switch裸连可以达到B（根据当前网络决定） 

## 前言

之前尝试使用KOP-XIAO的[Clash-Merlin](https://github.com/KOP-XIAO/Clash-Merlin)在AC86U上安装，但是他的教程相对比较简单，并且有很多端口进行了自定义，在尝试使用透明代理过程中走了不少弯路，因此在此记录自己折腾的过程。

## 准备工作

1. Win系统下——Winscp和Xshell或者同类软件；Mac下——Termius或者同类软件。
2. 利用`uname -a`命令查看路由器架构，AC86U为armV8。
3. 在[Clash项目地址](https://github.com/Dreamacro/clash)中下载与路由器架构对应的最新Clash文件，解压后重命名为`clash`。
4. Country.mmdb文件，虽然会在运行Clash时自动下载，但是速度不敢恭维，建议先行下载。
5. config.yaml配置文件，根据自己的机场或者VPS设置，如果可行可以在Gitlab利用私有Token实现托管（下面会讲）。

## 安装

1. 利用ssh连接路由器，利用以下命令创建文件夹，并赋予执行权限；

   ```shell
   mkdir /jffs/clash/
   chmod +x /jffs/clash/
   ```

2. Win系统下直接利用Winscp将上述clash、Country.mmdb、config.yaml上传到`/jffs/clash/`文件夹内；

   Mac系统下可以利用以下命令分别上传；

   ```shell
   scp *本地文件路径* *路由器登陆用户名*@*路由器地址*:/jffs/clash/
   ```

   * 使用时，省略*。

3. 赋予clash文件执行权限；

   ```shell
   chmod +x /jffs/clash/clash
   ```

4. 测试运行；

   ```sh ee
   /jffs/clash/clash -d /jffs/clash/
   ```

   如果配置无误，可以看到Clash开始跑并输出log。

## 透明代理

### 透明代理下的config.yaml配置

Merlin直接跑Clash相对于OpenClash和KoolClash无法自动更新托管配置。

所以我们可以将config.yaml文件存放在Gitlab的私有库中，然后利用脚本进行更新，当然也可以增加定时任务进行定时更新。

##### Gitlab的私有库RAW文件规则（Github不支持私有库拉取RAW）

*以下带【】部分为需要自己填写项*

> ```
> https://gitlab.com/api/v4/projects/【Project ID】/repository/files/【config.yaml】/raw?ref=master&private_token=【私有Token】
> ```

Project ID：可在以下位置找到。

<h1 align="left"><img src="https://raw.githubusercontent.com/ZiGmaX809/Clash-For-Merlin/master/Project ID.png" alt="Project ID" width="200"></h1>

config.yaml：无需变更名称，直接在仓库根目录创建，否则下载下来还需要重命名；

私有Token：需要自己申请Token。

### 配置文件编写

```yaml
---
port: 7890  #默认http代理端口
socks-port: 7891  #默认socks代理端口
redir-port: 7892  #默认redir端口
allow-lan: true
mode: Rule
log-level: info
external-controller: 0.0.0.0:9090  #控制台端口
experimental:
  interface-name: eth0  #（尝试）TUN模式下，流量出口，路由器为WAN口
  ignore-resolve-fail: true  #忽略DNS解析失败
  
dns:
  enable: true
  ipv6: false
  listen: 0.0.0.0:53  #监听路由器53端口
  enhanced-mode: fake-ip
  fake-ip-range: 198.168.0.1/16
  nameserver:
    - 114.114.114.114
    - 223.5.5.5
    - 8.8.8.8
    - 1.1.1.1
    
tun:
  enable: true  #（尝试）TUN模式
  
proxy-provider: #使用proxy-provider对不同的规则进行订阅，以Dler为例
  Dler:
    type: http
    path: ./Dler.yaml
    url: https://dler.cloud/subscribe/token?protocols=ss&list=clash&lv=3&noarea=cn
    interval: 3600
    health-check:
        enable: true
        url: http://www.gstatic.com/generate_204
        interval: 300
   
  ...
        
Proxy Group:
  - name: Auto - UrlTest
    type: url-test
    use:
      - Dler
    url: http://www.gstatic.com/generate_204
    interval: "1800"

  - name: Proxy
    type: select
    use:
      - Dler
    proxies:
      - Auto - UrlTest
      - DIRECT

  ...

Rule:
  ...
```

### 重启Clash脚本

创建重启Clash脚本并赋予执行权限；

```shell
touch /jffs/clash/restart.sh
chmod +x /jffs/clash/restart.sh
vi /jffs/clash/restart.sh
```

打开后输入以下内容；

```shell
if $(pidof clash) > /dev/null; then
	kill $(pidof clash)
fi
start-stop-daemon -S -b -x /jffs/clash/clash -m -p /tmp/clash.pid -- -d /jffs/clash/
```

后续直接运行`/jffs/clash/restart.sh`即可重启Clash。

### 更新config.yaml脚本

创建更新config.yaml脚本并赋予执行权限；

```shell
touch /jffs/clash/update.sh
chmod +x /jffs/clash/update.sh
vi /jffs/clash/update.sh
```

打开后输入以下内容；

```shell
wget -O /jffs/clash/config.yaml 'https://gitlab.com/api/v4/projects/【Project ID】/repository/files/【config.yaml】/raw?ref=master&private_token=【私有Token】'
/bin/sh /jffs/clash/restart.sh
```

即为你Gitlab托管地址。

后续直接运行`/jffs/clash/update.sh`即可更新config.yaml配置文件。

### iptables规则

新建sh脚本以自动添加iptables规则，并赋予执行权限，并添加规则；

```shell
touch /jffs/clash/clash-iptable.sh
chmod +x /jffs/clash/clash-iptable.sh
vi /jffs/clash/clash-iptable.sh
```

添加

```shell
#排除ssh端口
iptables -t nat -A PREROUTING -p tcp --dport 22 -j ACCEPT
#在nat表中首行添加排除来源为192.168.2.65规则（尚在尝试）
iptables -t nat -I PREROUTING -s 192.168.2.65/32 -j RETURN
#新建名为CLASH的CHAIN
iptables -t nat -N CLASH
#在CLASH的CHAIN中末行添加排除去向为192.168.0.0/16网段规则
iptables -t nat -A CLASH -d 192.168.0.0/16 -j RETURN 
#在CLASH的CHAIN中末行添加转发所有tcp数据至7892端口
iptables -t nat -A CLASH -p tcp -j REDIRECT --to-ports 7892
#在nat表中添加所有流量转向CLASH的CHAIN
iptables -t nat -A PREROUTING -p tcp -j CLASH
```

其中`iptables -t nat -I PREROUTING -s 192.168.2.65/32 -j RETURN`是跳过`192.168.2.65`IP，因为这是Switch的地址，最近在玩动物之森，需要用到UU加速器。（该功能尚在尝试阶段）

### 更新iptable配置

后续发现一个大神的iptable配置[项目地址](https://github.com/Outofpit/clash-merlin)，可以代理UDP；
稍作修改后可以和UU加速器共存，并且不再是F而是未开代理的B。

* 需要注意的是，需将Switch的网络设置如下：
* ip：192.168.2.100和192.168.2.101；
* 子网掩码：255.255.255.0；
* 网关：192.168.2.1；
* DNS：223.5.5.5；114.114.114.114；
* 或者自定义设置后，修改下面配置中的ip地址，将其排除代理。

```shell
#!/bin/sh
proxy_port=7892
lan_ip=192.168.2.1
ssh_port=22

# ports redirect for clash except port 22 for ssh connection
iptables -t nat -A PREROUTING -p tcp --dport $ssh_port -j ACCEPT

# redirect for clash except Nintendo Switch
iptables -t nat -I PREROUTING -s 192.168.2.100/32 -j RETURN
iptables -t nat -I PREROUTING -s 192.168.2.101/32 -j RETURN

#new
iptables -t nat -N CLASH
iptables -t nat -A CLASH -d 192.168.0.0/16 -j RETURN
iptables -t nat -A CLASH -d 10.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 127.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 169.254.0.0/16 -j RETURN
iptables -t nat -A CLASH -d 172.16.0.0/12 -j RETURN
iptables -t nat -A CLASH -d 224.0.0.0/4 -j RETURN
iptables -t nat -A CLASH -d 240.0.0.0/4 -j RETURN

#redirect to Clash
iptables -t nat -A CLASH -p tcp -j REDIRECT --to-ports $proxy_port
iptables -t nat -A PREROUTING -j CLASH
# fake-ip rules
iptables -t nat -A OUTPUT -p tcp -d 198.168.0.0/16 -j REDIRECT --to-ports $proxy_port

#DNS
#iptables -t nat -A PREROUTING -p udp -m udp --dport 53 -j DNAT --to-destination $lan_ip:55

# udp
modprobe xt_TPROXY
proxy_fwmark="0x162"
proxy_table="0x162"
ip rule add fwmark $proxy_fwmark table $proxy_table
ip route add local default dev lo table $proxy_table
iptables -t mangle -N CLASH
iptables -t mangle -A CLASH -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A CLASH -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A CLASH -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A CLASH -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A CLASH -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A CLASH -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A CLASH -d 240.0.0.0/4 -j RETURN

# redirect to Clash
iptables -t mangle -A CLASH -p udp ! --dport 53 -j TPROXY --on-port $proxy_port --tproxy-mark $proxy_table
iptables -t mangle -A PREROUTING -p udp ! --dport 53 -j CLASH

# redirect for clash except Nintendo Switch
iptables -t mangle -I PREROUTING -s 192.168.2.100/32 -j RETURN
iptables -t mangle -I PREROUTING -s 192.168.2.101/32 -j RETURN

# fake-ip rules
iptables -t mangle -A OUTPUT -p udp ! --dport 53 -d 198.168.0.0/16 -j MARK --set-mark $proxy_fwmark
```

> 待验证
> 在不需要代理的设备中将DNS地址设置为非路由器地址

#### dnsmasq更改端口

因为前面配置文件中Clash监听了DNS服务器的53端口，但是53端口已经被dnsmasq占用，所以要将dnsmasq端口更改或直接关闭。

```shell
touch /jffs/configs/dnsmasq.conf.add
vi /jffs/configs/dnsmasq.conf.add
```

将以下内容添加进去，0为关闭DNS功能，当然也可以更改为5353等端口；

```shell
port=0
```

我这边直接关闭dnsmasq的DNS解析功能，仅留存DHCP功能；

```shell
service restart_dnsmasq
```

重启dnsmasq服务。

### 其他

需要在路由器—内部网络中将DNS地址设置为路由器地址，至此连接至改路由器的所有设备都将会科学上网。

## 开机启动

不知道是AC86U的原因还是命令行位置不对，如果使用`start-stop-daemon`是无法在路由器重启时启动Clash。

所以我是使用`&`来进行后台运行；

```shell
vi /jffs/scripts/wan-start
```

在最后添加

```shell
/bin/sh /jffs/clash/clash-iptable.sh
/jffs/clash/clash -d /jffs/clash/ &
```

## 与UU加速器共存

* 需要将switch中DNS更改为非路由器地址，设备可直连。

尝试：

​	~~联系Merlin开发者，以更新iptable来使用-m owner -pid-owner直接排除UU加速器的端口，但是被以稳定性无法保障为由驳回。~~

通过iptable配置，使用TPROXY已解决。
