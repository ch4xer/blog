---
title: 域名配合隧道穿透服务(Godaddy+Cloudflare+Cpolar/Cloudflared)
author: ch4ser
date: 2024-12-08T17:33:39+08:00
categories:
  - 技术
cover:
  image: cover.jpeg
---

# 记录一次域名购买并使用在隧道服务的经历

## Godaddy 域名购买

其实Godaddy的域名还是偏贵的... 但一年50块钱的top域名还算可以接受就是了, 而且不用像在国内购买域名一样要实名制, 而且支持支付宝, 挺ok的.

## Cloudflare 域名托管

在购买域名完成之后, 我发现Godaddy是不允许为主域名设置CNAME的, 这就有点不能接受, 然而师弟告诉我Cloudflare可以, 遂用其托管.

点击添加域名, 填入在Godaddy购买的域名

![add domain](./adddomain.png)

点击Continue, 等到Cloudflare发现Godaddy下的dns记录, 继续, 看到套餐页面选择免费的套餐就可以了, 之后选择复制Cloudflare给出的ns记录.

来到Godaddy的控制台, 在DNS -> Nameservers 下面选择改变nameservers, 将Cloudflare给出的ns记录复制进去, 保存后等待Cloudflare页面将域名的状态更新为Active, 这样托管就完成了.

![nameservers](./nameservers.png)

## 内网穿透

### Cpolar

将域名解析到Cpolar提供的隧道中, 进入Cpolar的控制面板, 点击自定义域名, 创建一个cname记录.

![cname](./cpolar-cname.png)

然后在Cloudflare的DNS面板中为主域名添加这条CNAME, 这样就可以把主域名解析到cpolar提供的隧道中了.

![Cloudflare cname](./cloudflare-cname.png)

然后申请TLS证书，在Cloudflare的Origin Server中, 点击创建证书, 不得不感叹Cloudflare的良心, 在国内云厂商纷纷对SSL证书收费的情况下, Cloudflare能够提供15年有效期的免费证书, 真的友好.

![ssl](./ssl.png)

申请完毕之后将证书和密钥下载下来, 进入本地的Cpolar服务面板, 新建隧道并上传证书后启动即可, 如图所示.

![ssl cpolar](./ssl-cpolar.png)

另外, 在高级选项中, 似乎HTTP绑定方式必须为同时启动, 并且重定向到HTTPS这一项需要设置为false, 否则会出现404的情况, 不知道为什么.

### Cloudflared

Cpolar的限制非常大，不仅绑定域名的功能是收费的（每年150），而且让自己的重要服务依赖于小厂商也让人不太安心。在一番摸索之后，我发现宇宙良心企业Cloudflare也提供了类似的隧道穿透服务，名为Cloudflared Tunnel。

登陆到Cloudflare之后，选择Zero trust，在付费选项中选择Free Plan，然后在添加信用卡步骤时刷新页面，进入控制台。在Network -> Tunnels下面点击Create Tunnel，输入隧道名称，点击Save。

Tunnels需要在本地安装Cloudflared客户端，官方提供了Docker的安装方式，

```yaml
services:
  tunnel:
    image: cloudflare/cloudflared:latest
    network_mode: host
    command: tunnel --no-autoupdate run --token <TOKEN>
    restart: unless-stopped
```


token的具体内容可以在页面中找到。之后就可以将自己的域名解析到隧道中了，在Public hostnames中添加自己的子域名，然后选择对应的本地服务就完成了，实测下来访问速度也完全够用，薄纱Cpolar!

![tunnels](./tunnels.png)
