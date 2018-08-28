### 在centos7上安装docker

---
    由于docker需要翻墙，所有加载国内镜像，本文使用的阿里云的
    镜像源：http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
---

- **环境准备**

--- 
    准备一台centos7.x机器，本文采用的是centos7.5。
    1、关闭selinux
        
        $ sudo vim /etc/selinux/config
             SELINUX=disabled
        保存退出
        如果要立即生效：
        
        sudo setenforce 0
    
    2、开启防火墙端口
        
        $ sudo firewall-cmd --list-all #查看所有开启的端口
        $ sudo firewall-cmd --add-port=3306/tcp --permanent --zone=public #开启3306端口
        $ sudo firewall-cmd --remove-port=3306/tcp --permanent --zone=public #关闭3306端口
        
    3、关闭防火墙（不建议）
        
        # 开启：
        $ sudo systemctl start firewalld
        $ sudo systemctl enable firewalld 
        
        # 关闭:
        $ sudo systemctl stop firewalld
        $ sudo systemctl disable firewalld
        
        

- **安装**

---
    docker的安装有很多种方式，具体的可以参考官方 https://www.docker.com/
    本文介绍国内镜像的安装方式：
    
    1、卸载：
        
        $ sudo yum remove docker \
               docker-client \
               docker-client-latest \
               docker-common \
               docker-latest \
               docker-latest-logrotate \
               docker-logrotate \
               docker-selinux \
               docker-engine-selinux \
               docker-engine
    
    2、安装依赖：
        
        $ sudo yum install -y yum-utils \
            device-mapper-persistent-data \
            lvm2
    
    3、配置国内yum仓库
        
        $ sudo yum-config-manager \
            --add-repo \
            http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    
    4、查看docker版本：
        
        $ sudo yum list docker-ce --showduplicates
        
    5、选择对应的版本进行安装，本文使用17.03.2.ce版本进行安装
        
        $ sudo yum install docker-ce-17.03.2.ce
        
       如果需要安装最新的版本：
       
        $ sudo yum install docker-ce
        
       注意：在安装17.x版本时，需要安装 docker-ce-selinux-17.x
       下载地址：yum install https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm 
       所以要提前下载好该版本
    6、启动docker
        
        $ sudo systemctl start docker
        $ sudo systemctl enable docker #开机启动
        
    7、验证docker，安装完成后需要查看docker信息
    
        $ sudo docker version 
       
       或
        $ sudo docker info
        
    8、配置docker的跟目录
       默认在/var/lib/docker，修改（根据自己的规划），本文放在/data/docker
       
       $ sudo mkdir -p /data/docker
       $ sudo vim /lib/systemd/system/docker.service 
          ExecStart=/usr/bin/dockerd --graph /data/docker
       $ sudo systemctl daemon-reload
       $ sudo systemctl restart docker 
     
---

- **创建私有仓库**

---
    1、在docker上执行：
    
        $ docker run -d \
          -p 5000:5000 \
          -v /data/registry:/var/lib/registry \
          --restart=always \
          --name registry \
          registry:2
          
    错误：
        Unable to find image 'registry:2' locally
        docker: Error response from daemon: Get https://registry-1.docker.io/v2/: x509: certificate signed by unknown authority.
   
    原因一：
        证书的原因，建议使用CA机构颁发的证书，本文使用自制证书（不推荐），参考:证书制作
    
    带证书启动：
    
    $ docker run -d -p 5000:5000 --restart=always --name registry \
      -v /data/registry:/var/lib/registry \
      -v /data/certs:/certs \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      registry:2
    
    原因二：
       可能是国内镜像源的问题，请更换镜像源，本例子中可用的是http://abcd1234.m.daocloud.io
       在/etc/docker/daemon.json文件中添加：
       {"registry-mirrors": ["http://abcd1234.m.daocloud.io"]}
       如果该目录下没有daemon.json文件，创建一个
       重启docker服务。
    原因三：
        可能是openssl包没有安装全
        是用命令：yum install pcre pcre-devel zlib-devel openssl openssl-devel -y
    
    2、再daemon.json中添加"insecure-registries":["IP:5000"]，重启docker服务
       目的是为了忽略ssl，如果缺少，在push到私有仓库时，会有证书问题
    3、验证是否成功：
       curl http://ip:5000/v2/_catalog
       {"repositories":[]}
       说明成功
    4、push 镜像文件到私有仓库中
        
      $ docker tag busybox ip/busybox #用busybox作测试，ip为私有仓库ip地址
      $ docker push ip/busybox        #push镜像到私有仓库
      在游览器中输入http://ip:5000/v2/_catalog 就可以看到
      {"repositories":["busybox"]}
      说明push成功
    5、从私有仓库pull镜像到本地
       
       $ docker image rm ip/busybox  #删除本地的镜像
       $ docker pull 192.168.18.207:5000/busybox # 从私有仓库下载镜像到本地
       $ docker image ls 如果有，说明成功
        
      
            
---

- **CA证书制作**

---
    openssl req -newkey rsa:2048 -nodes -sha256 -keyout /data/certs/domain.key -x509 -days 365 -out /data/certs/domain.crt
    
    
    
      

