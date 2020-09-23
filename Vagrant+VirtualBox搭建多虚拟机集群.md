# Vagrant+VirtualBox搭建多虚拟机集群

> 初衷是为了学习kubernetes和kubesphere。
>
> 前置条件：
>
> 1）下载安装完成VirtualBox和Vagrant；
>
> 2）vagrant init centos/7 已经拉取了centos7的官方镜像 【https://www.vagrantup.com/downloads】

## 修改Vagrantfile

```shell
Vagrant.configure("2") do |config|
        (1..3).each do |i|
                config.vm.define "kube-node0#{i}" do |node|
                        # 设置虚拟机的Box
                        node.vm.box = "centos/7"
                        # 设置虚拟机的主机名
                        node.vm.hostname="kube-node0#{i}"
                        # 设置虚拟机的IP
                        node.vm.network "private_network", ip: "192.168.59.#{99+i}", netmask: "255.255.255.0"
                        # 设置主机与虚拟机的共享目录
                        node.vm.synced_folder "~/k8s-cluster/data/#{i}", "/home/vagrant/share"
                        # VirtaulBox相关配置
                        node.vm.provider "virtualbox" do |v|
                                # 设置虚拟机的名称
                                v.name = "kube-node0#{i}"
                                # 设置虚拟机的内存大小
                                v.memory = 4096
                                # 设置虚拟机的CPU个数
                                v.cpus = 4
                        end
                end
        end
end
```

## 安装vagrant-vbguest插件

```sh
vagrant plugin install vagrant-vbguest
```

解决了unreliable DMA position问题。

也就是vboxsf报错。不然。会卡很久。

```verilog
Failed to mount folders in Linux guest. This is usually because
the "vboxsf" file system is not available. Please verify that
the guest additions are properly installed in the guest and
can work properly. The command attempted was:

mount -t vboxsf -o uid=`id -u vagrant`,gid=`getent group vagrant | cut -d: -f3` vagrant /vagrant
mount -t vboxsf -o uid=`id -u vagrant`,gid=`id -g vagrant` vagrant /vagrant
```

## 虚拟机初始化

```bash
vagrant up
```

等待创建完成即可。

## 常用命令

### vagrant默认root密码

```shell
vagrant
```

- **vagrant ssh:** SSH登陆虚拟机

- **vagrant halt:** 关闭虚拟机

- **vagrant destroy:** 删除虚拟机

- **vagrant ssh-config** 查看虚拟机SSH配置

  **启动单个虚拟机：**

  ```
  vagrant up node1
  ```

  **启动多个虚拟机：**

  ```
  vagrant up node1 node3
  ```

  **启动所有虚拟机：**

  ```
  vagrant up
  ```

## NAT网络和密码登录设置

### 密码登录

> 修改/etc/ssh/sshd_config

```shell
sudo vim /etc/ssh/sshd_config
```

找到PasswordAuthentication，修改成yes

```config
# To disable tunneled clear text passwords, change to no here!
#PasswordAuthentication yes
#PermitEmptyPasswords no
PasswordAuthentication yes
```

重启sshd

```shell
service sshd restart
```

就可以使用账号密码登录了。

### NAT网络

初始创建完成之后。eth0网卡会使用默认的NAT网络。且。配置完全相同，需要修改。

> 都是使用10.0.2.15，且mac地址相同。

```shell
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:4d:77:d3 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 85596sec preferred_lft 85596sec
    inet6 fe80::5054:ff:fe4d:77d3/64 scope link
       valid_lft forever preferred_lft forever
```

需要在VirtualBox全局的网络设置里，添加一个NAT网络。

手动设置到三个虚拟机的NAT网卡里，切记要点开高级，手动刷新mac地址。

## 扩容问题

> 最简单的方式是使用vagrant插件
>
> 官网地址
>
> https://github.com/sprotheroe/vagrant-disksize

```shell
# 安装vagrant插件
vagrant plugin install vagrant-disksize
```

```properties
Vagrant.configure('2') do |config|
  config.vm.box = '镜像'
  config.disksize.size = '100GB' #设置磁盘大小
end
```

```shell
# 关闭虚拟机
vagrant halt
# 重新启动 
vagrant up
```

