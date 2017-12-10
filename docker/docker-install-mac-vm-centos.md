

# Docker for Mac/Windows

​	开发环境中有时候想把 Docker 容器实例当做正常的虚拟机来用，换句话说就是本机和Docker容器实例处于同一个子网中，本机可以直接通过IP地址访问Docker容器实例，而不是通过中间端口映射的方式来访问！

​	Mac和Windows操作系统都无法直接支持Docker，都是通过安装虚拟机的方式来支持Docker功能， Docker Toolbox同时支持Mac和Windows操作系统，实际上Docker Toolbox在本机的作用是安装一套对应的Docker命令可以间接访问虚拟机中真正的Docker宿主机。

> 在Mac下使用 Docker 可以选择 Docker Toolbox 或 Docker for Mac，Docker for Mac 是 Docker Toolbox 的替代品，但是 Docker for Mac 对网络连通性的支持比较有限，因此目前还是用Docker Toolbox。



**最终效果**：

* MacBook、VirtualBox、Docker容器实例相互之间可以自由的用IP地址互相访问，就像正常的虚拟机一样。
* 可以给 docker 容器指定静态 IP 地址
* 容器实例使用统一的DNS服务器



## 前言	

缺省情况下安装完`Docker Toolbox`的网络配置如下：

| Host               | IP Address   |
| ------------------ | ------------ |
| MacBook( vboxnet1) | 192.168.33.1 |
| docker(VirtualBox) | 172.17.0.1   |
| c1                 | 172.17.0.2   |
| c2                 | 172.17.0.3   |

说明

* `Docker Toolbox` 会自动安装`VirtualBox`虚拟机

* `192.168.33.1` 是 `VirtualBox` 在 Mac 上绑定的虚拟网卡`vboxnet1`的地址

  ```bash
  $ ifconfig
  vboxnet1: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
  	ether 0a:00:27:00:00:01
  	inet 192.168.33.1 netmask 0xffffff00 broadcast 192.168.33.255
  ```

* `c1` 和 `c2` 是 Docker 容器实例，采用 `bridge` 模式后自动分配的IP地址是`172.17.0.2`和`172.17.0.3`

* 因为处于不同的网络，所以很显然`MacBook`无法直接访问容器实例`c1` 和 `c2` 



**解决方案**：

> Docker为了安全原因会让Docker容器实例处于不同网段，既然因为是处于不同网段造成的网络不通，那么修改`vboxnet1`的IP地址，让它和docker虚拟机处于同一个网段即可。



Docker VM：

> 如果直接在Linux系统下使用 Docker那么这个Linux就是`Docker VM`，如果是在Windows或Mac操作系统中，这个`Docker VM`就是VirtualBox里的虚拟机，Docker Toolbox缺省使用`boot2docker`这个Linux系统，当然也可以改用`ubuntu`或`CentOS`



## 目标

​	本文以Mac为例，Windows上的操作也类似，因为要安装DNS服务器等其他服务，`boot2docker`可能不太够用，另外服务器上最常用的操作系统是`Centos`，因此打算把`Docker VM` 也换成 `Centos7`，这样本机还可以同时有一个Centos环境。

**最终效果**：

- MacBook、VirtualBox、Docker容器实例相互之间可以自由的用IP地址互相访问，就像正常的虚拟机一样。
- Docker VM 换成 Centos7
- Docker VM 同时也是DNS服务器
- 可以给 docker 容器指定静态 IP 地址
- 容器实例使用统一的DNS服务器

最终的混杂网络配置如下：

| Host               | IP Address   |
| ------------------ | ------------ |
| MacBook            | 172.17.0.253 |
| docker(VirtualBox) | 172.17.0.1   |
| c1                 | 172.17.0.2   |
| c2                 | 172.17.0.3   |

说明

- 172.17.0.253 是 VirtualBox 在 Mac 上绑定的虚拟网卡`vboxnet1`的新地址

  为什么用`172.17.0.253`，因为采用网桥（Bridge）模式后，所有的容器实例没有指定IP地址的时候是从最小IP开始分配的，并且分配的时候并不知道某个IP地址已经被外面的`vboxnet1`用了，为了避免容器自动分配的IP地址和外面的`vboxnet1`的IP地址冲突，因此尽可能的给`vboxnet1`设置比较大的IP地址



## 预备

* Mac OS X 10.12.6:

- [VirtualBox - v5.2.2](https://www.virtualbox.org/wiki/Downloads)：虚拟机
- [Vagrant - v2.0.1](https://www.vagrantup.com/downloads.html)：通过配置文件来快速创建定制的虚拟机
- [Docker Toolbox - v1.10.3](https://docs.docker.com/toolbox/toolbox_install_mac/)：Docker 工具箱，包含了 VirtualBox

请首先下载并安装 Vagrant 和 Docker Toolbox，Docker Toolbox中包含了VirtualBox，VirtualBox可以也单独安装最新的版本。



## 更改`vboxnet1` IP address

```bash
$ VBoxManage hostonlyif ipconfig vboxnet1 --ip 172.17.0.253 --netmask 255.255.0.0
```

`````bash
$ ifconfig
vboxnet1: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	ether 0a:00:27:00:00:01
	inet 172.17.0.253 netmask 0xffff0000 broadcast 172.17.255.255
`````



## 创建虚拟机配置文件 `Vagrantfile`

注意：这里不能直接用`centos/7`，而是用`dolbager/centos-7-docker`

``````bash
$ vi Vagrantfile
``````

```bash
Vagrant.configure(2) do |config|
  config.vm.box = "dolbager/centos-7-docker"
  config.vm.hostname = "default"
  config.vm.network "private_network", ip: "172.17.0.1", netmask: "255.255.0.0"

  config.vm.provider "virtualbox" do |v|
    v.name = "default"
    v.memory = "2048"
    # Change the network adapter type and promiscuous mode
    v.customize ['modifyvm', :id, '--nictype1', 'Am79C973']
    v.customize ['modifyvm', :id, '--nicpromisc1', 'allow-all']
    v.customize ['modifyvm', :id, '--nictype2', 'Am79C973']
    v.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
  end

  # Install bridge-utils
  config.vm.provision "shell", inline: <<-SHELL
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
    yum clean all
    yum makecache
    yum update -y
    yum install bridge-utils net-tools -y
  SHELL
end
```

配置说明：

- hostname: 

  Docker VM 主机的名称=default 

- private_network: 

  Docker VM 主机的IP=172.17.0.1

- v.memory

  内存2048M

- bridge-utils

  创建网桥需要的辅助工具



## 创建Docker machine `default`

根据 `Vagrantfile` 运行`vagrant up`命令创建虚拟机

```bash
$ vagrant up
```

如果下载太慢（它是用 ruby 下载的，速度很慢），也可以用其他方式下载后让入指定目录

- Mac OS X and Linux: `~/.vagrant.d/boxes`


- Windows: `C:/Users/USERNAME/.vagrant.d/boxes`

默认的`root`密码是`vagrant`你可以修改为其他密码

``````bash
$ vagrant ssh
Last login: Fri Jan 13 15:50:24 2017 from 10.0.2.2
[vagrant@default ~]$ su root
[root@default vagrant]# passwd
[root@default vagrant]# exit
[vagrant@default ~]$ exit
``````

默认没有把ssh用的`private_key`文件复制到`.vagrant`目录下，我们可以手动复制一下：

``````bash
$ vagrant ssh-config
``````

```bash
$ scp ~/.vagrant.d/boxes/dolbager-VAGRANTSLASH-centos-7-docker/0.2/virtualbox/vagrant_private_key .vagrant/machines/default/virtualbox/private_key
```

如果上次创建失败了，则需要先删除上次安装的虚拟机 `default`

```bash
$ docker-machine rm default
```

然后就是让虚拟机整合 Docker 功能（也就是在虚拟机中安装Docker）了

```bash
# Setup the VM as your Docker machine
$ docker-machine create \
 --driver "generic" \
 --generic-ip-address 172.17.0.1 \
 --generic-ssh-user vagrant \
 --generic-ssh-key .vagrant/machines/default/virtualbox/private_key \
 --generic-ssh-port 22 \
 default
```

- --driver "generic"

  > generic表示通用的
  >
  > docker-machine create  --driver "generic" 意思是通过SSH登录到已经存在的主机后，给该主机整合Docker功能，然后据此创建一个docker machine。

- 会出现一个错误提示，不用管它

  > (default) Couldn't copy SSH public key : unable to copy ssh key: open .vagrant/machines/default/virtualbox/private_key.pub: no such file or directory

``````bash
ssh-port 22  default
Running pre-create checks...
Creating machine...
(default) Importing SSH key...
(default) Couldn't copy SSH public key : unable to copy ssh key: open .vagrant/machines/default/virtualbox/private_key.pub: no such file or directory
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with centos...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env default
``````



## 修改虚拟机网桥配置

通过` vagrant` 从虚拟机的 `eth0` 登录到虚拟机

```bash
# Log in to the VM via eth0
$ vagrant ssh
Last login: Fri Dec  8 20:04:34 2017 from 10.0.2.2
[vagrant@default ~]$
```

``````bash
[vagrant@default ~]$ ip -4 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 1000
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 85763sec preferred_lft 85763sec
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 1000
    inet 172.17.0.1/24 brd 172.17.0.255 scope global eth1
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
``````

我们可以看到

- Docker 在虚拟机内创建了网桥`docker0`
- 网桥`docker0`地址范围为`172.17.0.1/16`
- 网卡`eth1`的IP地址为`172.17.0.1`

我们接下来要做的就是将虚拟机的`eth1`绑定到网桥`docker0`上，并且让它变成虚拟机重启也有效。

创建网桥配置文件`docker0`

```bash
[vagrant@default ~]$ sudo vi /etc/sysconfig/network-scripts/ifcfg-docker0
```

```
DEVICE=docker0
TYPE=Bridge
BOOTPROTO=static
ONBOOT=yes
STP=on
IPADDR=
NETMASK=
GATEWAY=
DNS1=
```

>  vi 保存文件并退出：按ESC键 跳到命令模式，然后输入 “:wq” 回车

修改网卡配置 `eth1` :

```bash
[vagrant@default ~]$ sudo vi /etc/sysconfig/network-scripts/ifcfg-eth1
```

```
DEVICE=eth1
BOOTPROTO=static
HWADDR=
ONBOOT=yes
NETMASK=
GATEWAY=
BRIDGE=docker0
TYPE=Ethernet
```

重启虚拟机

``````bash
[vagrant@default ~]$ sudo reboot now
``````

通过` vagrant` 从虚拟机的 `eth0` 登录到虚拟机

```bash
# Log in to the VM via eth0
$ vagrant ssh
Last login: Fri Dec  8 20:04:34 2017 from 10.0.2.2
[vagrant@default ~]$
```

```bash
[vagrant@default ~]$ ip -4 addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN qlen 1000
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86374sec preferred_lft 86374sec
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN qlen 1000
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
```

我们可以看到已经起作用了。



## 运行Docker容器实例

退出虚拟机后我们就可以测试是否一切正常了。

不指定IP启动容器，IP 将从`172.17.0.2`开始逐个递增:

```bash
$ docker run -d --net=bridge --name=c1 nginx
$ docker run -d --net=bridge --name=c2 nginx
```

通过可以`docker network inspect`命令查看给容器分配的IP地址

`````bash
$ docker network inspect bridge
`````

``````
        "Containers": {
            "79e804aa864cde4c919d85ba7a9ce273055dfd827b7216787c849192f46b753d": {
                "Name": "c2",
                "EndpointID": "b7c1cf0c0169bac07f8dba37152b0ac3515117655e35a948638a59c9e8ddf841",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "c95ce3c20b8fd96f1fbef513c60f3b2c0d5547f9a325a741cdffda51bf24d048": {
                "Name": "c1",
                "EndpointID": "e6db861d48f1ec20945af9dcbac6438cbc2c4c3dcdf59cac6c6e74664b6456ce",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
``````

当然你也可以指定IP

``````bash
$ docker run -d --net=bridge --name=c1 --ip=172.17.0.2 nginx
$ docker run -d --net=bridge --name=c2 --ip=172.17.0.3 nginx
``````

然后你应该可以在Mac中直接通过IP访问这些容器实例了:

```bash
$ ping -c 3 172.17.0.2
$ ping -c 3 172.17.0.3
```

也可以通过浏览器来确认是否可以访问

``````
http://172.17.0.2
http://172.17.0.3
``````



## DNS

因为安全的原因，Docker容器是不允许映射 `/etc/hosts`的，所以只能临时给容器实例增加IP映射或让容器使用统一的 DNS 服务器

### 方法一：临时增加IP映射

```bash
$ docker run -it --net=br --add-host foo:10.0.0.3 --name c3 busybox cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
10.0.0.3	foo
192.168.33.3	f06e3c61830b
```

###方法二：指定DNS服务器

我采用方法二：给虚拟机安装DNS服务器`dnsmasq`，并且让所有容器实例缺省指向这个DNS服务器。

通过` vagrant` 从虚拟机的 `eth0` 登录到虚拟机

```bash
# Log in to the VM via eth0
$ vagrant ssh
Last login: Fri Dec  8 20:04:34 2017 from 10.0.2.2
[vagrant@default ~]$
```

安装 `dnsmasq`

``````bash
[vagrant@default ~]$ sudo yum -y install dnsmasq
[vagrant@default ~]$ sudo systemctl start dnsmasq 
[vagrant@default ~]$ sudo systemctl enable dnsmasq 
``````

增加一条记录

``````bash
[vagrant@default ~]$ sudo vi /etc/hosts
``````

``````
172.17.0.2    zk
``````

修改Docker的启动配置，指定DNS服务器，顺便加上`registry-mirrors`

```bash
[vagrant@default ~]$ sudo vi /etc/docker/daemon.json
```

```
{
  "dns" : ["172.17.0.1"],
  "registry-mirrors" : ["https://fnfrb3qa.mirror.aliyuncs.com"]
}
```

[dockerd document](https://docs.docker.com/engine/reference/commandline/dockerd/)

重启 Docker 虚拟机

```bash
[vagrant@default ~]$ sudo systemctl daemon-reload
[vagrant@default ~]$ sudo systemctl restart docker
[vagrant@default ~]$ exit
```

确认DNS是否工作正常

``````bash
$ docker run -d --net=bridge --name c4 busybox top
$ docker exec c4 ping -c 3 zk
PING zk (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.018 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.042 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.042 ms
``````

## Done



Vargrant前面下载了虚拟机镜像文件，如果将来不再需要了闲它占用空间，则可以清理一下

``````bash
$ vagrant box list
$ vagrant box remove dolbager/centos-7-docker
``````