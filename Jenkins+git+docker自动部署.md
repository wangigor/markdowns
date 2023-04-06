# Jenkins+git+docker自动部署

## 环境准备

**准备三台虚拟机。**

> 这里采用vagrant。
>
> ```shell
> Vagrant.configure("2") do |config|
> 	(1..3).each do |i|
> 		config.disksize.size = "100GB"
> 		config.vm.define "3nodes-node0#{i}" do |node|
> 			# 设置虚拟机的Box
> 			node.vm.box = "centos/7"
> 			# 设置虚拟机的主机名
> 			node.vm.hostname="3nodes-node0#{i}"
> 			# 设置虚拟机的IP
> 			node.vm.network "public_network", :bridge => 'en9: USB 10/100/1000 LAN',ip: "192.168.8.#{100+i}"
> 			# 设置主机与虚拟机的共享目录
> 			node.vm.synced_folder "/Users/wangke/3nodes-virtual/data/#{i}", "/home/vagrant/share"
> 			# VirtaulBox相关配置
> 			node.vm.provider "virtualbox" do |v|
> 				# 设置虚拟机的名称
> 				v.name = "3nodes-node0#{i}"
> 				# 设置虚拟机的内存大小
> 				v.memory = 4096
> 				# 设置虚拟机的CPU个数
> 				v.cpus = 4
> 			end
> 		end
> 	end
> end
> ```
>
> 并开启ssh。
>
> 这样我们就有了三台虚拟机「192.168.9.101」「192.168.9.102」「192.168.9.103」。

三台都安装docker。

> 一台用来做docker hub。
>
> 一台用来做jenkins服务的docker版。
>
> 一台用来发布web服务。
>
> ```shell
> yum install -y yum-utils
> 
> yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
> 
> yum -y install docker-ce-18.09.6
> 
> systemctl start docker && systemctl enable docker
> ```
>
> 添加镜像加速器。
>
> ```shell
> sudo mkdir -p /etc/docker
> sudo tee /etc/docker/daemon.json <<-'EOF'
> {
>   "registry-mirrors": ["https://XXXXXXXXX.mirror.aliyuncs.com"]
> }
> EOF
> sudo systemctl daemon-reload
> sudo systemctl restart docker
> ```
>
> 安装docker-compose
>
> ```shell
> curl -L "https://get.daocloud.io/docker/compose/releases/download/1.27.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
> chmod +x /usr/local/bin/docker-compose
> ```





## **192.168.9.101 安装『docker registry』**

```yaml
version: '3.1'
services:
  registry:
    image: registry
    restart: always
    container_name: registry
    ports:
      - 5000:5000
    volumes:
      - /usr/local/docker/registry/data:/var/lib/registry
```

> 这个没有图形界面的，通过访问```http://192.168.8.101:5000/v2/_catalog```查看已上传的docker镜像
>
> ```log
> {"repositories":[]}
> ```

docker客户端需要修改/etc/docker/daemon.json。

```json
{
  "registry-mirrors": ["https://XXXXXXXXX.mirror.aliyuncs.com"],
  "insecure-registries":["192.168.8.101:5000"]
}
```

然后重启客户端docker

```shell 
sudo systemctl daemon-reload && sudo systemctl restart docker
```

测试一下

```shell
#随便拉一个nginx的最新镜像
docker pull nginx
#标记本地镜像为远程镜像 「<ip>:<port>/<image name><:tag>」
docker tag nginx:lastest 192.168.8.101:5000/nginx
#推送
docker push 192.168.8.101:5000/nginx
```

 查看全部镜像

```shell
curl -XGET http://192.168.75.133:5000/v2/_catalog
```

查看指定镜像 

```shell
curl -XGET http://192.168.75.133:5000/v2/nginx/tags/list
```

```result
{"name":"nginx","tags":["latest"]}
```

## 192.168.8.102安装jenkins

```yml
version: '3'
services:
  jenkins:
    image: 'jenkins/jenkins:lts'
    container_name: jenkins
    privileged: true
    user: root
    restart: always
    ports:
      - '8080:8080'
      - '50000:50000'
    volumes:
      - '/home/vagrant/jenkins/jenkins_home:/var/jenkins_home'
```

> 启动访问192.168.8.102:8080
>
> ![image-20210802162045927](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210802162045927.jpeg)
>
> 这个密码就在控制台，可以使用```docker logs jenkins```查看。
>
> ![image-20210802164934983](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210802164934983.png)

> 也有其他版本，可以看[官网的安装教程](https://www.jenkins.io/zh/doc/book/installing/)
>
> 然后选择**建议安装**等待就可以了。

