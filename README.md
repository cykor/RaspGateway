# RaspGateway
  使用 Raspberry Pi 和 Clash 打造即插即用的透明网关/代理

# 编译和准备 Clash

# 网络配置

# 使用 Supervisor 监护 Clash 进程

# 使用 yacd 作为控制前端

# 使用 Cron 自动更新和重启 Clash

# Extra

## 使用阿里镜像
  在 /etc/apt/sources.list 中注释掉原有的 repo，加入阿里的镜像：
  
    deb https://mirrors.aliyun.com/raspbian/raspbian/ buster main contrib non-free rpi

    # deb http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi


## 关闭蓝牙和 Wi-Fi
  如果只作为一个网关运行，可以关闭蓝牙和 Wi-Fi，省电的同时纯净无线空间。
  
  编辑/boot 分区的 config.txt，加入以下两行：
 
    dtoverlay=pi3-disable-wifi

    dtoverlay=pi3-disable-bt

