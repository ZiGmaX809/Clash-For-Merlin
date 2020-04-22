# Clash For Merlin记录

> 设备：华硕AC-86U
>
> 固件：KoolShare改版Merlin 384.16
>
> Clash版本：0.19

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

## config.yaml配置

Merlin直接跑Clash相对于OpenClash和KoolClash无法自动更新托管配置。

所以我们可以将config.yaml文件存放在Gitlab的私有库中，然后利用脚本进行更新，当然也可以增加定时任务进行定时更新。

##### Gitlab的私有库RAW文件规则（Github不支持私有库拉取RAW）

*以下带【】部分为需要自己填写项*

> ```
> https://gitlab.com/api/v4/projects/【Project ID】/repository/files/【config.yaml】/raw?ref=master&private_token=【私有Token】
> ```

需要说明的是：

Project ID：是你项目的ID，可在下图位置找到


## 透明代理

Clash的透明代理需要搭配iptables使用。

## 开机启动

## 手动更新config.yaml脚本

## 重启脚本

## 与UU加速器共存（待验证）


