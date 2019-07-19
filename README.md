# RaspGateway
  使用 Raspberry Pi 和 Clash 打造即插即用的透明网关/代理

# 编译和准备 Clash
## 交叉编译

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
    static ip_address=192.168.1.6/24
    static routers=192.168.1.1

打开转发：

    echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf && sysctl -p
    echo "net.ipv6.conf.all.forwarding = 1" >> /etc/sysctl.conf && sysctl -p

设置并保存转发规则：

    iptables -t nat -N Clash
    iptables -t nat -A Clash -d 192.168.0.0/16 -j RETURN
    iptables -t nat -A Clash -p tcp -j REDIRECT --to-ports 29567
    iptables -t nat -A PREROUTING -p tcp -j Clash
    netfilter-persistent save



# 使用 Supervisor 监护 Clash 进程

# 使用 yacd 作为控制前端

# 使用 Cron 自动更新和重启 Clash

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

