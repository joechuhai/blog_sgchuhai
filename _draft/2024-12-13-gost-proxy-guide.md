---
title: 如何快速申请企业邓白氏编码（D-U-N-S Number）
date: 2024-12-13 15:40:00 +0800
categories: [业务指南, 商业认证]    
tags: [邓氏编码, D-U-N-S Number, 企业注册, Dun & Bradstreet, 申请指南, 商业信息, 开发者账号, 商业认证]
description: 提供方便快捷的企业邓白氏编码注册指南
image: assets/img/preview/DUNS.jpg
---



## 前言





## 一、VPS选购及实例创建

笔者选择的VPS提供商：AWS lightSail

VPS规格：最低配置即可，如1个CPU，1GB内存，50GB硬盘，流量按需选择；

创建过程中，手动把“默认密钥”下载到本地，后续用PuTTY等远程终端登录器访问服务器时需要；

![](https://image.sgchuhai.com/image/2024/8b13c8a736104b82824a470f9ce5d57a.png)

![](https://image.sgchuhai.com/image/2024/2aca7fa9b4a2feb85484a64437864b29.png)

点击上述“创建实例”，等待后台完成实例创建。

## 二、VPS实例远程连接

LightSail平台有提供浏览器版本ssh访问方式，但是操作不够友好。这里更建议使用自己熟悉的终端工具，来远程访问实例。

![](https://image.sgchuhai.com/image/2024/b03e908fa7cee34369bc726473c01c87.png)

本文以PC端为例，主要使用PuTTY工具来访问。PuTTY官网下载地址[点击此处](https://www.putty.org/)；

![](https://image.sgchuhai.com/image/2024/b8d28080c41b8fe14043ddc1fb265d66.jpg)

下载安装完成PuTTY后，会发现有如下几个工具：

- PuTTY：远程服务器链接工具
- PuTTYgen：密钥生成、转换工具
- PSFTP：上传文件至远程服务器

### 2.1 密钥格式转换

先打开PuTTYgen，点击“Conversions > import”，选择此前服务器的私钥（.pem），转换成特定的格式（.ppk）。

![](https://image.sgchuhai.com/image/2024/f9a5b3ea0b58be06d8413cd78aed3ee4.png)

### 2.2 登录信息配置

打开PuTTY，在“Session > Host Name”中，输入实例的公网IP（**公有 IPv4 地址**），其他信息保持默认；

![](https://image.sgchuhai.com/image/2024/3295b872d62933d554d65c59a5f4b2e5.png)

此外，依次点击“Connection > SSH > Auth > Credentials”，选择此前转换好的“.ppk”密钥；

![](https://image.sgchuhai.com/image/2024/de5a717853ab8291f74d0dac82bc1e4c.png)

完成后，点击右下角“Open”，将会唤起终端窗口，输入用户名，等待片刻，应该就能登录成功；

如下为一个登录成功的示例；

![](https://image.sgchuhai.com/image/2024/0c433375e8c7e054227144da68cf3652.png)

更完整的连接方式参考[使用 PuTTY 连接到 Linux 实例](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/connect-linux-inst-from-windows.html)。

## 三、域名申请及配置

域名购买：选择任意一家域名提供商，例如GoDaddy（海外版）、Namecheap（个人推荐），购买一个域名；

切换DNS解析：将该域名的DNS解析，指向CloudFlare。注册完CloudFlare后，平台会指引你如何完成操作；

添加子域名解析：在CloudFlare上，创建一个子域名，解析到VPS公网IP，代理状态设置为“仅DNS”；

![](https://image.sgchuhai.com/image/2024/0935370578e8eb328b907bc52f0bfbcd.png)

域名证书配置：为实现https访问加密效果，我们需要为域名申请加密证书，后续的部署代码会提供let's encrypt的证书申请指引；

## 四、Gost代理服务搭建

本文使用的是博主提供的一键搭建脚本。

使用PuTTY连接远程服务器，在终端窗口，输入“vim gost_start.sh”，创建并编辑脚本；

复制如下[一键搭建脚本代码（原作者：@codepiano）](https://github.com/haoel/haoel.github.io/blob/master/scripts/install.ubuntu.18.04.sh)，粘贴到终端窗口（鼠标右键粘贴）；

```shell
#!/bin/bash

# Author
# original author:https://github.com/gongzili456
# modified by:https://github.com/haoel

# Ubuntu 18.04 系统环境

COLOR_ERROR="\e[38;5;198m"
COLOR_NONE="\e[0m"
COLOR_SUCC="\e[92m"

update_core(){
    echo -e "${COLOR_ERROR}当前系统内核版本太低 <$VERSION_CURR>,需要更新系统内核.${COLOR_NONE}"
    sudo apt install -y -qq --install-recommends linux-generic-hwe-18.04
    sudo apt autoremove

    echo -e "${COLOR_SUCC}内核更新完成,重新启动机器...${COLOR_NONE}"
    sudo reboot
}

check_bbr(){
    has_bbr=$(lsmod | grep bbr)

    # 如果已经发现 bbr 进程
    if [ -n "$has_bbr" ] ;then
        echo -e "${COLOR_SUCC}TCP BBR 拥塞控制算法已经启动${COLOR_NONE}"
    else
        start_bbr
    fi
}

start_bbr(){
    echo "启动 TCP BBR 拥塞控制算法"
    sudo modprobe tcp_bbr
    echo "tcp_bbr" | sudo tee --append /etc/modules-load.d/modules.conf
    echo "net.core.default_qdisc=fq" | sudo tee --append /etc/sysctl.conf
    echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee --append /etc/sysctl.conf
    sudo sysctl -p
    sysctl net.ipv4.tcp_available_congestion_control
    sysctl net.ipv4.tcp_congestion_control
}

install_bbr() {
    # 如果内核版本号满足最小要求
    if [[ $VERSION_CURR > $VERSION_MIN ]]; then
        check_bbr
    else
        update_core
    fi
}

install_docker() {
    if ! [ -x "$(command -v docker)" ]; then
        echo "开始安装 Docker CE"
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        sudo add-apt-repository \
            "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
            $(lsb_release -cs) \
            stable"
        sudo apt-get update -qq
        sudo apt-get install -y docker-ce
    else
        echo -e "${COLOR_SUCC}Docker CE 已经安装成功了${COLOR_NONE}"
    fi
}


check_container(){
    has_container=$(sudo docker ps --format "{{.Names}}" | grep "$1")

    # test 命令规范： 0 为 true, 1 为 false, >1 为 error
    if [ -n "$has_container" ] ;then
        return 0
    else
        return 1
    fi
}

install_certbot() {
    echo "开始安装 certbot 命令行工具"
    sudo apt-get update -qq
    sudo apt-get install -y software-properties-common
    sudo add-apt-repository universe
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt-get update -qq
    sudo apt-get install -y certbot
}

create_cert() {
    if ! [ -x "$(command -v certbot)" ]; then
        install_certbot
    fi

    echo "开始生成 SSL 证书"
    echo -e "${COLOR_ERROR}注意：生成证书前,需要将域名指向一个有效的 IP,否则无法创建证书.${COLOR_NONE}"
    read -r -p "是否已经将域名指向了 IP？[Y/n]" has_record

    if ! [[ "$has_record" = "Y" ]] ;then
        echo "请操作完成后再继续."
        return
    fi

    read -r -p "请输入你要使用的域名:" domain

    sudo certbot certonly --standalone -d "${domain}"
}

install_gost() {
    if ! [ -x "$(command -v docker)" ]; then
        echo -e "${COLOR_ERROR}未发现Docker，请求安装 Docker ! ${COLOR_NONE}"
        return
    fi

    if check_container gost ; then
        echo -e "${COLOR_ERROR}Gost 容器已经在运行了，你可以手动停止容器，并删除容器，然后再执行本命令来重新安装 Gost。 ${COLOR_NONE}"
        return
    fi

    echo "准备启动 Gost 代理程序,为了安全,需要使用用户名与密码进行认证."
    read -r -p "请输入你要使用的域名：" DOMAIN
    read -r -p "请输入你要使用的用户名:" USER
    read -r -p "请输入你要使用的密码:" PASS
    read -r -p "请输入HTTP/2需要侦听的端口号(443)：" PORT

    if [[ -z "${PORT// }" ]] || ! [[ "${PORT}" =~ ^[0-9]+$ ]] || ! { [ "$PORT" -ge 1 ] && [ "$PORT" -le 65535 ]; }; then
        echo -e "${COLOR_ERROR}非法端口,使用默认端口 443 !${COLOR_NONE}"
        PORT=443
    fi

    BIND_IP=0.0.0.0
    CERT_DIR=/etc/letsencrypt
    CERT=${CERT_DIR}/live/${DOMAIN}/fullchain.pem
    KEY=${CERT_DIR}/live/${DOMAIN}/privkey.pem

    sudo docker run -d --name gost \
        -v ${CERT_DIR}:${CERT_DIR}:ro \
        --net=host ginuerzh/gost \
        -L "http2://${USER}:${PASS}@${BIND_IP}:${PORT}?cert=${CERT}&key=${KEY}&probe_resist=code:400&knock=www.google.com"
}

crontab_exists() {
    crontab -l 2>/dev/null | grep "$1" >/dev/null 2>/dev/null
}

create_cron_job(){
    # 写入前先检查，避免重复任务。
    if ! crontab_exists "certbot renew --force-renewal"; then
        echo "0 0 1 * * /usr/bin/certbot renew --force-renewal" >> /var/spool/cron/crontabs/root
        echo "${COLOR_SUCC}成功安装证书renew定时作业！${COLOR_NONE}"
    else
        echo "${COLOR_SUCC}证书renew定时作业已经安装过！${COLOR_NONE}"
    fi

    if ! crontab_exists "docker restart gost"; then
        echo "5 0 1 * * /usr/bin/docker restart gost" >> /var/spool/cron/crontabs/root
        echo "${COLOR_SUCC}成功安装gost更新证书定时作业！${COLOR_NONE}"
    else
        echo "${COLOR_SUCC}gost更新证书定时作业已经成功安装过！${COLOR_NONE}"
    fi
}

install_shadowsocks(){
    if ! [ -x "$(command -v docker)" ]; then
        echo -e "${COLOR_ERROR}未发现Docker，请求安装 Docker ! ${COLOR_NONE}"
        return
    fi

    if check_container ss ; then
        echo -e "${COLOR_ERROR}ShadowSocks 容器已经在运行了，你可以手动停止容器，并删除容器，然后再执行本命令来重新安装 ShadowSocks。${COLOR_NONE}"
        return
    fi

    echo "准备启动 ShadowSocks 代理程序,为了安全,需要使用用户名与密码进行认证."
    read -r -p "请输入你要使用的密码:" PASS
    read -r -p "请输入ShadowSocks需要侦听的端口号(1984)：" PORT

    BIND_IP=0.0.0.0

    if [[ -z "${PORT// }" ]] || ! [[ "${PORT}" =~ ^[0-9]+$ ]] || ! { [ "$PORT" -ge 1 ] && [ "$PORT" -le 65535 ]; }; then
        echo -e "${COLOR_ERROR}非法端口,使用默认端口 1984 !${COLOR_NONE}"
        PORT=1984
    fi

    sudo docker run -dt --name ss \
        -p "${PORT}:${PORT}" mritd/shadowsocks \
        -s "-s ${BIND_IP} -p ${PORT} -m aes-256-cfb -k ${PASS} --fast-open"
}

install_vpn(){
    if ! [ -x "$(command -v docker)" ]; then
        echo -e "${COLOR_ERROR}未发现Docker，请求安装 Docker ! ${COLOR_NONE}"
        return
    fi

    if check_container vpn ; then
        echo -e "${COLOR_ERROR}VPN 容器已经在运行了，你可以手动停止容器，并删除容器，然后再执行本命令来重新安装 VPN。${COLOR_NONE}"
        return
    fi

    echo "准备启动 VPN/L2TP 代理程序,为了安全,需要使用用户名与密码进行认证."
    read -r -p "请输入你要使用的用户名:" USER
    read -r -p "请输入你要使用的密码:" PASS
    read -r -p "请输入你要使用的PSK Key:" PSK

    sudo docker run -d --name vpn --privileged \
        -e PSK="${PSK}" \
        -e USERNAME="${USER}" -e PASSWORD="${PASS}" \
        -p 500:500/udp \
        -p 4500:4500/udp \
        -p 1701:1701/tcp \
        -p 1194:1194/udp  \
        siomiz/softethervpn
}

install_brook(){
    brook_file="/usr/local/brook/brook"
    [[ -e ${brook_file} ]] && echo -e "${COLOR_ERROR}Brook 已经安装，请检查!" && return
    wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/brook.sh &&\
        chmod +x brook.sh && sudo bash brook.sh
}

# TODO: install v2ray

init(){
    VERSION_CURR=$(uname -r | awk -F '-' '{print $1}')
    VERSION_MIN="4.9.0"

    OIFS=$IFS  # Save the current IFS (Internal Field Separator)
    IFS=','    # New IFS

    COLUMNS=50
    echo -e "\n菜单选项\n"

    while true
    do
        PS3="Please select a option:"
        re='^[0-9]+$'
        select opt in "安装 TCP BBR 拥塞控制算法" \
                    "安装 Docker 服务程序" \
                    "创建 SSL 证书" \
                    "安装 Gost HTTP/2 代理服务" \
                    "安装 ShadowSocks 代理服务" \
                    "安装 VPN/L2TP 服务" \
                    "安装 Brook 代理服务" \
                    "创建证书更新 CronJob" \
                    "退出" ; do

            if ! [[ $REPLY =~ $re ]] ; then
                echo -e "${COLOR_ERROR}Invalid option. Please input a number.${COLOR_NONE}"
                break;
            elif (( REPLY == 1 )) ; then
                install_bbr
                break;
            elif (( REPLY == 2 )) ; then
                install_docker
                break
            elif (( REPLY == 3 )) ; then
                create_cert
                #loop=1
                break
            elif (( REPLY == 4 )) ; then
                install_gost
                break
            elif (( REPLY == 5  )) ; then
                install_shadowsocks
                break
            elif (( REPLY == 6 )) ; then
                install_vpn
                break
            elif (( REPLY == 7 )) ; then
                install_brook
                break
            elif (( REPLY == 8 )) ; then
                create_cron_job
                break
            elif (( REPLY == 9 )) ; then
                exit
            else
                echo -e "${COLOR_ERROR}Invalid option. Try another one.${COLOR_NONE}"
            fi
        done
    done

    echo "${opt}"
    IFS=$OIFS  # Restore the IFS
}

init
```

![](https://image.sgchuhai.com/image/2024/fcffceeb1fa7868a7292a92a5e89ae84.png)

粘贴完毕后，按下“ESC”键退出编辑模式，输入`:wq!`保存并退出；

接下来，输入`chmod +x gost_start.sh`，为脚本添加执行权限；

终端命令行输入`sudo ./gost_start.sh`，运行脚本；

![](https://image.sgchuhai.com/image/2024/504dfb19dd81b0a414231877b534f6be.png)

按照如下顺序依次安装：

1. 安装Docker服务程序

2. 安装TCP BBR 拥塞控制算法

3. 创建SSL证书

   ![](https://image.sgchuhai.com/image/2024/e4751e11572563192d0d4e305ef6203d.png)

4. 创建证书更新 CronJob

   由于let's encrypt证书有一定有效期，因此需要定期重新申请；

   ![](https://image.sgchuhai.com/image/2024/2eb0b5d7622af7dff25273decb80196b.png)

5. 搭建Gost HTTP代理服务

   ![](https://image.sgchuhai.com/image/2024/79880d5803406d3af9dd872e9d681017.png)

   针对访问端口，使用默认的443即可，使用其他的也行；但是确保VPS后台的防火墙有开放对应端口。

   ![](https://image.sgchuhai.com/image/2024/26f11ab5286b9bb3cdc81fb60648c2c1.png)

至此，服务搭建完成，命令行输入如下指令，检查代理是否成功。

`curl -v "https://www.google.com" --proxy "https://DOMAIN" --proxy-user 'USER:PASS'`

如下是正常的返回内容：

![](https://image.sgchuhai.com/image/2024/596758361cf00dba9920a8de5f39c0bd.png)

如果你的服务运行不正常，可使用如下命令来详细排查：

- `sudo docker ps` 查看 gost 是否在运行
- `netstat -nolp | grep 443` 查看 gost 是否在监听 443 端口
- `sudo docker logs gost` 查看 gost 的日志
- `find / -name "certbot" ` 查看是否正常安装 Certbot （证书下发）
- `certbot certificates` 查看证书剩余有效期（需要进入程序目录运行）

## 五、代理服务使用

### 5.1 代理应用逻辑

上述搭建的HTTP代理服务，一般有2种使用逻辑：

- 逻辑一：直接在相关应用场景的代理配置栏，配置http代理信息，配置内容包括：

  代理协议 （https）、代理服务器（代理域名）、代理端口 （443，如有其他可按实际填）、用户名、密码

- 逻辑二：将上述HTTP配置信息，转换成clash等通用的规则配置，导入clash、shadowrockt等代理工具；

### 5.2 代理场景及工具推荐

- PC浏览器：Chrome 插件 [SwitchyOmega](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif)

  ![](https://image.sgchuhai.com/image/2024/1aa81e2b98381bc89f4bd6c5e3cd338b.png)

  注：由于开启了嗅探，代理开启后要先访问“`knock` 参数”（本示例为www.google.com），才能生效；

- PC端全局/规则代理：Clash For Windows，需要自己写规则

- iOS端：Shadowrockt，直接配置HTTPS代理 or 导入Clash 规则

- Android端：Clash For Android，导入配置规则使用

- 其他不可自由配置的设备：软路由中转流量

更完整的代理工具参考[此链接@Hackl0us](https://github.com/Hackl0us/SS-Rule-Snippet?tab=readme-ov-file#%E5%B8%B8%E7%94%A8%E4%BB%A3%E7%90%86%E5%B7%A5%E5%85%B7)。

| 工具                                                         | 适用平台                   | 售价            |
| ------------------------------------------------------------ | -------------------------- | --------------- |
| [Surge](https://nssurge.com/)                                | 📱 iOS / ⬛️ iPadOS / 💻 macOS | $49.99 - $99.99 |
| [Quantumult X](https://apps.apple.com/us/app/quantumult-x/id1443988620) | 📱 iOS / ⬛️ iPadOS           | $7.99           |
| [Shadowrocket](https://apps.apple.com/us/app/shadowrocket/id932747118) | 📱 iOS / ⬛️ iPadOS           | $2.99           |
| [clash](https://github.com/Dreamacro/clash)                  | 🧬 Multiple                 | 免费，🎖 开源    |
| [clash Premium](https://github.com/Dreamacro/clash/releases/tag/premium) | 🧬 Multiple                 | 免费，⭕️ 闭源    |
| [Clash for Windows](https://github.com/Fndroid/clash_for_windows_pkg) | 🪟 Windows                  | 免费，⭕️ 闭源    |
| [Clash X](https://github.com/yichengchen/clashX/)            | 💻 macOS                    | 免费，🎖 开源    |
| [Clash X Pro](https://install.appcenter.ms/users/clashx/apps/clashx-pro/distribution_groups/public) | 💻 macOS (Intel)            | 免费，⭕️ 闭源    |
| [Clash X Pro - Apple Silicon](https://install.appcenter.ms/users/clashx/apps/cxp-applesilicon/distribution_groups/public) | 💻 macOS (Apple M1)         | 免费，⭕️ 闭源    |
| [Clash for Android](https://github.com/Kr328/ClashForAndroid) | 🤖️ Android                  | 免费，⭕️ 闭源    |
| [OpenClash](https://github.com/vernesong/OpenClash)          | 📶 OpenWRT                  | 免费，⭕️ 闭源    |

### 5.3 Clash 规则快速生成

使用clash的优势是能基于配置规则，自定义访问站点的代理，一个基础的规则是“国内能直连的就直连，不能直连的走代理”，更详细的规则暂不讨论；

如下是一个示例，复制内容，按需编辑节点服务器（proxies）、节点群组（proxy-groups），保存成`.yaml`格式后导入代理工具即可；

```yaml
mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: Rule
log-level: info
external-controller: '127.0.0.1:9090'

dns:
  enable: true
  listen: 0.0.0.0:53
  #enhanced-mode: redir-host
  nameserver:
    - 223.5.5.5
    - 180.76.76.76
    - 8.8.8.8
    - 1.1.1.1

# 修改如下即可，演示了2个节点服务器，可以按需删除
proxies:
  - name: "自定义代理服务器 1"
    type: http
    server: 自定义代理服务器1
    port: 443
    username: [用户名]
    password: "用户密码"
    tls: true # https
    skip-cert-verify: true  

  - name: "自定义代理服务器 2"
    type: http
    server: 自定义代理服务器2
    port: 443
    username: 用户名
    password: "用户密码"
    tls: true # https
    skip-cert-verify: true  


proxy-groups:
  - name: Proxy #代理服务器群组名称，建议不要修改
    type: select
    proxies:
      - 自定义代理服务器 1
      - 自定义代理服务器 2 #可按需删除

#可前往 https://cdn.jsdelivr.net/gh/Hackl0us/SS-Rule-Snippet@master/LAZY_RULES/clash.yaml 查看模块最新的内容，粘贴即可
rules: 
  # 自定义规则
  ## 您可以在此处插入您补充的自定义规则（请注意保持缩进）

  # Apple
  - DOMAIN,safebrowsing.urlsec.qq.com,DIRECT # 如果您并不信任此服务提供商或防止其下载消耗过多带宽资源，可以进入 Safari 设置，关闭 Fraudulent Website Warning 功能，并使用 REJECT 策略。
  - DOMAIN,safebrowsing.googleapis.com,DIRECT # 如果您并不信任此服务提供商或防止其下载消耗过多带宽资源，可以进入 Safari 设置，关闭 Fraudulent Website Warning 功能，并使用 REJECT 策略。
  - DOMAIN,developer.apple.com,Proxy
  - DOMAIN-SUFFIX,digicert.com,Proxy
  - DOMAIN,ocsp.apple.com,Proxy
  - DOMAIN,ocsp.comodoca.com,Proxy
  - DOMAIN,ocsp.usertrust.com,Proxy
  - DOMAIN,ocsp.sectigo.com,Proxy
  - DOMAIN,ocsp.verisign.net,Proxy
  - DOMAIN-SUFFIX,apple-dns.net,Proxy
  - DOMAIN,testflight.apple.com,Proxy
  - DOMAIN,sandbox.itunes.apple.com,Proxy
  - DOMAIN,itunes.apple.com,Proxy
  - DOMAIN-SUFFIX,apps.apple.com,Proxy
  - DOMAIN-SUFFIX,blobstore.apple.com,Proxy
  - DOMAIN,cvws.icloud-content.com,Proxy
  - DOMAIN-SUFFIX,mzstatic.com,DIRECT
  - DOMAIN-SUFFIX,itunes.apple.com,DIRECT
  - DOMAIN-SUFFIX,icloud.com,DIRECT
  - DOMAIN-SUFFIX,icloud-content.com,DIRECT
  - DOMAIN-SUFFIX,me.com,DIRECT
  - DOMAIN-SUFFIX,aaplimg.com,DIRECT
  - DOMAIN-SUFFIX,cdn20.com,DIRECT
  - DOMAIN-SUFFIX,cdn-apple.com,DIRECT
  - DOMAIN-SUFFIX,akadns.net,DIRECT
  - DOMAIN-SUFFIX,akamaiedge.net,DIRECT
  - DOMAIN-SUFFIX,edgekey.net,DIRECT
  - DOMAIN-SUFFIX,mwcloudcdn.com,DIRECT
  - DOMAIN-SUFFIX,mwcname.com,DIRECT
  - DOMAIN-SUFFIX,apple.com,DIRECT
  - DOMAIN-SUFFIX,apple-cloudkit.com,DIRECT
  - DOMAIN-SUFFIX,apple-mapkit.com,DIRECT
  # - DOMAIN,e.crashlytics.com,REJECT //注释此选项有助于大多数App开发者分析崩溃信息；如果您拒绝一切崩溃数据统计、搜集，请取消 # 注释。

  # 国内网站
  - DOMAIN-SUFFIX,cn,DIRECT
  - DOMAIN-KEYWORD,-cn,DIRECT

  - DOMAIN-SUFFIX,126.com,DIRECT
  - DOMAIN-SUFFIX,126.net,DIRECT
  - DOMAIN-SUFFIX,127.net,DIRECT
  - DOMAIN-SUFFIX,163.com,DIRECT
  - DOMAIN-SUFFIX,360buyimg.com,DIRECT
  - DOMAIN-SUFFIX,36kr.com,DIRECT
  - DOMAIN-SUFFIX,acfun.tv,DIRECT
  - DOMAIN-SUFFIX,air-matters.com,DIRECT
  - DOMAIN-SUFFIX,aixifan.com,DIRECT
  - DOMAIN-KEYWORD,alicdn,DIRECT
  - DOMAIN-KEYWORD,alipay,DIRECT
  - DOMAIN-KEYWORD,taobao,DIRECT
  - DOMAIN-SUFFIX,amap.com,DIRECT
  - DOMAIN-SUFFIX,autonavi.com,DIRECT
  - DOMAIN-KEYWORD,baidu,DIRECT
  - DOMAIN-SUFFIX,bdimg.com,DIRECT
  - DOMAIN-SUFFIX,bdstatic.com,DIRECT
  - DOMAIN-SUFFIX,bilibili.com,DIRECT
  - DOMAIN-SUFFIX,bilivideo.com,DIRECT
  - DOMAIN-SUFFIX,caiyunapp.com,DIRECT
  - DOMAIN-SUFFIX,clouddn.com,DIRECT
  - DOMAIN-SUFFIX,cnbeta.com,DIRECT
  - DOMAIN-SUFFIX,cnbetacdn.com,DIRECT
  - DOMAIN-SUFFIX,cootekservice.com,DIRECT
  - DOMAIN-SUFFIX,csdn.net,DIRECT
  - DOMAIN-SUFFIX,ctrip.com,DIRECT
  - DOMAIN-SUFFIX,dgtle.com,DIRECT
  - DOMAIN-SUFFIX,dianping.com,DIRECT
  - DOMAIN-SUFFIX,douban.com,DIRECT
  - DOMAIN-SUFFIX,doubanio.com,DIRECT
  - DOMAIN-SUFFIX,duokan.com,DIRECT
  - DOMAIN-SUFFIX,easou.com,DIRECT
  - DOMAIN-SUFFIX,ele.me,DIRECT
  - DOMAIN-SUFFIX,feng.com,DIRECT
  - DOMAIN-SUFFIX,fir.im,DIRECT
  - DOMAIN-SUFFIX,frdic.com,DIRECT
  - DOMAIN-SUFFIX,g-cores.com,DIRECT
  - DOMAIN-SUFFIX,godic.net,DIRECT
  - DOMAIN-SUFFIX,gtimg.com,DIRECT
  - DOMAIN,cdn.hockeyapp.net,DIRECT
  - DOMAIN-SUFFIX,hongxiu.com,DIRECT
  - DOMAIN-SUFFIX,hxcdn.net,DIRECT
  - DOMAIN-SUFFIX,iciba.com,DIRECT
  - DOMAIN-SUFFIX,ifeng.com,DIRECT
  - DOMAIN-SUFFIX,ifengimg.com,DIRECT
  - DOMAIN-SUFFIX,ipip.net,DIRECT
  - DOMAIN-SUFFIX,iqiyi.com,DIRECT
  - DOMAIN-SUFFIX,jd.com,DIRECT
  - DOMAIN-SUFFIX,jianshu.com,DIRECT
  - DOMAIN-SUFFIX,knewone.com,DIRECT
  - DOMAIN-SUFFIX,le.com,DIRECT
  - DOMAIN-SUFFIX,lecloud.com,DIRECT
  - DOMAIN-SUFFIX,lemicp.com,DIRECT
  - DOMAIN-SUFFIX,licdn.com,DIRECT
  - DOMAIN-SUFFIX,luoo.net,DIRECT
  - DOMAIN-SUFFIX,meituan.com,DIRECT
  - DOMAIN-SUFFIX,meituan.net,DIRECT
  - DOMAIN-SUFFIX,mi.com,DIRECT
  - DOMAIN-SUFFIX,miaopai.com,DIRECT
  - DOMAIN-SUFFIX,microsoft.com,DIRECT
  - DOMAIN-SUFFIX,microsoftonline.com,DIRECT
  - DOMAIN-SUFFIX,miui.com,DIRECT
  - DOMAIN-SUFFIX,miwifi.com,DIRECT
  - DOMAIN-SUFFIX,mob.com,DIRECT
  - DOMAIN-SUFFIX,netease.com,DIRECT
  - DOMAIN-SUFFIX,office.com,DIRECT
  - DOMAIN-SUFFIX,office365.com,DIRECT
  - DOMAIN-KEYWORD,officecdn,DIRECT
  - DOMAIN-SUFFIX,oschina.net,DIRECT
  - DOMAIN-SUFFIX,ppsimg.com,DIRECT
  - DOMAIN-SUFFIX,pstatp.com,DIRECT
  - DOMAIN-SUFFIX,qcloud.com,DIRECT
  - DOMAIN-SUFFIX,qdaily.com,DIRECT
  - DOMAIN-SUFFIX,qdmm.com,DIRECT
  - DOMAIN-SUFFIX,qhimg.com,DIRECT
  - DOMAIN-SUFFIX,qhres.com,DIRECT
  - DOMAIN-SUFFIX,qidian.com,DIRECT
  - DOMAIN-SUFFIX,qihucdn.com,DIRECT
  - DOMAIN-SUFFIX,qiniu.com,DIRECT
  - DOMAIN-SUFFIX,qiniucdn.com,DIRECT
  - DOMAIN-SUFFIX,qiyipic.com,DIRECT
  - DOMAIN-SUFFIX,qq.com,DIRECT
  - DOMAIN-SUFFIX,qqurl.com,DIRECT
  - DOMAIN-SUFFIX,rarbg.to,DIRECT
  - DOMAIN-SUFFIX,ruguoapp.com,DIRECT
  - DOMAIN-SUFFIX,segmentfault.com,DIRECT
  - DOMAIN-SUFFIX,sinaapp.com,DIRECT
  - DOMAIN-SUFFIX,smzdm.com,DIRECT
  - DOMAIN-SUFFIX,snapdrop.net,DIRECT
  - DOMAIN-SUFFIX,sogou.com,DIRECT
  - DOMAIN-SUFFIX,sogoucdn.com,DIRECT
  - DOMAIN-SUFFIX,sohu.com,DIRECT
  - DOMAIN-SUFFIX,soku.com,DIRECT
  - DOMAIN-SUFFIX,speedtest.net,DIRECT
  - DOMAIN-SUFFIX,sspai.com,DIRECT
  - DOMAIN-SUFFIX,suning.com,DIRECT
  - DOMAIN-SUFFIX,taobao.com,DIRECT
  - DOMAIN-SUFFIX,tencent.com,DIRECT
  - DOMAIN-SUFFIX,tenpay.com,DIRECT
  - DOMAIN-SUFFIX,tianyancha.com,DIRECT
  - DOMAIN-SUFFIX,tmall.com,DIRECT
  - DOMAIN-SUFFIX,tudou.com,DIRECT
  - DOMAIN-SUFFIX,umetrip.com,DIRECT
  - DOMAIN-SUFFIX,upaiyun.com,DIRECT
  - DOMAIN-SUFFIX,upyun.com,DIRECT
  - DOMAIN-SUFFIX,veryzhun.com,DIRECT
  - DOMAIN-SUFFIX,weather.com,DIRECT
  - DOMAIN-SUFFIX,weibo.com,DIRECT
  - DOMAIN-SUFFIX,xiami.com,DIRECT
  - DOMAIN-SUFFIX,xiami.net,DIRECT
  - DOMAIN-SUFFIX,xiaomicp.com,DIRECT
  - DOMAIN-SUFFIX,ximalaya.com,DIRECT
  - DOMAIN-SUFFIX,xmcdn.com,DIRECT
  - DOMAIN-SUFFIX,xunlei.com,DIRECT
  - DOMAIN-SUFFIX,yhd.com,DIRECT
  - DOMAIN-SUFFIX,yihaodianimg.com,DIRECT
  - DOMAIN-SUFFIX,yinxiang.com,DIRECT
  - DOMAIN-SUFFIX,ykimg.com,DIRECT
  - DOMAIN-SUFFIX,youdao.com,DIRECT
  - DOMAIN-SUFFIX,youku.com,DIRECT
  - DOMAIN-SUFFIX,zealer.com,DIRECT
  - DOMAIN-SUFFIX,zhihu.com,DIRECT
  - DOMAIN-SUFFIX,zhimg.com,DIRECT
  - DOMAIN-SUFFIX,zimuzu.tv,DIRECT
  - DOMAIN-SUFFIX,zoho.com,DIRECT

  # 抗 DNS 污染
  - DOMAIN-KEYWORD,amazon,Proxy
  - DOMAIN-KEYWORD,google,Proxy
  - DOMAIN-KEYWORD,gmail,Proxy
  - DOMAIN-KEYWORD,youtube,Proxy
  - DOMAIN-KEYWORD,facebook,Proxy
  - DOMAIN-SUFFIX,fb.me,Proxy
  - DOMAIN-SUFFIX,fbcdn.net,Proxy
  - DOMAIN-KEYWORD,twitter,Proxy
  - DOMAIN-KEYWORD,instagram,Proxy
  - DOMAIN-KEYWORD,dropbox,Proxy
  - DOMAIN-SUFFIX,twimg.com,Proxy
  - DOMAIN-KEYWORD,blogspot,Proxy
  - DOMAIN-SUFFIX,youtu.be,Proxy
  - DOMAIN-KEYWORD,whatsapp,Proxy

  # 常见广告域名屏蔽
  - DOMAIN-KEYWORD,admarvel,REJECT
  - DOMAIN-KEYWORD,admaster,REJECT
  - DOMAIN-KEYWORD,adsage,REJECT
  - DOMAIN-KEYWORD,adsmogo,REJECT
  - DOMAIN-KEYWORD,adsrvmedia,REJECT
  - DOMAIN-KEYWORD,adwords,REJECT
  - DOMAIN-KEYWORD,adservice,REJECT
  - DOMAIN-SUFFIX,appsflyer.com,REJECT
  - DOMAIN-KEYWORD,domob,REJECT
  - DOMAIN-SUFFIX,doubleclick.net,REJECT
  - DOMAIN-KEYWORD,duomeng,REJECT
  - DOMAIN-KEYWORD,dwtrack,REJECT
  - DOMAIN-KEYWORD,guanggao,REJECT
  - DOMAIN-KEYWORD,lianmeng,REJECT
  - DOMAIN-SUFFIX,mmstat.com,REJECT
  - DOMAIN-KEYWORD,mopub,REJECT
  - DOMAIN-KEYWORD,omgmta,REJECT
  - DOMAIN-KEYWORD,openx,REJECT
  - DOMAIN-KEYWORD,partnerad,REJECT
  - DOMAIN-KEYWORD,pingfore,REJECT
  - DOMAIN-KEYWORD,supersonicads,REJECT
  - DOMAIN-KEYWORD,uedas,REJECT
  - DOMAIN-KEYWORD,umeng,REJECT
  - DOMAIN-KEYWORD,usage,REJECT
  - DOMAIN-SUFFIX,vungle.com,REJECT
  - DOMAIN-KEYWORD,wlmonitor,REJECT
  - DOMAIN-KEYWORD,zjtoolbar,REJECT

  # 国外网站
  - DOMAIN-SUFFIX,9to5mac.com,Proxy
  - DOMAIN-SUFFIX,abpchina.org,Proxy
  - DOMAIN-SUFFIX,adblockplus.org,Proxy
  - DOMAIN-SUFFIX,adobe.com,Proxy
  - DOMAIN-SUFFIX,akamaized.net,Proxy
  - DOMAIN-SUFFIX,alfredapp.com,Proxy
  - DOMAIN-SUFFIX,amplitude.com,Proxy
  - DOMAIN-SUFFIX,ampproject.org,Proxy
  - DOMAIN-SUFFIX,android.com,Proxy
  - DOMAIN-SUFFIX,angularjs.org,Proxy
  - DOMAIN-SUFFIX,aolcdn.com,Proxy
  - DOMAIN-SUFFIX,apkpure.com,Proxy
  - DOMAIN-SUFFIX,appledaily.com,Proxy
  - DOMAIN-SUFFIX,appshopper.com,Proxy
  - DOMAIN-SUFFIX,appspot.com,Proxy
  - DOMAIN-SUFFIX,arcgis.com,Proxy
  - DOMAIN-SUFFIX,archive.org,Proxy
  - DOMAIN-SUFFIX,armorgames.com,Proxy
  - DOMAIN-SUFFIX,aspnetcdn.com,Proxy
  - DOMAIN-SUFFIX,att.com,Proxy
  - DOMAIN-SUFFIX,awsstatic.com,Proxy
  - DOMAIN-SUFFIX,azureedge.net,Proxy
  - DOMAIN-SUFFIX,azurewebsites.net,Proxy
  - DOMAIN-SUFFIX,bing.com,Proxy
  - DOMAIN-SUFFIX,bintray.com,Proxy
  - DOMAIN-SUFFIX,bit.com,Proxy
  - DOMAIN-SUFFIX,bit.ly,Proxy
  - DOMAIN-SUFFIX,bitbucket.org,Proxy
  - DOMAIN-SUFFIX,bjango.com,Proxy
  - DOMAIN-SUFFIX,bkrtx.com,Proxy
  - DOMAIN-SUFFIX,blog.com,Proxy
  - DOMAIN-SUFFIX,blogcdn.com,Proxy
  - DOMAIN-SUFFIX,blogger.com,Proxy
  - DOMAIN-SUFFIX,blogsmithmedia.com,Proxy
  - DOMAIN-SUFFIX,blogspot.com,Proxy
  - DOMAIN-SUFFIX,blogspot.hk,Proxy
  - DOMAIN-SUFFIX,bloomberg.com,Proxy
  - DOMAIN-SUFFIX,box.com,Proxy
  - DOMAIN-SUFFIX,box.net,Proxy
  - DOMAIN-SUFFIX,cachefly.net,Proxy
  - DOMAIN-SUFFIX,chromium.org,Proxy
  - DOMAIN-SUFFIX,cl.ly,Proxy
  - DOMAIN-SUFFIX,cloudflare.com,Proxy
  - DOMAIN-SUFFIX,cloudfront.net,Proxy
  - DOMAIN-SUFFIX,cloudmagic.com,Proxy
  - DOMAIN-SUFFIX,cmail19.com,Proxy
  - DOMAIN-SUFFIX,cnet.com,Proxy
  - DOMAIN-SUFFIX,cocoapods.org,Proxy
  - DOMAIN-SUFFIX,comodoca.com,Proxy
  - DOMAIN-SUFFIX,crashlytics.com,Proxy
  - DOMAIN-SUFFIX,culturedcode.com,Proxy
  - DOMAIN-SUFFIX,d.pr,Proxy
  - DOMAIN-SUFFIX,danilo.to,Proxy
  - DOMAIN-SUFFIX,dayone.me,Proxy
  - DOMAIN-SUFFIX,db.tt,Proxy
  - DOMAIN-SUFFIX,deskconnect.com,Proxy
  - DOMAIN-SUFFIX,disq.us,Proxy
  - DOMAIN-SUFFIX,disqus.com,Proxy
  - DOMAIN-SUFFIX,disquscdn.com,Proxy
  - DOMAIN-SUFFIX,dnsimple.com,Proxy
  - DOMAIN-SUFFIX,docker.com,Proxy
  - DOMAIN-SUFFIX,dribbble.com,Proxy
  - DOMAIN-SUFFIX,droplr.com,Proxy
  - DOMAIN-SUFFIX,duckduckgo.com,Proxy
  - DOMAIN-SUFFIX,dueapp.com,Proxy
  - DOMAIN-SUFFIX,dytt8.net,Proxy
  - DOMAIN-SUFFIX,edgecastcdn.net,Proxy
  - DOMAIN-SUFFIX,edgekey.net,Proxy
  - DOMAIN-SUFFIX,edgesuite.net,Proxy
  - DOMAIN-SUFFIX,engadget.com,Proxy
  - DOMAIN-SUFFIX,entrust.net,Proxy
  - DOMAIN-SUFFIX,eurekavpt.com,Proxy
  - DOMAIN-SUFFIX,evernote.com,Proxy
  - DOMAIN-SUFFIX,fabric.io,Proxy
  - DOMAIN-SUFFIX,fast.com,Proxy
  - DOMAIN-SUFFIX,fastly.net,Proxy
  - DOMAIN-SUFFIX,fc2.com,Proxy
  - DOMAIN-SUFFIX,feedburner.com,Proxy
  - DOMAIN-SUFFIX,feedly.com,Proxy
  - DOMAIN-SUFFIX,feedsportal.com,Proxy
  - DOMAIN-SUFFIX,fiftythree.com,Proxy
  - DOMAIN-SUFFIX,firebaseio.com,Proxy
  - DOMAIN-SUFFIX,flexibits.com,Proxy
  - DOMAIN-SUFFIX,flickr.com,Proxy
  - DOMAIN-SUFFIX,flipboard.com,Proxy
  - DOMAIN-SUFFIX,g.co,Proxy
  - DOMAIN-SUFFIX,gabia.net,Proxy
  - DOMAIN-SUFFIX,geni.us,Proxy
  - DOMAIN-SUFFIX,gfx.ms,Proxy
  - DOMAIN-SUFFIX,ggpht.com,Proxy
  - DOMAIN-SUFFIX,ghostnoteapp.com,Proxy
  - DOMAIN-SUFFIX,git.io,Proxy
  - DOMAIN-KEYWORD,github,Proxy
  - DOMAIN-SUFFIX,globalsign.com,Proxy
  - DOMAIN-SUFFIX,gmodules.com,Proxy
  - DOMAIN-SUFFIX,godaddy.com,Proxy
  - DOMAIN-SUFFIX,golang.org,Proxy
  - DOMAIN-SUFFIX,gongm.in,Proxy
  - DOMAIN-SUFFIX,goo.gl,Proxy
  - DOMAIN-SUFFIX,goodreaders.com,Proxy
  - DOMAIN-SUFFIX,goodreads.com,Proxy
  - DOMAIN-SUFFIX,gravatar.com,Proxy
  - DOMAIN-SUFFIX,gstatic.com,Proxy
  - DOMAIN-SUFFIX,gvt0.com,Proxy
  - DOMAIN-SUFFIX,hockeyapp.net,Proxy
  - DOMAIN-SUFFIX,hotmail.com,Proxy
  - DOMAIN-SUFFIX,icons8.com,Proxy
  - DOMAIN-SUFFIX,ifixit.com,Proxy
  - DOMAIN-SUFFIX,ift.tt,Proxy
  - DOMAIN-SUFFIX,ifttt.com,Proxy
  - DOMAIN-SUFFIX,iherb.com,Proxy
  - DOMAIN-SUFFIX,imageshack.us,Proxy
  - DOMAIN-SUFFIX,img.ly,Proxy
  - DOMAIN-SUFFIX,imgur.com,Proxy
  - DOMAIN-SUFFIX,imore.com,Proxy
  - DOMAIN-SUFFIX,instapaper.com,Proxy
  - DOMAIN-SUFFIX,ipn.li,Proxy
  - DOMAIN-SUFFIX,is.gd,Proxy
  - DOMAIN-SUFFIX,issuu.com,Proxy
  - DOMAIN-SUFFIX,itgonglun.com,Proxy
  - DOMAIN-SUFFIX,itun.es,Proxy
  - DOMAIN-SUFFIX,ixquick.com,Proxy
  - DOMAIN-SUFFIX,j.mp,Proxy
  - DOMAIN-SUFFIX,js.revsci.net,Proxy
  - DOMAIN-SUFFIX,jshint.com,Proxy
  - DOMAIN-SUFFIX,jtvnw.net,Proxy
  - DOMAIN-SUFFIX,justgetflux.com,Proxy
  - DOMAIN-SUFFIX,kat.cr,Proxy
  - DOMAIN-SUFFIX,klip.me,Proxy
  - DOMAIN-SUFFIX,libsyn.com,Proxy
  - DOMAIN-SUFFIX,linkedin.com,Proxy
  - DOMAIN-SUFFIX,linode.com,Proxy
  - DOMAIN-SUFFIX,lithium.com,Proxy
  - DOMAIN-SUFFIX,littlehj.com,Proxy
  - DOMAIN-SUFFIX,live.com,Proxy
  - DOMAIN-SUFFIX,live.net,Proxy
  - DOMAIN-SUFFIX,livefilestore.com,Proxy
  - DOMAIN-SUFFIX,llnwd.net,Proxy
  - DOMAIN-SUFFIX,macid.co,Proxy
  - DOMAIN-SUFFIX,macromedia.com,Proxy
  - DOMAIN-SUFFIX,macrumors.com,Proxy
  - DOMAIN-SUFFIX,mashable.com,Proxy
  - DOMAIN-SUFFIX,mathjax.org,Proxy
  - DOMAIN-SUFFIX,medium.com,Proxy
  - DOMAIN-SUFFIX,mega.co.nz,Proxy
  - DOMAIN-SUFFIX,mega.nz,Proxy
  - DOMAIN-SUFFIX,megaupload.com,Proxy
  - DOMAIN-SUFFIX,microsofttranslator.com,Proxy
  - DOMAIN-SUFFIX,mindnode.com,Proxy
  - DOMAIN-SUFFIX,mobile01.com,Proxy
  - DOMAIN-SUFFIX,modmyi.com,Proxy
  - DOMAIN-SUFFIX,msedge.net,Proxy
  - DOMAIN-SUFFIX,myfontastic.com,Proxy
  - DOMAIN-SUFFIX,name.com,Proxy
  - DOMAIN-SUFFIX,nextmedia.com,Proxy
  - DOMAIN-SUFFIX,nsstatic.net,Proxy
  - DOMAIN-SUFFIX,nssurge.com,Proxy
  - DOMAIN-SUFFIX,nyt.com,Proxy
  - DOMAIN-SUFFIX,nytimes.com,Proxy
  - DOMAIN-SUFFIX,omnigroup.com,Proxy
  - DOMAIN-SUFFIX,onedrive.com,Proxy
  - DOMAIN-SUFFIX,onenote.com,Proxy
  - DOMAIN-SUFFIX,ooyala.com,Proxy
  - DOMAIN-SUFFIX,openvpn.net,Proxy
  - DOMAIN-SUFFIX,openwrt.org,Proxy
  - DOMAIN-SUFFIX,orkut.com,Proxy
  - DOMAIN-SUFFIX,osxdaily.com,Proxy
  - DOMAIN-SUFFIX,outlook.com,Proxy
  - DOMAIN-SUFFIX,ow.ly,Proxy
  - DOMAIN-SUFFIX,paddleapi.com,Proxy
  - DOMAIN-SUFFIX,parallels.com,Proxy
  - DOMAIN-SUFFIX,parse.com,Proxy
  - DOMAIN-SUFFIX,pdfexpert.com,Proxy
  - DOMAIN-SUFFIX,periscope.tv,Proxy
  - DOMAIN-SUFFIX,pinboard.in,Proxy
  - DOMAIN-SUFFIX,pinterest.com,Proxy
  - DOMAIN-SUFFIX,pixelmator.com,Proxy
  - DOMAIN-SUFFIX,pixiv.net,Proxy
  - DOMAIN-SUFFIX,playpcesor.com,Proxy
  - DOMAIN-SUFFIX,playstation.com,Proxy
  - DOMAIN-SUFFIX,playstation.com.hk,Proxy
  - DOMAIN-SUFFIX,playstation.net,Proxy
  - DOMAIN-SUFFIX,playstationnetwork.com,Proxy
  - DOMAIN-SUFFIX,pushwoosh.com,Proxy
  - DOMAIN-SUFFIX,rime.im,Proxy
  - DOMAIN-SUFFIX,servebom.com,Proxy
  - DOMAIN-SUFFIX,sfx.ms,Proxy
  - DOMAIN-SUFFIX,shadowsocks.org,Proxy
  - DOMAIN-SUFFIX,sharethis.com,Proxy
  - DOMAIN-SUFFIX,shazam.com,Proxy
  - DOMAIN-SUFFIX,skype.com,Proxy
  - DOMAIN-SUFFIX,smartdnsProxy.com,Proxy
  - DOMAIN-SUFFIX,smartmailcloud.com,Proxy
  - DOMAIN-SUFFIX,sndcdn.com,Proxy
  - DOMAIN-SUFFIX,sony.com,Proxy
  - DOMAIN-SUFFIX,soundcloud.com,Proxy
  - DOMAIN-SUFFIX,sourceforge.net,Proxy
  - DOMAIN-SUFFIX,spotify.com,Proxy
  - DOMAIN-SUFFIX,squarespace.com,Proxy
  - DOMAIN-SUFFIX,sstatic.net,Proxy
  - DOMAIN-SUFFIX,st.luluku.pw,Proxy
  - DOMAIN-SUFFIX,stackoverflow.com,Proxy
  - DOMAIN-SUFFIX,startpage.com,Proxy
  - DOMAIN-SUFFIX,staticflickr.com,Proxy
  - DOMAIN-SUFFIX,steamcommunity.com,Proxy
  - DOMAIN-SUFFIX,symauth.com,Proxy
  - DOMAIN-SUFFIX,symcb.com,Proxy
  - DOMAIN-SUFFIX,symcd.com,Proxy
  - DOMAIN-SUFFIX,tapbots.com,Proxy
  - DOMAIN-SUFFIX,tapbots.net,Proxy
  - DOMAIN-SUFFIX,tdesktop.com,Proxy
  - DOMAIN-SUFFIX,techcrunch.com,Proxy
  - DOMAIN-SUFFIX,techsmith.com,Proxy
  - DOMAIN-SUFFIX,thepiratebay.org,Proxy
  - DOMAIN-SUFFIX,theverge.com,Proxy
  - DOMAIN-SUFFIX,time.com,Proxy
  - DOMAIN-SUFFIX,timeinc.net,Proxy
  - DOMAIN-SUFFIX,tiny.cc,Proxy
  - DOMAIN-SUFFIX,tinypic.com,Proxy
  - DOMAIN-SUFFIX,tmblr.co,Proxy
  - DOMAIN-SUFFIX,todoist.com,Proxy
  - DOMAIN-SUFFIX,trello.com,Proxy
  - DOMAIN-SUFFIX,trustasiassl.com,Proxy
  - DOMAIN-SUFFIX,tumblr.co,Proxy
  - DOMAIN-SUFFIX,tumblr.com,Proxy
  - DOMAIN-SUFFIX,tweetdeck.com,Proxy
  - DOMAIN-SUFFIX,tweetmarker.net,Proxy
  - DOMAIN-SUFFIX,twitch.tv,Proxy
  - DOMAIN-SUFFIX,txmblr.com,Proxy
  - DOMAIN-SUFFIX,typekit.net,Proxy
  - DOMAIN-SUFFIX,ubertags.com,Proxy
  - DOMAIN-SUFFIX,ublock.org,Proxy
  - DOMAIN-SUFFIX,ubnt.com,Proxy
  - DOMAIN-SUFFIX,ulyssesapp.com,Proxy
  - DOMAIN-SUFFIX,urchin.com,Proxy
  - DOMAIN-SUFFIX,usertrust.com,Proxy
  - DOMAIN-SUFFIX,v.gd,Proxy
  - DOMAIN-SUFFIX,v2ex.com,Proxy
  - DOMAIN-SUFFIX,vimeo.com,Proxy
  - DOMAIN-SUFFIX,vimeocdn.com,Proxy
  - DOMAIN-SUFFIX,vine.co,Proxy
  - DOMAIN-SUFFIX,vivaldi.com,Proxy
  - DOMAIN-SUFFIX,vox-cdn.com,Proxy
  - DOMAIN-SUFFIX,vsco.co,Proxy
  - DOMAIN-SUFFIX,vultr.com,Proxy
  - DOMAIN-SUFFIX,w.org,Proxy
  - DOMAIN-SUFFIX,w3schools.com,Proxy
  - DOMAIN-SUFFIX,webtype.com,Proxy
  - DOMAIN-SUFFIX,wikiwand.com,Proxy
  - DOMAIN-SUFFIX,wikileaks.org,Proxy
  - DOMAIN-SUFFIX,wikimedia.org,Proxy
  - DOMAIN-SUFFIX,wikipedia.com,Proxy
  - DOMAIN-SUFFIX,wikipedia.org,Proxy
  - DOMAIN-SUFFIX,windows.com,Proxy
  - DOMAIN-SUFFIX,windows.net,Proxy
  - DOMAIN-SUFFIX,wire.com,Proxy
  - DOMAIN-SUFFIX,wordpress.com,Proxy
  - DOMAIN-SUFFIX,workflowy.com,Proxy
  - DOMAIN-SUFFIX,wp.com,Proxy
  - DOMAIN-SUFFIX,wsj.com,Proxy
  - DOMAIN-SUFFIX,wsj.net,Proxy
  - DOMAIN-SUFFIX,xda-developers.com,Proxy
  - DOMAIN-SUFFIX,xeeno.com,Proxy
  - DOMAIN-SUFFIX,xiti.com,Proxy
  - DOMAIN-SUFFIX,yahoo.com,Proxy
  - DOMAIN-SUFFIX,yimg.com,Proxy
  - DOMAIN-SUFFIX,ying.com,Proxy
  - DOMAIN-SUFFIX,yoyo.org,Proxy
  - DOMAIN-SUFFIX,ytimg.com,Proxy

  # Telegram
  - DOMAIN-SUFFIX,telegra.ph,Proxy
  - DOMAIN-SUFFIX,telegram.org,Proxy
  - IP-CIDR,91.108.4.0/22,Proxy
  - IP-CIDR,91.108.8.0/21,Proxy
  - IP-CIDR,91.108.16.0/22,Proxy
  - IP-CIDR,91.108.56.0/22,Proxy
  - IP-CIDR,149.154.160.0/20,Proxy
  - IP-CIDR6,2001:67c:4e8::/48,Proxy
  - IP-CIDR6,2001:b28:f23d::/48,Proxy
  - IP-CIDR6,2001:b28:f23f::/48,Proxy

  # LAN
  - DOMAIN,injections.adguard.org,DIRECT
  - DOMAIN,local.adguard.org,DIRECT
  - DOMAIN-SUFFIX,local,DIRECT
  - IP-CIDR,127.0.0.0/8,DIRECT
  - IP-CIDR,172.16.0.0/12,DIRECT
  - IP-CIDR,192.168.0.0/16,DIRECT
  - IP-CIDR,10.0.0.0/8,DIRECT
  - IP-CIDR,17.0.0.0/8,DIRECT
  - IP-CIDR,100.64.0.0/10,DIRECT
  - IP-CIDR,224.0.0.0/4,DIRECT
  - IP-CIDR6,fe80::/10,DIRECT

  # 最终规则
  - GEOIP,CN,DIRECT
  - MATCH,Proxy
```