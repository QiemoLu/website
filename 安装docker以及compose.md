```
    yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 添加docker yum 源
```
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```
#### 配置镜像加速器：自由选择
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://zggyaen3.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
#### 更新yum源缓存， 安装docker-ce

```
sudo yum makecache fast
sudo yum install docker-ce 
```
#### 普通用户需要加入docker组
```
sudo usermod -a -G docker ${USER} 
```
#### start docker
```
sudo systemctl start docker
sudo systemctl enable docker
```
#### 安装docker-compose

```
$ sudo curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose

无法访问github 可以用pip 安装
pip install docker-compose 
```
