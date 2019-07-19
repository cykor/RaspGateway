# RaspGateway
  使用 Raspberry Pi (Rapbian 10) 和 Clash (v0.15.0) 打造即插即用的透明网关/代理。
  
- [编译和准备 Clash](#编译和准备-clash)
  * [准备配置文件](#准备配置文件)
  * [交叉编译 Clash](#交叉编译-clash)
  * [准备 Clash](#准备-clash)
- [网络配置](#网络配置)
- [使用 Supervisor 监护 Clash 进程](#使用-supervisor-监护-clash-进程)
- [使用 yacd 作为控制前端](#使用-yacd-作为控制前端)
- [使用 Cron 自动更新和重启 Clash](#使用-cron-自动更新和重启-clash)
- [使用方法](#使用方法)
- [Extra](#extra)
  * [使用阿里镜像](#使用阿里镜像)
  * [关闭蓝牙和 Wi-Fi](#关闭蓝牙和-Wi-Fi)
- [感谢](#感谢)


# 编译和准备 Clash

## 准备配置文件
Clash 配置文件的前半部分修改如下：

    port: HTTP代理端口
    socks-port: SOCKS代理端口
    redir-port: 转发端口
    allow-lan: true
    mode: Rule
    log-level: error
    external-controller: 0.0.0.0:配置端口
    secret: ""
    dns:
      enable: true
      ipv6: false
      listen: 0.0.0.0:53
      enhanced-mode: fake-ip
      nameserver:
      - 223.5.5.5
      - tls://dns.rubyfish.cn:853
      - tls://1.1.1.1:853
      - tls://dns.google
      - tcp://1.1.1.1:53
      - tcp://208.67.222.222:443


## 交叉编译 Clash

  Mac 或 Linux 上可以：
  
    go get -u -v github.com/Dreamacro/clash
    cd ~/go/src/github.com/Dreamacro/clash
    GOARCH=arm GOOS=linux GOARM=7 CGO_ENABLED=0 go build -ldflags '-w -s'
 
 编译的时候有个小陷阱，编译到 `go-shadowsocks2` 时会报 `undefined ... chacha20poly1305.NewX`。
 
 找到那个源文件把 `X` 删掉就好了。

## 准备 Clash
 
 上传 Clash 和你的配置文件到 Raspberry Pi：
 
    scp clash pi@raspberrypi.local:
    scp config.yaml pi@raspberrypi.local:
    
 登录 Raspberry Pi，把 Clash 和配置文件放到 `/opt/clash` 目录
 
    sudo -i
    cd /home/pi
    chown root:root clash
    mkdir -p /opt/clash
    mv clash config.yaml /opt/clash/


# 网络配置

编辑 `/etc/dhcpcd.conf`，固定 Raspberry Pi 的 IP：

    interface eth0
    static ip_address=固定IP/24
    static routers=路由器IP

打开转发：

    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf && sysctl -p
    echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf && sysctl -p

设置并保存转发规则：

    iptables -t nat -N Clash
    iptables -t nat -A Clash -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A Clash -p tcp -j REDIRECT --to-ports 转发端口
    iptables -t nat -A PREROUTING -p tcp -j Clash
    netfilter-persistent save


# 使用 Supervisor 监护 Clash 进程

创建 `/etc/supervisor/conf.d/clash.conf`：

    [supervisord]
    nodaemon=false

    [program:clash]
    priority=1
    directory=/opt/clash
    command=/opt/clash/clash -d .
    autorestart=true

启动 Supervisor：

    systemctl restart supervisor
    systemctl enable supervisor


# 使用 yacd 作为控制前端

安装 git 和 build 工具：

    apt install git build-essential

安装 nodejs：

    curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash - 
    apt install nodejs 

安装 yarn：

    curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
    echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
    apt update && sudo apt install yarn

安装 nginx：

    apt -y install nginx unzip
    mv /etc/nginx/sites-enabled/default /etc/nginx/sites-enabled/default.bak

下载 yacd：

    git clone https://github.com/haishanh/yacd.git
    cd yacd

可以编辑 `src/ducks/app.js`，把登录页面上缺省的服务器和端口改为你习惯的：

    const defaultState = {
      clashAPIConfig: {
        hostname: '固定IP’,
        port: ‘配置端口’,
        secret: ''
      },

使用 yarn 编译安装：

    yarn
    yarn build
    cp -r public/. /usr/share/nginx/html/yacd

创建 `/etc/nginx/conf.d/yacd.conf`：

    server {
        listen       80;
        server_name  固定IP;
        root /usr/share/nginx/html/yacd;
        index index.html;
    }

启动 nginx：

    systemctl start nginx
    systemctl enable nginx



# 使用 Cron 自动更新和重启 Clash

  执行 `crontab -e`，加入以下两行（每天 3:45 更新配置，3:50 重启 Clash）：
    
    45 3 * * 1 /usr/bin/wget 你的配置链接 -O /opt/clash/config.yaml
    50 3 * * * /usr/bin/supervisorctl -c /etc/supervisor/supervisord.conf restart clash 


# 使用方法

* 作为网关

将设备的 IP 设置改为手动，设置如下：

    IP：随便，不冲突就行
    网关：固定IP
    DNS：固定IP

- 作为代理

  - 作为 HTTP 代理

         固定IP:HTTP端口

  - 作为 SOCKS 代理

        固定IP:SOCKS端口

- 访问配置界面
在任何浏览器中打开：

    http://固定IP

# Extra

## 使用阿里镜像
  在 `/etc/apt/sources.list` 中注释掉原有的 repo，加入阿里的镜像：
  
    deb https://mirrors.aliyun.com/raspbian/raspbian/ buster main contrib non-free rpi
    # deb http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi


## 关闭蓝牙和 Wi-Fi
  如果只作为一个网关运行，可以关闭蓝牙和 Wi-Fi，省电的同时纯净无线空间。
  
  编辑 `/boot/config.txt` 中加入以下两行：
 
    dtoverlay=pi3-disable-wifi
    dtoverlay=pi3-disable-bt

# 感谢
  
* [Clash](https://github.com/Dreamacro/clash)
* [yacd](https://github.com/haishanh/yacd)
* [go-shadowsocks2](https://github.com/riobard/go-shadowsocks2)
* [V2Ray](https://github.com/v2ray/v2ray-core)

* [使用Debian9自己打造一个旁路由](https://lala.im/5727.html)
* [在树莓派上使用kone和clash](https://beyondkmp.com/post/kone_clash/)

