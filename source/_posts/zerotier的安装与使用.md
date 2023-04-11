---
title: zerotier的安装与使用
date: 2023-02-27 22:55:49
tags:
---

# zerotier的安装与配置

## 安装

deb/rpm

```bash
curl -s https://install.zerotier.com | sudo bash
```

debian 系会把zerotier的仓库加入到/etc/apt/sources.list里后自动用apt安装

安装完之后需要添加开机自启

```bash
systemctl enable zerotier-one	# 添加服务
systemctl start zerotier-one	# 启动服务
systemctl status zerotier-one	# 查看服务状态
```

arch系安装更加简单，这款软件已经在官方源里了。

```bash
pacman -S zerotier-one
```

windows/mac/ios/android均可以直接下载安装https://www.zerotier.com/download/

### 加入网络

接下来要做的事情是到 [https://my.zerotier.com/](https://link.zhihu.com/?target=https%3A//my.zerotier.com/) 里注册帐号并且登录，然后创建一个Network，创建之后点到这个网络里，拉到最上面，选择 Private(私有网络) ，这样别人加入的时候就需要认证， 如果想改名字的话，改个名字，其他不用动。然后复制 `Network ID`，就是拉到最上面的时候可以看到的一个类似 `3efa5cbd2a3c91e3` 的字符串。

![image-20230227205610163](https://blog-1308084928.cos.ap-shanghai.myqcloud.com/zerotier的安装与使用1.png)

随后在内网机器上执行

```bash
sudo zerotier-cli join 3efa5cbd2a3c91e3	# linux
```

windows 在状态栏打开zerotier-one选择加入网络

![image-20230227205919704](https://blog-1308084928.cos.ap-shanghai.myqcloud.com/zerotier的安装与使用2.png)

加入之后他们就启动了，但是还连不进我们创建的网络，因为我们选择了 Private(私有网络) ，我们还需要到 [https://my.zerotier.com/](https://link.zhihu.com/?target=https%3A//my.zerotier.com/) 上面对接入的机器打勾，拉到 Members 这一节，把前面的两个勾勾选上。(有一定延迟，等个20几秒才会出现)

这个时候执行一下 `ip a` 你会发现多了一个叫做 `ztuzethlza` 或者类似名字的设备，还有IP地址，这就是zerotier组建的局域网的IP 地址，但是这个时候你如果直接连接另外一台机器的话可能会非常慢，所以为了加速，我们还需要一台在国内的机器做转发。

## 搭建中转机器(moon)

如果在创建Moon 节点前机器还没有加入网络的话，创建moon节点后加入的机器就不需要额外的配置了，个人更推荐这样做。

首先执行

```bash
cd /var/lib/zerotier-one/
sudo zerotier-idtool initmoon identity.public > moon.json
```

接下来编辑一下 `moon.json`，把 `"stableEndpoints": []` 这一节里加入中转机器的公网IP，例如 `"stableEndpoints": ["1.2.3.4/9993"]`，其中 9993是默认监听的端口，接下来要把9993端口的防火墙放开(注意是UDP)，如果你的机器外边还有防火墙的话，也要一起放开，例如阿里云的机器就有 防火墙规则，要一起把对应端口的UDP流量放行，此后，我们要生成moon的配置：

```bash
sudo ufw allow 9993/udp	# 放行端口
sudo zerotier-idtool genmoon moon.json	# 生成moon节点
```

上一步会生成一个000000deadbeef00.moon类型的文件，它是由json文件里的secret签名得来。你需要把他放到/var/lib/zerotier-one/moons.d/文件夹里面，如果没有该文件夹就创一个。

接下来把中转服务器的 `zerotier-one` 重启：

```bash
sudo systemctl restart zerotier-one
```

## 加入moon
注意我们搭建moon的时候有一个 moon.json，我们把内网机器加入这个moon的时候，需要其中的一个id：

```bash 
grep id /var/lib/zerotier-one/moon.json | head -n 1
"id": "xxxxxxxxxx",
```


复制这个id，然后在内网机器执行：

```bash
sudo zerotier-cli orbit xxxxxxxxxx xxxxxxxxxx
```

成功加入后应该是下面这个样子

![image-20230227212522035](https://blog-1308084928.cos.ap-shanghai.myqcloud.com/zerotier的安装与使用3.png)



You can add these roots to regular nodes in one of two ways: by placing the same world definition file in their `moons.d` directories or by using the `zerotier-cli orbit` command: `zerotier-cli orbit deadbeef00 deadbeef00`. The first argument is the world ID (which we can shorten by removing the two leading zeroes) and the second is the address of any of its roots. This will contact the root and obtain the full world definition from it if it’s online and reachable.

Once you’ve “orbited” your moon, try `zerotier-cli listpeers`. You should see the roots you’ve created listed as `MOON` instead of `LEAF`. They will now be used as alternative root servers.

可以用ping来测试网络连通性，但是注意如果ping的目标是windows时注意防火墙。在同一局域网下网络还是很稳定的

![image-20230227212822115](https://blog-1308084928.cos.ap-shanghai.myqcloud.com/zerotier的安装与使用4.png)

## 常用操作

```bash
zerotier-cli join xxxxxxxxxx	# 加入网络 xxxxxxxxxx
zerotier-cli listpeers	# 列出节点，包括moon
```
