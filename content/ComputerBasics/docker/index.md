---
title: "docker基础"  
date: 2026-06-02T20:00:00+08:00
draft: false   
---

### docker配置

安装docker

```
brew install --cask --appdir=/Applications docker
```

设置-Docker Engine-换国内的镜像

```
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "20GB"
    }
  },
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "registry-mirrors":[
    "https://alzgoonw.mirror.aliyuncs.com"
  ]
}
```

命令行安装awvs

```
docker pull secfa/awvs
```

启动容器

```
docker run -d -p 3443:3443 secfa/awvs
```

```
账号： admin@admin.com
密码：Admin123
```

docker查看所有容器

```
docker ps -a
```

docker查看正在运行的容器

```
docker ps
```

确认是否正常连接

```
curl -k https://localhost:13443
```

浏览器可以打开

```
https://localhost:13443
```

修改镜像源配置

```
vim ~/.docker/daemon.json
```

镜像源配置

docker配置镜像源：

```
sudo vim /etc/docker/daemon.json
```

输入：

```
{
  "registry-mirrors": [
    "https://docker-cf.registry.cyou",
    "https://dockercf.jsdelivr.fyi",
    "https://docker.jsdelivr.fyi",
    "https://dockertest.jsdelivr.fyi",
    "https://mirror.aliyuncs.com",
    "https://dockerproxy.com",
    "https://mirror.baidubce.com",
    "https://docker.m.daocloud.io",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.sjtug.sjtu.edu.cn",
    "https://docker.mirrors.ustc.edu.cn",
    "https://mirror.iscas.ac.cn",
    "https://docker.rainbond.cc"
  ]
}
```

问题：

在配置题目环境时芯片不兼容问题，修改dockerfile中的镜像源解决

```
chmox -x file 
将unix可执行文件转化为文稿

chmox +x file
将文稿转化为unix可执行文件
```

或者直接使用转译

```
docker build --platform=linux/amd64 -t my_image_name .
```

容器启动，创建终端会话

```
docker run -it my_image_name

//创到端口
docker run -it -p 8090:80 my_image_name
```

容器关闭

```
docker stop name
```

连接ssh

```
ssh ctfix@localhost -p 8091
```

容器内查看apache状态

```
ps -ef | grep apache2 
```

重启apache

```
sudo apache2ctl restart
```

遇到路径失效，可以使用sudo -l查看普通用户啊也能使用的sudo命令，这些命令在/etc/sudoers文件夹中保存的

遇到restart.sh文件重启apache失效，将其重写为

```
#!/bin/bash
set -e

echo "Stopping Apache..."
apache2ctl stop || true

echo "Removing stale PID file if any..."
rm -f /var/run/apache2/apache2.pid || true

echo "Starting Apache..."
apache2ctl start

echo "Apache restarted successfully!"
```

查看apache最后启动时间

```
ps -p $(cat /var/run/apache2/apache2.pid) -o lstart
```

修改docker文件

```
docker exec -it 容器ID/容器名称 /bin/bash
```

做题流程：

修改apache配置文件，在etc/apache2/apache2.conf文件下，把AllowOveride改为ALL

在需要隐藏的目录下添加.htaccess文件，内容为,控制当访问的目录下没有默认索引文件

```
Options Indexes 
```

sudo -l可以查看当前用户能执行的sudo权限，在/etc/sudoers中能进行修改

保存新镜像

```
docker commit tomcat9 my-tomcat:cve-2025-24813
```

打包为tar文件

```
docker save -o my-tomcat-cve.tar my-tomcat:cve-2025-24813
```

注意：在服务器上部署时，需要保证docker镜像的repository名字和服务器的docker容器名相同，否则拉不起来

### docker-compose

多容器应用，管理整个应用程序的服务集群

使用`docker-compose.yml`定义服务

```bash
# 启动并运行
docker compose up

# 后台启动运行
docker compose up -d

# 停止并删除目录下所有容器（不会删除images，在目录下执行）
docker compose down
```

### 打包KVM

本地文件传输到远程服务器

```
scp -r "C:\题目信息\题目信息2025_10" root@192.168.255.128:/root/docker_env/
```

生成flag文件，注意使用echo，使用vim会多一行换行

```
echo -n "before" > /flag.txt
```

构建镜像

```
docker build -t st_paicha_hnjs2025_nmdl:latest .
```

导出为tar包

```
docker save -o st_paicha_hnjs2025_nmdl.tar st_paicha_hnjs2025_nmdl:latest
```

/root/start_docker.sh添加自启动脚本

```
docker run -itd \
  -p 8806:80 \
  -p 2222:22 \
  -v "/flag.txt:/tmp/temp" \
  --name st_paicha_hnjs2025_shirofxlh \
  st_paicha_hnjs2025_shirofxlh
```

将脚本添加到crontab（linux周期性执行命令）中

```
chmod +x start_docker.sh
crontab -e
### 加入一行
@reboot /root/start_docker.sh
```

删除所有痕迹

```
cat ~/.bash_history
cat /root/.bash_history
# 删除
history -c         # 清除当前 shell session 历史
cat /dev/null > ~/.bash_history  # 清空历史文件
```

导出qcow2，关机-压缩磁盘-导出为ovf，磁盘-压缩-文件-导出为ovf

在ovf文件路径下转换为qcow2格式

```
& 'D:\qemu\qemu-img.exe' convert -c -O qcow2 ubuntu-disk1.vmdk st_paicha_hnjs2025_nmdl.qcow2
```

