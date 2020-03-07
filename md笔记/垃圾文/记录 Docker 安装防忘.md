# 记录 Docker 安装防忘

### 第一步、官方安装 Docker

具体看官方文档，官方是真滴烂

 https://docs.docker.com/install/linux/docker-ce/centos/ 





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

