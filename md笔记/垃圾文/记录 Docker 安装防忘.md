# 记录 Docker 安装防迷路

### 第一步、官方安装 Docker

具体看官方文档

记得更新 yum

```
sudo yum update
```

 https://docs.docker.com/install/linux/docker-ce/centos/ 



可是官方太烂，无脑下载看这个

 https://qizhanming.com/blog/2019/01/25/how-to-install-docker-ce-on-centos-7 

### 第二部、配置国内镜像

[阿里云](https://cr.console.aliyun.com/)

试了无数家都不能用，还是阿里好用。

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://54q7ua79.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### mysql 的安装

 https://www.jianshu.com/p/10769f985516 

